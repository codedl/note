nginx跳转失败的配置:
```
proxy_next_upstream http_502 http_504 error invalid_header;
proxy_redirect off;
proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```