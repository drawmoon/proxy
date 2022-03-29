# HAProxy

- [部署](#部署)
  - [Windows](#windows)
- [配置模板](#配置模板)
- [访问控制](#访问控制)
  - [匹配 URL 路径前缀](#匹配-url-路径前缀)
  - [匹配正则表达式](#匹配正则表达式)
- [重写 HTTP 请求](#重写-http-请求)
  - [重写请求路径](#重写请求路径)
  - [重写请求方法](#重写请求方法)
  - [添加请求头](#添加请求头)
  - [重写请求头](#重写请求头)
  - [移除请求头](#移除请求头)
- [重写 HTTP 响应](#重写-http-响应)
  - [添加响应头](#添加响应头)
  - [重写响应头](#重写响应头)
  - [移除响应头](#移除响应头)
- [扩展](#扩展)
  - [启用 CORS 跨域](#启用-cors-跨域)
  - [处理 OPTIONS 方法的请求](#处理-options-方法的请求)
- [Docker](#docker)
  - [Dockerfile](#dockerfile)
- [Docker-Compose](#docker-compose)

## 部署

### Windows

安装 [Cygwin](https://cygwin.com/install.html)，在 Select Packages 界面中，展开 `devel` 选项卡，搜索并选择 `gcc`、`make` 包，下一步继续安装步骤。安装完成后，将 [haproxy-xxx.tar.gz](https://www.haproxy.org/download/) 拷贝到 Cygwin 的 `home` 目录下，接下来启动 Cygwin Terminal，执行以下命令进行编译：

```bash
# 解压 HAProxy 包，并进入到 HAProxy 目录下
tar -xzvf haproxy-xxx.tar.gz
cd haproxy-xxx

# 执行编译
make TARGET=cygwin
make install
```

编译完成后，将 `haproxy.exe` 可执行文件，和 Cygwin 根目录的 `bin` 文件夹下的 `cygwin1.dll` 、`cyggcc_s-1.dll` 类库文件一起拷贝到新建目录。在新建目录下执行以下命令即可启动 HAProxy：

```bash
haproxy.exe -f haproxy.cfg
```

## 配置模板

```conf
global
   strict-limits # refuse to start if insufficient FDs/memory

defaults
   mode http
   timeout client 60s
   timeout server 60s
   timeout connect 1s

frontend www
   bind :8000

   default_backend serve

backend serve
   server s1 localhost:5000
```

## 访问控制

### 匹配 URL 路径前缀

```conf
# 匹配 /report 前缀开始的 HTTP 请求发送到 report 后端
# 其他则发送到 main 后端

frontend wwww
   bind :8000

   acl report_url path_beg -i /report

   use_backend report if report_url
   default_backend admin

backend admin
   server s1 localhost:5000

backend report
   server s1 localhost:3000
```

### 匹配正则表达式

```conf
# 匹配 /report, /api/dir, /api/--/dir 等前缀开始的 HTTP 请求发送到 report 后端
# 其他则发送到 main 后端

frontend wwww
   bind :8000

   acl report_url path_reg -i \/(report\/)?api\/(--\/)?(dir|file|report|tmp|trash|browser|chart|sheet)

   use_backend report if report_url
   default_backend admin

backend admin
   server s1 localhost:5000

backend report
   server s1 localhost:3000
```

## 重写 HTTP 请求

使用 `http-request` 配置指令重写 HTTP 请求，可以写在 `listen`, `frontend` 或 `backend` 块中。

### 重写请求路径

使用 `set-path` 重写请求的 URL 路径。

```conf
# 将 /report-rest 更改为 /rest 示例
# GET /report-rest/{id} -> GET /rest/{i}

frontend wwww
   bind :8000

   http-request set-path %[path,regsub(^\/report-rest,/rest)]

   default_backend serve

backend serve
   server s1 localhost:5000
```

### 重写请求方法

使用 `set-method` 重写请求的方法。

```conf
# 将 PUT 方法的请求更改为 POST 方法

frontend wwww
   bind :8000

   http-request set-method POST if METH_PUT

   default_backend serve

backend serve
   server s1 localhost:5000
```

### 添加请求头

使用 `add-header` 添加请求头。

```conf
# 判断请求头中是否存在 X-Forwarded-For
# 如果不存在则添加包含客户端地址的 X-Forwarded-For 请求头

frontend wwww
   bind :8000

   acl has_x_forwarded_for req.hdr(X-Forwarded-For) -m found

   http-request add-header X-Forwarded-For %[src] unless has_x_forwarded_for

   default_backend serve

backend serve
   server s1 localhost:5000
```

### 重写请求头

使用 `set-header` 重写请求头。

```conf
frontend wwww
   bind :8000

   http-request set-header X-Forwarded-For %[src]

   default_backend serve

backend serve
   server s1 localhost:5000
```

### 移除请求头

使用 `del-header` 移除请求头。

```conf
frontend wwww
   bind :8000

   http-request del-header Cookie

   default_backend serve

backend serve
   server s1 localhost:5000
```

## 重写 HTTP 响应

### 添加响应头

```conf
frontend wwww
   bind :8000

   http-response add-header X-Powered-By "WebServer"

   default_backend serve

backend serve
   server s1 localhost:5000
```

### 重写响应头

```conf
frontend wwww
   bind :8000

   http-response set-header X-Powered-By "WebServer"

   default_backend serve

backend serve
   server s1 localhost:5000
```

### 移除响应头

```conf
frontend wwww
   bind :8000

   http-response del-header X-Powered-By

   default_backend serve

backend serve
   server s1 localhost:5000
```

## 扩展

### 启用 CORS 跨域

```conf
frontend www
   bind :8000

   capture request header Origin len -1

   http-response set-header Access-Control-Allow-Origin %[capture.req.hdr(0)]
   http-response set-header Access-Control-Allow-Methods "*"
   http-response set-header Access-Control-Allow-Headers "*"

   default_backend serve

backend serve
   server s1 localhost:5000
```

### 处理 OPTIONS 方法的请求

```conf
frontend www
   bind :8000

   capture request header Origin len -1

   http-response set-header Access-Control-Allow-Origin %[capture.req.hdr(0)] if METH_OPTIONS
   http-response set-header Access-Control-Allow-Methods "*" if METH_OPTIONS
   http-response set-header Access-Control-Allow-Headers "*" if METH_OPTIONS
   http-response set-header Access-Control-Max-Age "1728000" if METH_OPTIONS
   http-response set-status 204 if METH_OPTIONS

   default_backend serve

backend serve
   server s1 localhost:5000
```

## Docker

```bash
docker pull haproxy

docker run -v ./haproxy.cfg:/usr/local/etc/haproxy.cfg:ro -p 8000:8000
```

### Dockerfile

```docker
FROM haproxy
COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
```

## Docker-Compose

```yaml
version: "3"
services:

  httpproxy:
    image: haproxy:latest
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    ports:
      - "8000:8000"
    restart: always
```