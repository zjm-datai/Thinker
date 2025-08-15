

```
location /dify_api/ {
    rewrite ^/dify_api/(.*)$ /$1 break;
    proxy_pass http://211.90.240.240:30010/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

