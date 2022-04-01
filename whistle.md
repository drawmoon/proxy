# Whistle

- [访问控制](#访问控制)
  - [匹配 URL 路径前缀](#匹配-url-路径前缀)
  - [通配符匹配](#通配符匹配)
- [重写 HTTP 请求](#重写-http-请求)
  - [重写请求路径](#重写请求路径)
  - [添加请求头](#添加请求头)

## 访问控制

### 匹配 URL 路径前缀

```conf
# Rules

myweb.test localhost:5000
```

## 重写 HTTP 请求

### 重写请求路径

```conf
# Rules

myweb.test localhost:5000

# 将 api/file/{id}/picture 开始的 HTTP 请求转发到 api/file/{id}/chart
# http://myweb.test/api/file/718/picture/window/100?filename=expamle.png -> http://myweb.test/api/file/718/chart/window/100?filename=expamle.png
# 其中 ^ 表示以 ... 开头，$1 = 718，$2 = window/100?filename=expamle.png
^myweb.test/api/file/*/picture/*** localhost:5000/api/file/$1/chart/$2
```

### 添加请求头

```conf
# Values

# 创建名称为 websocket 的 Values
x-whistle-policy: tunnel

# Rules

myweb.test localhost:5000

myweb.test reqHeaders://{websocket}
```
