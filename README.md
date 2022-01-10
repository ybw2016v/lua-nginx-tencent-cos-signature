# lua-nginx-tencent-cos-signature

通过几项简单的修改，可以实现在无OpenResty的情况下，仅使用Ubuntu/Debian的nginx，在对nginx影响尽可能小的前提下，在包管理的框架下实现对生成腾讯云 COS 对象储存的[请求签名 (XML version)][docs]。

## 几项改动

1. 使用nginx自带的`hmac_sha1`计算函数替代OpenResty中的相关依赖库
2. 使用包管理中一键安装的`lua-nginx-string`替代OpenResty中的相关依赖库
3. 将`ACCESS_KEY_ID` 和 `SECRET_ACCESS_KEY`放置在nginx的配置而不是通过环境变量

## 使用方法

1. 安装必要的依赖

```bash
sudo apt install libnginx-mod-http-lua
sudo apt install lua-nginx-string
```

2. 将`cos.lua`放置在`/etc/nginx/lua/`目录下(或其他能正常使用的目录)

3. 写nginx配置文件，仅增加一个`server`即可

```nginx
server {
        listen 443 ssl http2;
        server_name a.neko.red;
        ssl_certificate     /root/key/dog.pem;
        ssl_certificate_key /root/key/dog.key;
        add_header 'Access-Control-Allow-Origin' '*'; #跨域用
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE'; #跨域用
        add_header 'Access-Control-Allow-Headers' 'Content-Type'; #跨域用
        proxy_intercept_errors on;
        location / {
            set $cos_access_key "";#access_key
            set $cos_secret_key "";#secret_key
            rewrite_by_lua_file "/etc/nginx/lua/cos.lua";#脚本放置位置
        }
        # internal redirect
        location @cos {
            proxy_pass https://<bucketname-appid>.cos.ap-beijing.myqcloud.com; #backet访问地址
            proxy_set_header   Host   <bucketname-appid>.cos.ap-beijing.myqcloud.com; #也可以换成自定义域名
        }
}
```

4. 检查配置文件并重启nginx

```bash
sudo nginx -t
sudo nginx -s reload
```

# lua-resty-tencent-cos-signature


## Overview

This library implements request signing using the [Tencent QCloud Signature XML
Version]([docs]) specification. We can use this signature to request objects from
COS an proxy them with Nginx (Openresty or [lua-nginx-module](https://github.com/openresty/lua-nginx-module)).

这个库用于生成腾讯云 COS 对象储存的[请求签名 (XML version)][docs]，故可用于配置 Nginx (需要安装
Openresty 或者编译 [lua-nginx-module](https://github.com/openresty/lua-nginx-module))
反代理私有仓库。

[docs]: https://cloud.tencent.com/document/product/436/7778#.E5.87.86.E5.A4.87.E5.B7.A5.E4.BD.9C

## Usage

This library uses environment variables as credentials.

使用以下环境变量定义 `ACCESS_KEY_ID` 和 `SECRET_ACCESS_KEY`。

```bash
export COS_ACCESS_KEY_ID=AKIDEXAMPLE
export COS_SECRET_ACCESS_KEY=AKIDEXAMPLE
```

To be accessible in your nginx configuration, these variables should be
declared in `nginx.conf` file.

之后还需要在 `nginx.conf` 中声明。

```nginx
#user  nobody;
worker_processes  1;

pid logs/nginx.pid;

env COS_ACCESS_KEY_ID;
env COS_SECRET_ACCESS_KEY;

# or specify them in Nginx config file only
#env COS_ACCESS_KEY_ID=AKIDEXAMPLE;
#env COS_SECRET_ACCESS_KEY=AKIDEXAMPLE;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /dev/stdout;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

You can then use the library to add COS Signature headers and `proxy_pass` to a
given COS bucket.

之后就可以在 server 块中添加配置了。

```nginx
server {
  listen 80 default_server;

  set $cos_bucket 'example-1200000000';
  set $cos_host $cos_bucket.cos.ap-hongkong.myqcloud.com;

  location / {
    resolver 127.0.0.53 valid=300s;
    resolver_timeout 10s;

    rewrite (.*) /$1 break;

    access_by_lua_block {
      require("resty.cos-signature").cos_set_headers()
    }

    proxy_set_header Host $cos_host;
    proxy_pass http://$cos_host;
  }
}
```

## Installing

It is recommend to install script with [OPM](https://opm.openresty.org/).

建议用 [OPM](https://opm.openresty.org/) 安装。

```bash
opm get mashirozx/lua-resty-tencent-cos-signature
```

## Contributing

Check [CONTRIBUTING.md](CONTRIBUTING.md) for more information.

## License

Copyright 2021 Mashiro

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

  [http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
