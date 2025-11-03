## 发布并运行你的 ASP.NET Core 项目


✅ 确认服务运行在 http://localhost:5000（或你配置的端口）


## 配置 Nginx 反向代理
C:\nginx-1.24.0\ 
├── conf\          ← 配置文件  
├── html\          ← 默认网页
|—— logs\          ← 日志    
├── nginx.exe      ← 主程序  
└── ...

## 配置 Nginx 反向代理
编辑配置文件：conf/nginx.conf
```
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    # ========== 反向代理配置开始 ==========
    server {
        listen       80;
        server_name  localhost;

        location / {
            proxy_pass http://localhost:5000;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        # 可选：静态文件缓存（如果有前端）
        # location /static/ {
        #    alias C:/path/to/static/files/;
        #    expires 30d;
        # }
    }
    # ========== 反向代理配置结束 ==========
}
```

- listen 80;	监听 80 端口（HTTP）
- server_name localhost;	域名（可改为 yourdomain.com）
- proxy_pass http://localhost:5000;	转发到你的 ASP.NET Core 服务
- proxy_set_header	传递原始请求信息，确保后端能正确获取 IP、协议等
⚠️ 注意：proxy_set_header Host $host; 很重要，否则后端生成的链接可能出错。

所有请求都经过 Nginx，你可以在此基础上添加：
- 负载均衡
- 缓存
- 静态文件服务
- HTTPS
- 访问日志 C:\nginx-1.24.0\logs\access.log
- 限流等
### 后续
- 生成 HTTPS 证书
- 写一个自动启动脚本
- 配置负载均衡
- 部署到 Linux 服务器

