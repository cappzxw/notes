## 机器翻译项目

项目底层基于tensorflow和tensor2tensor的翻译模型，将训练好的模型通过tensorflow server加载到内存中，构建flask后台程序提供 `/translate` api接口以供调用。

### 项目整体框架包括（从上到下）：

* [nginx 反向代理](#nginx_反向代理)
* [flask构建api接口](#flask构建api接口)
* [tensorflow server](#tensorflow_server)
* [tensor2tensor模型训练](#tensor2tensor模型训练)

### nginx 反向代理

翻译项目涉及到的语言有很多种，一般不一定部署在同一台机器上，所以需要反向代理将用户的请求代理到具体的机器上。项目中用OpenResty 实现nginx+lua的反向代理功能。

通过以下命令在centos上安装OpenResty

```shell
sudo yum install yum-utils
sudo yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo
sudo yum install openresty
```

将nginx添加到环境变量`export PATH=/usr/local/openresty/nginx/sbin:$PATH`

在目录`/usr/local/openresty/nginx/conf`中添加更改配置文件

```shell
conf
├── lua-script
│   └── main.lua
├── nginx.conf
└── upstream.conf
```

更改nginx.conf文件，引入lua模块获取post请求的内容

```nginx
lua_package_path '/usr/local/openresty/nginx/conf/lua-script/?.lua;;';
    include upstreams.conf;
    server {
        listen       80;
        server_name  localhost;

        location / {
            root   /path/to/web_server;
            index  index.html index.htm;
        }

        lua_need_request_body on;
        location /translate {
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            set $backend '';
            rewrite_by_lua_file /usr/local/openresty/nginx/conf/lua-script/main.lua;
            proxy_pass http://$backend/translate;
        }
    }
```



添加main.lua文件实现获取post内容来指定请求具体访问到哪个后端

```lua
function _send_error_response(err)
    ngx.say(err)
end


ngx.req.read_body()
local post = ngx.req.get_post_args()
local s = post.source
local t = post.target

if s == nil or s == '' then
    _send_error_response('missing source')
    return
end

if t == nil or t == '' then
    _send_error_response('missing target')
    return
end

_upstream = 'tool-proxy-'..tostring(s)..'-'..tostring(t)

if _upstream == nil or _upstream == '' then
    _send_error_response('Upstream is illegal')
    return
end
ngx.var.backend = _upstream
```



在upstream.conf中添加路由的具体地址

```
upstream tool-proxy-en-zh{     # source language - target language
    server 192.168.1.1:5000;   # ip:port
}
```

通过命令`ngnix -s reload` 重新加载服务

### flask构建api接口

参照 [flask_tensor_api](https://github.com/cappzxw/flask_tensor_api)

### tensorflow server

tensorflow server是tensorflow自带的服务，将训练好的模型添加到内存中提供服务。项目中应用的是tensorflow server的最基本的grpc机制，上面介绍的flask构建的restful API 去请求grpc，tensorflow server也可以直接提供restful API 功能（学习中...）。

tensorflow server在centos上安装比较复杂，源码编译需要很多额外工具，编译的过程也很漫长。项目采用docker中安装tensorflow server的镜像，再运行容器来提供服务。

在centos中采用以下命令安装并且启动docker ce

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum-config-manager --enable docker-ce-edge
yum install -y docker-ce
systemctl start docker
systemctl enable docker
```

在官网上下载tensorflow server的[镜像](https://www.tensorflow.org/serving/docker)，或者下载简单预先构建的镜像`docker pull registry.cn-hangzhou.aliyuncs.com/xwzhang/tfserver:1.0` 

下载好镜像后运行容器`docker run -d --name tfserver -p 8500:8000 -v /path/to/config:/config:ro  registry.cn-hangzhou.aliyuncs.com/xwzhang/tfserver:1.0`

`-v`后的参数是将本地的配置文件目录挂载到容器中，配置文件包括主配置文件和tensor2tensor训练好并且导出的模型。

```
trans
├── en2zh
│   ├── en2zh_data
│   ├── en2zh_script
│   └── export
└── models.json
```



`en2zh`是导出的模型目录，`models.json`为主要的配置文件

```
model_config_list: {
  config: {
    name: "en-zh",
    base_path: "path/to/trans/en2zh/export/Servo",
    model_platform: "tensorflow"
  }
}
```

如果是官网提供的镜像，需要在容器中运行启动服务的命令`tensorflow_model_server --port=9000 --model_config_file=/path/to/model.js`

tensorflow server 也提供gpu支持，源码编译时也提供很多种选择（学习中...）

### tensor2tensor 模型训练

主要参考tensor2tensor的github，[训练](https://github.com/tensorflow/tensor2tensor#translation)和[模型导出](https://github.com/tensorflow/tensor2tensor/blob/01af43d2b3e806035de461048a1d9fbe20f77bee/tensor2tensor/serving/README.md)介绍的都很详细。



### 参考资料

[openresty](https://openresty.org/cn/getting-started.html)

[反向代理部署](https://zhuanlan.zhihu.com/p/25202281)

[tensorflow server](https://www.tensorflow.org/serving/)

[tensor2tensor](https://github.com/tensorflow/tensor2tensor)

