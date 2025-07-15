```python
# controller.py
class AppImportApi(Resource):
    @setup_required
    @login_required
    @account_initialization_required
    @marshal_with(app_import_fields)
    def post(self):
        # Check user role first
        if not current_user.is_editor:
            raise Forbidden()

        parser = reqparse.RequestParser()
        parser.add_argument("mode", type=str, required=True, location="json")
        parser.add_argument("yaml_content", type=str, location="json")
        parser.add_argument("yaml_url", type=str, location="json")
        parser.add_argument("name", type=str, location="json")
        parser.add_argument("description", type=str, location="json")
        parser.add_argument("icon_type", type=str, location="json")
        parser.add_argument("icon", type=str, location="json")
        parser.add_argument("icon_background", type=str, location="json")
        parser.add_argument("app_id", type=str, location="json")
        args = parser.parse_args()

        # Create service with session
        with Session(db.engine) as session:
            import_service = AppDslService(session)
            # Import app
            account = cast(Account, current_user)
            result = import_service.import_app(
                account=account,
                import_mode=args["mode"],
                yaml_content=args.get("yaml_content"),
                yaml_url=args.get("yaml_url"),
                name=args.get("name"),
                description=args.get("description"),
                icon_type=args.get("icon_type"),
                icon=args.get("icon"),
                icon_background=args.get("icon_background"),
                app_id=args.get("app_id"),
            )
            session.commit()

        # Return appropriate status code based on result
        status = result.status
        if status == ImportStatus.FAILED.value:
            return result.model_dump(mode="json"), 400
        elif status == ImportStatus.PENDING.value:
            return result.model_dump(mode="json"), 202
        return result.model_dump(mode="json"), 200
```

```python
class AppDslService:
    def __init__(self, session: Session):
        self._session = session

    def import_app(
        self,
        *,
        account: Account,
        import_mode: str,
        yaml_content: Optional[str] = None,
        yaml_url: Optional[str] = None,
        name: Optional[str] = None,
        description: Optional[str] = None,
        icon_type: Optional[str] = None,
        icon: Optional[str] = None,
        icon_background: Optional[str] = None,
        app_id: Optional[str] = None,
    ) -> Import:
        """Import an app from YAML content or URL."""
        import_id = str(uuid.uuid4())

        # Validate import mode
        try:
            mode = ImportMode(import_mode)
        except ValueError:
            raise ValueError(f"Invalid import_mode: {import_mode}")

        # Get YAML content
        content: str = ""
        if mode == ImportMode.YAML_URL:
            if not yaml_url:
                return Import(
                    id=import_id,
                    status=ImportStatus.FAILED,
                    error="yaml_url is required when import_mode is yaml-url",
                )
            try:
                parsed_url = urlparse(yaml_url)
                if (
                    parsed_url.scheme == "https"
                    and parsed_url.netloc == "github.com"
                    and parsed_url.path.endswith((".yml", ".yaml"))
                ):
                    yaml_url = yaml_url.replace("https://github.com", "https://raw.githubusercontent.com")
                    yaml_url = yaml_url.replace("/blob/", "/")
                response = ssrf_proxy.get(yaml_url.strip(), follow_redirects=True, timeout=(10, 10))
                response.raise_for_status()
                content = response.content.decode()

                if len(content) > DSL_MAX_SIZE:
                    return Import(
                        id=import_id,
                        status=ImportStatus.FAILED,
                        error="File size exceeds the limit of 10MB",
                    )

                if not content:
                    return Import(
                        id=import_id,
                        status=ImportStatus.FAILED,
                        error="Empty content from url",
                    )
            except Exception as e:
                return Import(
                    id=import_id,
                    status=ImportStatus.FAILED,
                    error=f"Error fetching YAML from URL: {str(e)}",
                )
        elif mode == ImportMode.YAML_CONTENT:
            if not yaml_content:
                return Import(
                    id=import_id,
                    status=ImportStatus.FAILED,
                    error="yaml_content is required when import_mode is yaml-content",
                )
            content = yaml_content

        # Process YAML content
        try:
            # Parse YAML to validate format
            data = yaml.safe_load(content)
            if not isinstance(data, dict):
                return Import(
                    id=import_id,
                    status=ImportStatus.FAILED,
                    error="Invalid YAML format: content must be a mapping",
                )

            # Validate and fix DSL version
            if not data.get("version"):
                data["version"] = "0.1.0"
            if not data.get("kind") or data.get("kind") != "app":
                data["kind"] = "app"

            imported_version = data.get("version", "0.1.0")
            # check if imported_version is a float-like string
            if not isinstance(imported_version, str):
                raise ValueError(f"Invalid version type, expected str, got {type(imported_version)}")
            status = _check_version_compatibility(imported_version)

            # Extract app data
            app_data = data.get("app")
            if not app_data:
                return Import(
                    id=import_id,
                    status=ImportStatus.FAILED,
                    error="Missing app data in YAML content",
                )

            # If app_id is provided, check if it exists
            app = None
            if app_id:
                stmt = select(App).where(App.id == app_id, App.tenant_id == account.current_tenant_id)
                app = self._session.scalar(stmt)

                if not app:
                    return Import(
                        id=import_id,
                        status=ImportStatus.FAILED,
                        error="App not found",
                    )

                if app.mode not in [AppMode.WORKFLOW, AppMode.ADVANCED_CHAT]:
                    return Import(
                        id=import_id,
                        status=ImportStatus.FAILED,
                        error="Only workflow or advanced chat apps can be overwritten",
                    )

            # If major version mismatch, store import info in Redis
            if status == ImportStatus.PENDING:
                pending_data = PendingData(
                    import_mode=import_mode,
                    yaml_content=content,
                    name=name,
                    description=description,
                    icon_type=icon_type,
                    icon=icon,
                    icon_background=icon_background,
                    app_id=app_id,
                )
                redis_client.setex(
                    f"{IMPORT_INFO_REDIS_KEY_PREFIX}{import_id}",
                    IMPORT_INFO_REDIS_EXPIRY,
                    pending_data.model_dump_json(),
                )

                return Import(
                    id=import_id,
                    status=status,
                    app_id=app_id,
                    imported_dsl_version=imported_version,
                )

            # Extract dependencies
            dependencies = data.get("dependencies", [])
            check_dependencies_pending_data = None
            if dependencies:
                check_dependencies_pending_data = [PluginDependency.model_validate(d) for d in dependencies]
            elif imported_version <= "0.1.5":
                if "workflow" in data:
                    graph = data.get("workflow", {}).get("graph", {})
                    dependencies_list = self._extract_dependencies_from_workflow_graph(graph)
                else:
                    dependencies_list = self._extract_dependencies_from_model_config(data.get("model_config", {}))

                check_dependencies_pending_data = DependenciesAnalysisService.generate_latest_dependencies(
                    dependencies_list
                )

            # Create or update app
            app = self._create_or_update_app(
                app=app,
                data=data,
                account=account,
                name=name,
                description=description,
                icon_type=icon_type,
                icon=icon,
                icon_background=icon_background,
                dependencies=check_dependencies_pending_data,
            )

            return Import(
                id=import_id,
                status=status,
                app_id=app.id,
                app_mode=app.mode,
                imported_dsl_version=imported_version,
            )

        except yaml.YAMLError as e:
            return Import(
                id=import_id,
                status=ImportStatus.FAILED,
                error=f"Invalid YAML format: {str(e)}",
            )

        except Exception as e:
            logger.exception("Failed to import app")
            return Import(
                id=import_id,
                status=ImportStatus.FAILED,
                error=str(e),
            )

    def confirm_import(self, *, import_id: str, account: Account) -> Import:
        """
        Confirm an import that requires confirmation
        """
        redis_key = f"{IMPORT_INFO_REDIS_KEY_PREFIX}{import_id}"
        pending_data = redis_client.get(redis_key)

        if not pending_data:
            return Import(
                id=import_id,
                status=ImportStatus.FAILED,
                error="Import information expired or does not exist",
            )

        try:
            if not isinstance(pending_data, str | bytes):
                return Import(
                    id=import_id,
                    status=ImportStatus.FAILED,
                    error="Invalid import information",
                )
            pending_data = PendingData.model_validate_json(pending_data)
            data = yaml.safe_load(pending_data.yaml_content)

            app = None
            if pending_data.app_id:
                stmt = select(App).where(App.id == pending_data.app_id, App.tenant_id == account.current_tenant_id)
                app = self._session.scalar(stmt)

            # Create or update app
            app = self._create_or_update_app(
                app=app,
                data=data,
                account=account,
                name=pending_data.name,
                description=pending_data.description,
                icon_type=pending_data.icon_type,
                icon=pending_data.icon,
                icon_background=pending_data.icon_background,
            )

            # Delete import info from Redis
            redis_client.delete(redis_key)

            return Import(
                id=import_id,
                status=ImportStatus.COMPLETED,
                app_id=app.id,
                app_mode=app.mode,
                current_dsl_version=CURRENT_DSL_VERSION,
                imported_dsl_version=data.get("version", "0.1.0"),
            )

        except Exception as e:
            logger.exception("Error confirming import")
            return Import(
                id=import_id,
                status=ImportStatus.FAILED,
                error=str(e),
            )

    def check_dependencies(
        self,
        *,
        app_model: App,
    ) -> CheckDependenciesResult:
        """Check dependencies"""
        # Get dependencies from Redis
        redis_key = f"{CHECK_DEPENDENCIES_REDIS_KEY_PREFIX}{app_model.id}"
        dependencies = redis_client.get(redis_key)
        if not dependencies:
            return CheckDependenciesResult()

        # Extract dependencies
        dependencies = CheckDependenciesPendingData.model_validate_json(dependencies)

        # Get leaked dependencies
        leaked_dependencies = DependenciesAnalysisService.get_leaked_dependencies(
            tenant_id=app_model.tenant_id, dependencies=dependencies.dependencies
        )
        return CheckDependenciesResult(
            leaked_dependencies=leaked_dependencies,
        )

    def _create_or_update_app(
        self,
        *,
        app: Optional[App],
        data: dict,
        account: Account,
        name: Optional[str] = None,
        description: Optional[str] = None,
        icon_type: Optional[str] = None,
        icon: Optional[str] = None,
        icon_background: Optional[str] = None,
        dependencies: Optional[list[PluginDependency]] = None,
    ) -> App:
        """Create a new app or update an existing one."""
        app_data = data.get("app", {})
        app_mode = app_data.get("mode")
        if not app_mode:
            raise ValueError("loss app mode")
        app_mode = AppMode(app_mode)

        # Set icon type
        icon_type_value = icon_type or app_data.get("icon_type")
        if icon_type_value in ["emoji", "link"]:
            icon_type = icon_type_value
        else:
            icon_type = "emoji"
        icon = icon or str(app_data.get("icon", ""))

        if app:
            # Update existing app
            app.name = name or app_data.get("name", app.name)
            app.description = description or app_data.get("description", app.description)
            app.icon_type = icon_type
            app.icon = icon
            app.icon_background = icon_background or app_data.get("icon_background", app.icon_background)
            app.updated_by = account.id
        else:
            if account.current_tenant_id is None:
                raise ValueError("Current tenant is not set")

            # Create new app
            app = App()
            app.id = str(uuid4())
            app.tenant_id = account.current_tenant_id
            app.mode = app_mode.value
            app.name = name or app_data.get("name", "")
            app.description = description or app_data.get("description", "")
            app.icon_type = icon_type
            app.icon = icon
            app.icon_background = icon_background or app_data.get("icon_background", "#FFFFFF")
            app.enable_site = True
            app.enable_api = True
            app.use_icon_as_answer_icon = app_data.get("use_icon_as_answer_icon", False)
            app.created_by = account.id
            app.updated_by = account.id

            self._session.add(app)
            self._session.commit()
            app_was_created.send(app, account=account)

        # save dependencies
        if dependencies:
            redis_client.setex(
                f"{CHECK_DEPENDENCIES_REDIS_KEY_PREFIX}{app.id}",
                IMPORT_INFO_REDIS_EXPIRY,
                CheckDependenciesPendingData(app_id=app.id, dependencies=dependencies).model_dump_json(),
            )

        # Initialize app based on mode
        if app_mode in {AppMode.ADVANCED_CHAT, AppMode.WORKFLOW}:
            workflow_data = data.get("workflow")
            if not workflow_data or not isinstance(workflow_data, dict):
                raise ValueError("Missing workflow data for workflow/advanced chat app")

            environment_variables_list = workflow_data.get("environment_variables", [])
            environment_variables = [
                variable_factory.build_environment_variable_from_mapping(obj) for obj in environment_variables_list
            ]
            conversation_variables_list = workflow_data.get("conversation_variables", [])
            conversation_variables = [
                variable_factory.build_conversation_variable_from_mapping(obj) for obj in conversation_variables_list
            ]

            workflow_service = WorkflowService()
            current_draft_workflow = workflow_service.get_draft_workflow(app_model=app)
            if current_draft_workflow:
                unique_hash = current_draft_workflow.unique_hash
            else:
                unique_hash = None
            workflow_service.sync_draft_workflow(
                app_model=app,
                graph=workflow_data.get("graph", {}),
                features=workflow_data.get("features", {}),
                unique_hash=unique_hash,
                account=account,
                environment_variables=environment_variables,
                conversation_variables=conversation_variables,
            )
        elif app_mode in {AppMode.CHAT, AppMode.AGENT_CHAT, AppMode.COMPLETION}:
            # Initialize model config
            model_config = data.get("model_config")
            if not model_config or not isinstance(model_config, dict):
                raise ValueError("Missing model_config for chat/agent-chat/completion app")
            # Initialize or update model config
            if not app.app_model_config:
                app_model_config = AppModelConfig().from_model_config_dict(model_config)
                app_model_config.id = str(uuid4())
                app_model_config.app_id = app.id
                app_model_config.created_by = account.id
                app_model_config.updated_by = account.id

                app.app_model_config_id = app_model_config.id

                self._session.add(app_model_config)
                app_model_config_was_updated.send(app, app_model_config=app_model_config)
        else:
            raise ValueError("Invalid app mode")
        return app

    @classmethod
    def export_dsl(cls, app_model: App, include_secret: bool = False) -> str:
        """
        Export app
        :param app_model: App instance
        :return:
        """
        app_mode = AppMode.value_of(app_model.mode)

        export_data = {
            "version": CURRENT_DSL_VERSION,
            "kind": "app",
            "app": {
                "name": app_model.name,
                "mode": app_model.mode,
                "icon": "🤖" if app_model.icon_type == "image" else app_model.icon,
                "icon_background": "#FFEAD5" if app_model.icon_type == "image" else app_model.icon_background,
                "description": app_model.description,
                "use_icon_as_answer_icon": app_model.use_icon_as_answer_icon,
            },
        }

        if app_mode in {AppMode.ADVANCED_CHAT, AppMode.WORKFLOW}:
            cls._append_workflow_export_data(
                export_data=export_data, app_model=app_model, include_secret=include_secret
            )
        else:
            cls._append_model_config_export_data(export_data, app_model)

        return yaml.dump(export_data, allow_unicode=True)  # type: ignore

    @classmethod
    def _append_workflow_export_data(cls, *, export_data: dict, app_model: App, include_secret: bool) -> None:
        """
        Append workflow export data
        :param export_data: export data
        :param app_model: App instance
        """
        workflow_service = WorkflowService()
        workflow = workflow_service.get_draft_workflow(app_model)
        if not workflow:
            raise ValueError("Missing draft workflow configuration, please check.")

        export_data["workflow"] = workflow.to_dict(include_secret=include_secret)
        dependencies = cls._extract_dependencies_from_workflow(workflow)
        export_data["dependencies"] = [
            jsonable_encoder(d.model_dump())
            for d in DependenciesAnalysisService.generate_dependencies(
                tenant_id=app_model.tenant_id, dependencies=dependencies
            )
        ]

    @classmethod
    def _append_model_config_export_data(cls, export_data: dict, app_model: App) -> None:
        """
        Append model config export data
        :param export_data: export data
        :param app_model: App instance
        """
        app_model_config = app_model.app_model_config
        if not app_model_config:
            raise ValueError("Missing app configuration, please check.")

        export_data["model_config"] = app_model_config.to_dict()
        dependencies = cls._extract_dependencies_from_model_config(app_model_config.to_dict())
        export_data["dependencies"] = [
            jsonable_encoder(d.model_dump())
            for d in DependenciesAnalysisService.generate_dependencies(
                tenant_id=app_model.tenant_id, dependencies=dependencies
            )
        ]

    @classmethod
    def _extract_dependencies_from_workflow(cls, workflow: Workflow) -> list[str]:
        """
        Extract dependencies from workflow
        :param workflow: Workflow instance
        :return: dependencies list format like ["langgenius/google"]
        """
        graph = workflow.graph_dict
        dependencies = cls._extract_dependencies_from_workflow_graph(graph)
        return dependencies

    @classmethod
    def _extract_dependencies_from_workflow_graph(cls, graph: Mapping) -> list[str]:
        """
        Extract dependencies from workflow graph
        :param graph: Workflow graph
        :return: dependencies list format like ["langgenius/google"]
        """
        dependencies = []
        for node in graph.get("nodes", []):
            try:
                typ = node.get("data", {}).get("type")
                match typ:
                    case NodeType.TOOL.value:
                        tool_entity = ToolNodeData(**node["data"])
                        dependencies.append(
                            DependenciesAnalysisService.analyze_tool_dependency(tool_entity.provider_id),
                        )
                    case NodeType.LLM.value:
                        llm_entity = LLMNodeData(**node["data"])
                        dependencies.append(
                            DependenciesAnalysisService.analyze_model_provider_dependency(llm_entity.model.provider),
                        )
                    case NodeType.QUESTION_CLASSIFIER.value:
                        question_classifier_entity = QuestionClassifierNodeData(**node["data"])
                        dependencies.append(
                            DependenciesAnalysisService.analyze_model_provider_dependency(
                                question_classifier_entity.model.provider
                            ),
                        )
                    case NodeType.PARAMETER_EXTRACTOR.value:
                        parameter_extractor_entity = ParameterExtractorNodeData(**node["data"])
                        dependencies.append(
                            DependenciesAnalysisService.analyze_model_provider_dependency(
                                parameter_extractor_entity.model.provider
                            ),
                        )
                    case NodeType.KNOWLEDGE_RETRIEVAL.value:
                        knowledge_retrieval_entity = KnowledgeRetrievalNodeData(**node["data"])
                        if knowledge_retrieval_entity.retrieval_mode == "multiple":
                            if knowledge_retrieval_entity.multiple_retrieval_config:
                                if (
                                    knowledge_retrieval_entity.multiple_retrieval_config.reranking_mode
                                    == "reranking_model"
                                ):
                                    if knowledge_retrieval_entity.multiple_retrieval_config.reranking_model:
                                        dependencies.append(
                                            DependenciesAnalysisService.analyze_model_provider_dependency(
                                                knowledge_retrieval_entity.multiple_retrieval_config.reranking_model.provider
                                            ),
                                        )
                                elif (
                                    knowledge_retrieval_entity.multiple_retrieval_config.reranking_mode
                                    == "weighted_score"
                                ):
                                    if knowledge_retrieval_entity.multiple_retrieval_config.weights:
                                        vector_setting = (
                                            knowledge_retrieval_entity.multiple_retrieval_config.weights.vector_setting
                                        )
                                        dependencies.append(
                                            DependenciesAnalysisService.analyze_model_provider_dependency(
                                                vector_setting.embedding_provider_name
                                            ),
                                        )
                        elif knowledge_retrieval_entity.retrieval_mode == "single":
                            model_config = knowledge_retrieval_entity.single_retrieval_config
                            if model_config:
                                dependencies.append(
                                    DependenciesAnalysisService.analyze_model_provider_dependency(
                                        model_config.model.provider
                                    ),
                                )
                    case _:
                        # TODO: Handle default case or unknown node types
                        pass
            except Exception as e:
                logger.exception("Error extracting node dependency", exc_info=e)

        return dependencies

    @classmethod
    def _extract_dependencies_from_model_config(cls, model_config: Mapping) -> list[str]:
        """
        Extract dependencies from model config
        :param model_config: model config dict
        :return: dependencies list format like ["langgenius/google"]
        """
        dependencies = []

        try:
            # completion model
            model_dict = model_config.get("model", {})
            if model_dict:
                dependencies.append(
                    DependenciesAnalysisService.analyze_model_provider_dependency(model_dict.get("provider", ""))
                )

            # reranking model
            dataset_configs = model_config.get("dataset_configs", {})
            if dataset_configs:
                for dataset_config in dataset_configs.get("datasets", {}).get("datasets", []):
                    if dataset_config.get("reranking_model"):
                        dependencies.append(
                            DependenciesAnalysisService.analyze_model_provider_dependency(
                                dataset_config.get("reranking_model", {})
                                .get("reranking_provider_name", {})
                                .get("provider")
                            )
                        )

            # tools
            agent_configs = model_config.get("agent_mode", {})
            if agent_configs:
                for agent_config in agent_configs.get("tools", []):
                    dependencies.append(
                        DependenciesAnalysisService.analyze_tool_dependency(agent_config.get("provider_id"))
                    )

        except Exception as e:
            logger.exception("Error extracting model config dependency", exc_info=e)

        return dependencies

    @classmethod
    def get_leaked_dependencies(cls, tenant_id: str, dsl_dependencies: list[dict]) -> list[PluginDependency]:
        """
        Returns the leaked dependencies in current workspace
        """
        dependencies = [PluginDependency(**dep) for dep in dsl_dependencies]
        if not dependencies:
            return []

        return DependenciesAnalysisService.get_leaked_dependencies(tenant_id=tenant_id, dependencies=dependencies)
```

```python
class WorkflowService:
    """
    Workflow Service
    """

    def get_draft_workflow(self, app_model: App) -> Optional[Workflow]:
        """
        Get draft workflow
        """
        # fetch draft workflow by app_model
        workflow = (
            db.session.query(Workflow)
            .filter(
                Workflow.tenant_id == app_model.tenant_id, Workflow.app_id == app_model.id, Workflow.version == "draft"
            )
            .first()
        )

        # return draft workflow
        return workflow
```

```python
class Workflow(Base):
    """
    Workflow, for `Workflow App` and `Chat App workflow mode`.

    Attributes:

    - id (uuid) Workflow ID, pk
    - tenant_id (uuid) Workspace ID
    - app_id (uuid) App ID
    - type (string) Workflow type

        `workflow` for `Workflow App`

        `chat` for `Chat App workflow mode`

    - version (string) Version

        `draft` for draft version (only one for each app), other for version number (redundant)

    - graph (text) Workflow canvas configuration (JSON)

        The entire canvas configuration JSON, including Node, Edge, and other configurations

        - nodes (array[object]) Node list, see Node Schema

        - edges (array[object]) Edge list, see Edge Schema

    - created_by (uuid) Creator ID
    - created_at (timestamp) Creation time
    - updated_by (uuid) `optional` Last updater ID
    - updated_at (timestamp) `optional` Last update time
    """

    __tablename__ = "workflows"
    __table_args__ = (
        db.PrimaryKeyConstraint("id", name="workflow_pkey"),
        db.Index("workflow_version_idx", "tenant_id", "app_id", "version"),
    )

    id: Mapped[str] = mapped_column(StringUUID, server_default=db.text("uuid_generate_v4()"))
    tenant_id: Mapped[str] = mapped_column(StringUUID, nullable=False)
    app_id: Mapped[str] = mapped_column(StringUUID, nullable=False)
    type: Mapped[str] = mapped_column(db.String(255), nullable=False)
    version: Mapped[str]
    marked_name: Mapped[str] = mapped_column(default="", server_default="")
    marked_comment: Mapped[str] = mapped_column(default="", server_default="")
    graph: Mapped[str] = mapped_column(sa.Text)
    _features: Mapped[str] = mapped_column("features", sa.TEXT)
    created_by: Mapped[str] = mapped_column(StringUUID, nullable=False)
    created_at: Mapped[datetime] = mapped_column(db.DateTime, nullable=False, server_default=func.current_timestamp())
    updated_by: Mapped[Optional[str]] = mapped_column(StringUUID)
    updated_at: Mapped[datetime] = mapped_column(
        db.DateTime,
        nullable=False,
        default=datetime.now(UTC).replace(tzinfo=None),
        server_onupdate=func.current_timestamp(),
    )
    _environment_variables: Mapped[str] = mapped_column(
        "environment_variables", db.Text, nullable=False, server_default="{}"
    )
    _conversation_variables: Mapped[str] = mapped_column(
        "conversation_variables", db.Text, nullable=False, server_default="{}"
    )
```

```python
class App(Base):
    __tablename__ = "apps"
    __table_args__ = (db.PrimaryKeyConstraint("id", name="app_pkey"), db.Index("app_tenant_id_idx", "tenant_id"))

    id = db.Column(StringUUID, server_default=db.text("uuid_generate_v4()"))
    tenant_id: Mapped[str] = db.Column(StringUUID, nullable=False)
    name = db.Column(db.String(255), nullable=False)
    description = db.Column(db.Text, nullable=False, server_default=db.text("''::character varying"))
    mode: Mapped[str] = mapped_column(db.String(255), nullable=False)
    icon_type = db.Column(db.String(255), nullable=True)  # image, emoji
    icon = db.Column(db.String(255))
    icon_background = db.Column(db.String(255))
    app_model_config_id = db.Column(StringUUID, nullable=True)
    workflow_id = db.Column(StringUUID, nullable=True)
    status = db.Column(db.String(255), nullable=False, server_default=db.text("'normal'::character varying"))
    enable_site = db.Column(db.Boolean, nullable=False)
    enable_api = db.Column(db.Boolean, nullable=False)
    api_rpm = db.Column(db.Integer, nullable=False, server_default=db.text("0"))
    api_rph = db.Column(db.Integer, nullable=False, server_default=db.text("0"))
    is_demo = db.Column(db.Boolean, nullable=False, server_default=db.text("false"))
    is_public = db.Column(db.Boolean, nullable=False, server_default=db.text("false"))
    is_universal = db.Column(db.Boolean, nullable=False, server_default=db.text("false"))
    tracing = db.Column(db.Text, nullable=True)
    max_active_requests: Mapped[Optional[int]] = mapped_column(nullable=True)
    created_by = db.Column(StringUUID, nullable=True)
    created_at = db.Column(db.DateTime, nullable=False, server_default=func.current_timestamp())
    updated_by = db.Column(StringUUID, nullable=True)
    updated_at = db.Column(db.DateTime, nullable=False, server_default=func.current_timestamp())
    use_icon_as_answer_icon = db.Column(db.Boolean, nullable=False, server_default=db.text("false"))
```

我们要构建一个导入 dify DSL，直接生成接口的项目，由于这个接口会由 go 来做发布，所以会很酷。

在这里我们需要完成项目的第一个接口，将 dsl 中的信息切换成 app 和 workflow 实例存入数据库中。

上述信息是 dify 中实现这个接口的示例。

请你根据上面的信息，将我上述逻辑改造为一个 Gin 项目，要求首先给出你的思考，和步骤规划，然后一步一步的给出操作说明。