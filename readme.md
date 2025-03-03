# CloudWeGo

### 安装cwgo脚手架

`GOPROXY=https://goproxy.cn/,direct go install github.com/cloudwego/cwgo@latest` 

### 安装thrift脚手架

`go install github.com/cloudwego/thriftgo@latest`

### 添加auto-completion功能
```sh
mkdir autocomplete # You can choose any location you like
cwgo completion zsh > ./autocomplete/zsh_autocomplete
source ./autocomplete/zsh_autocomplete
```

### cwgo生成脚手架代码

```sh
cwgo server --type RPC --module github.com/cloudwego/biz-demo/gomall/demo/demo_thrift --service demo_thrift --idl ../../idl/echo.thrift
```


### 配置protoc

我们依赖protoc来编译protobuf文件,在github下载对应Release版本到ubuntu主机

将protoc拷贝到/usr/local/bin目录下
将include/google拷贝到 /usr/local/include目录下
这样即可使用cwgo生成protobuf对应脚手架
对于protobuf需要指定一个search_path

`cwgo server -I ../../idl --type RPC --module github.com/cloudwego/biz-demo/gomall/demo/demo_proto --service demo_proto --idl ../../idl/echo.proto`

这里需要注意下先使用go work init 再使用go work use .