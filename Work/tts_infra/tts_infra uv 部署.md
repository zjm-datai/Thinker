
目前项目采用很原始的 requirements.txt 加上 conda_unpack 等方式进行镜像打包，感觉有点疑似太原始了。

需要切换成 uv 部署，参考 openavatar 项目，这个项目是所有要镜像中安装 cuda 和 torch 等东西的一个部署模范我感觉。