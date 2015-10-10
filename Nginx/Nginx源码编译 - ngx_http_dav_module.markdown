# Nginx源码编译 - ngx_http_dav_module

标签（空格分隔）： Nginx

---
[TOC]

Nginx支持WebDAV的部分方法，但不包括PROPFIND & OPTIONS方法，若需要支持，就需要添加ngx_http_dav_module模块，该模块不包含在nginx的源码中，需要单独下载，下载地址：[ngx_http_dav_module](https://github.com/arut/nginx-dav-ext-module)。

### 编译配置
```
./configure --prefix=/usr/local/nginx --sbin-path=/usr/local/nginx/nginx --conf-path=/usr/local/nginx/nginx.conf --pid-path=/usr/local/nginx/nginx.pid --with-http_dav_module --add-module=../nginx-dav-ext-module-master --add-module=../nginx_upload_module-2.2.0 --add-module=../nginx-upload-progress-module-master
```

### 配置样例
```
location / {
    dav_methods PUT DELETE MKCOL COPY MOVE;
    dav_ext_methods PROPFIND OPTIONS;
    root /var/root/;
}
```
