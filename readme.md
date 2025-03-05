# GoMall

## 功能需求

#### （一）注册中心集成

1. 服务注册与发现
   1. 该服务能够与注册中心（如 Consul、Nacos 、etcd 等）进行集成，自动注册服务数据。

#### （二）身份认证

1. 登录认证
   1. 可以使用第三方现成的登录验证框架（CasBin、Satoken等），对请求进行身份验证
   2. 可配置的认证白名单，对于某些不需要认证的接口或路径，允许直接访问
   3. 可配置的黑名单，对于某些异常的用户，直接进行封禁处理（可选）
2. 权限认证（高级）
   1. 根据用户的角色和权限，对请求进行授权检查，确保只有具有相应权限的用户能够访问特定的服务或接口。
   2. 支持正则表达模式的权限匹配（加分项）
   3. 支持动态更新用户权限信息，当用户权限发生变化时，权限校验能够实时生效。

#### （三）可观测要求

1. 日志记录与监控
   1. 对服务的运行状态和请求处理过程进行详细的日志记录，方便故障排查和性能分析。
   2. 提供实时监控功能，能够及时发现和解决系统中的问题。

#### （四）可靠性要求（高级）

1. 容错机制
   1. 该服务应具备一定的容错能力，当出现部分下游服务不可用或网络故障时，能够自动切换到备用服务或进行降级处理。
   2. 保证下游在异常情况下，系统的整体可用性不会受太大影响，且核心服务可用。
   3. 服务应该具有一定的流量兜底措施，在服务流量激增时，应该给予一定的限流措施。

## 前端界面

前端界面有CloudWeGo提供，没有前端美化需求无需修改。

```sh
go mod tidy #先安装对应依赖
```

## 通知服务

异步处理消息/事件，对系统有一个很好的解耦，保证响应速度；使用中间件构建通知服务

消息中间件：Message queue；Message scheme；Notification

![image-20250304154028648](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250304154028648.png)

```sh
go get github.com/nats-io/nats.go/ #go client安装nats golang sdk
```

开发流程类似，都是先定义好IDL文件，利用cwgo生成相对应的服务，然后利用生成好的框架编写对应的业务逻辑，编写好对应Init函数，在main.go中调用即可

## 可观测要求

![image-20250304153808022](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250304153808022.png)

Prometheus+Grafana实现指标观测

使用 OpenTelemetry 进行 链路追踪（trace） 和日志（Log）打点和输出

## 云服务器部署

本次部署在阿里云服务器进行部署

### Go编译运行环境配置

```shell
Golang的官网下载地址：<https://golang.org/dl/>
下载golang代码解压放至 /usr/local/go
然后按照图示在$USER目录下 .bashrc中添加以下内容即可
```

![](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250303170124617.png)

```go
// 在主文件中必须引入main的包
package main
 
import "fmt"
 
//通过找到该main()方法进行执行程序
func main() {
        fmt.Printf("Golang env Test-2025-01-09!\\n")
}          
```

### 项目启动

```sh
make init #拷贝运行环境文件
make tidy #拉取go相关依赖
sudo apt install docker-compose
```

- 本次项目为Herts框架，可以新建一个工程感受一下，类似Django启动
- linux下配置好Golang环境，本次项目建议go1.22
- 还需要安装air；mysql；docker-compose
- open是Macos系统调用可以export GOPROXY=https://goproxy.cn,direct打开网页，linux换成xdg-open即可
- 重点 `make run` 的时候我们要先启动前端页面，再启动后端数据库相关服务，比如order/product等等
- 关于make tidy时候报错：一方面是网络代理问题，使用下面这句话解决即可，另一方面可能是之前缓存没清理干净，及时清理缓存再链接下载相关包

```bash
export GOPROXY=https://goproxy.cn,direct
go clean -modcache
```

![image-20250303171902274](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250303171902274.png)

云服务器无法启动docker守护进程

```sh
sudo chmod 666 /var/run/docker.sock # /var/run/docker.sock 权限不对，会导致非 root 用户无法访问
export DOCKER_HOST=unix:///var/run/docker.sock #确保 Docker Daemon 在监听正确的 Socket

#当前用户组加入docker
sudo usermod -aG docker $USER
newgrp docker  # 让用户组变更立即生效
```

```sh
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json

{
  "registry-mirrors": ["https://mirror.ccs.tencentyun.com"]
}

sudo systemctl daemon-reload
sudo systemctl restart docker
```



安装air

```sh
go install github.com/cosmtrek/air@v1.50
然后把 $GOPATH/bin 加入环境变量,这里需要注意我们需要提前在bashrc中声明GOPATH：
export PATH=$PATH:$(go env GOPATH)/bin
永久生效（添加到 ~/.bashrc 或 ~/.profile）：

source ~/.bashrc
air -v
```



## 问题汇总

### docker容器占用相关操作

### docker启动异常

直接启动`make env-start`如果本地安装mysql或者redis会出现报错，启动dockedr容器时候，会显示端口号被占用。解决办法可以是修改源码中的对应端口号或者是直接杀死本地的mysql和redis。

```sh
# 停止 MySQL 服务（systemd 会管理进程生命周期）
sudo systemctl stop mysql

# 确认状态（应显示 "inactive (dead)"）
sudo systemctl status mysql

# 禁用 MySQL 开机自启动
sudo systemctl disable mysql

# 通过 systemctl 停止 Redis 服务（如果通过包管理安装）
sudo systemctl stop redis-server

# 禁用 Redis 开机自启
sudo systemctl disable redis-server

# 查找所有 MySQL 相关进程
sudo pgrep -f mysql

# 强制杀死所有 MySQL 进程
sudo pkill -9 -f mysql

```

**修改 Docker Compose 端口映射（可选）**

如果仍需保留本地 Redis，可以修改 `docker-compose.yaml`，将 Redis 的宿主机端口改为其他值（例如 `6380:6379`）.为了从宿主机或外部访问容器内的服务，必须将容器端口映射到宿主机的端口。端口映射的格式通常是`宿主端口:容器端口`，也就是左边的端口是宿主机的，右边是容器内部的。在修改docker-compose.yml后，需要重新启动容器才能使新的端口映射生效。

```yaml
services:
  redis:
    image: redis:alpine
    ports:
      - "6380:6379"  # 修改左侧的宿主机端口
```

**清理 Docker 残留**：如果多次启动失败，先清理旧容器：

```bash
docker-compose down
```

### conventional commit

安装cz工具

```sh
npm install -g commitizen cz-conventional-changelog
npm ls -g -depth=0
echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc
npm install -g @commitlint/cli @commitlint/config-conventional # 校验规范的l
```

这里我们需要先安装nvm npm以及升级nodejs

```sh
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
nvm install 16   # 安装 Node.js 16.x（或更高版本）
nvm use 16       # 切换到 Node.js 16.x
node -v   # 确保版本为 14.17 或更高
npm install -g typescript
source ~/.bashrc #记得及时source使得环境变量生效demo
```

### 无法完成注册

注意`./app/frontend/.env`文件中一定要添加SESSION_SELECT否则无法完成注册

```
SESSION_SECRET="ad0nkey"
```

### cvv格式报错

注意一下订单输入的格式

### 启动user数据库无法连接

没有预先写好.env文件


# CloudWeGo基础


## 相关命令行指令

### `netstat`

`netstat`（Network Statistics）是一个用于显示 **网络连接、路由表、接口统计信息**、**masquerade 连接** 和 **多播成员信息** 的工具。功能如下：

- 查看网络连接状态。
- 查看端口的占用情况。
- 查看路由表。

#### 相关参数

##### **1. `-t`**

- **表示 TCP 连接**。
- `-t` 选项告诉 `netstat` 只显示 **TCP** 协议的连接。默认情况下，`netstat` 会显示所有类型的网络连接（包括 TCP 和 UDP），使用 `-t` 只会列出 **TCP** 连接。

##### **2. `-n`**

- **表示显示数字地址和端口**。

- `-n` 选项告诉 `netstat` **以数字形式显示 IP 地址和端口号**，而不是解析成域名和服务名。这个选项通常可以提高命令执行的速度，尤其是在没有 DNS 解析的情况下。

  例如：

  - 没有 `-n`：`localhost` 和 `http`。
  - 使用 `-n`：`127.0.0.1` 和 `80`。

##### **3. `-u`**

- **表示 UDP 连接**。
- `-u` 选项告诉 `netstat` 只显示 **UDP** 协议的连接。默认情况下，`netstat` 会显示所有类型的网络连接（包括 TCP 和 UDP），使用 `-u` 只会列出 **UDP** 连接。

##### **4. `-l`**

- **表示显示监听中的端口**。
- `-l` 选项会列出所有当前正在**监听**的端口。监听端口是指等待建立连接的端口，不是已经建立的连接。这个选项是查看系统中哪些服务正在监听特定端口时非常有用。

##### **5. `-p`**

- **显示与每个连接相关的进程信息**。
- `-p` 选项允许你看到与每个连接或端口相关的进程 ID (PID) 和进程名称。这样你可以知道是哪个进程正在使用某个端口。

##### **总结：**

`netstat -tnulp` 的组合使用会显示以下内容：

- **`-t`**：只显示 TCP 连接。
- **`-n`**：以数字形式显示地址和端口号。
- **`-u`**：只显示 UDP 连接。
- **`-l`**：仅显示正在监听的端口。
- **`-p`**：显示与每个连接相关的进程信息（PID 和进程名）。

------

### `lsof`

`lsof`（List Open Files）是一个 **列出系统中所有打开文件** 的工具。在 Unix 和 Linux 中，一切都是文件，包括网络连接和设备。

#### **常用功能：**

- 查看哪些文件被哪些进程打开。
- 查看占用端口的进程。

#### **常用命令：**

- **查看哪个进程占用了某个端口：**

  ```
  sudo lsof -i :3306
  ```

  这会列出占用 3306 端口的进程。

- **查看当前所有打开的文件：**

  ```
  lsof
  ```

  显示所有打开的文件，包括普通文件、设备文件和网络连接。

- **查看某个进程打开的文件：**

  ```
  lsof -p <PID>
  ```

  显示指定进程打开的所有文件。

------

###  `ps`

`ps`（Process Status）是用于显示 **当前正在运行的进程** 的工具。它可以查看进程的 PID、占用的资源等信息。

- 查看当前运行的进程。
- 获取某个进程的详细信息。

- **查看当前所有进程：**

  ```
  ps aux
  ```

  显示系统中所有进程的详细信息，包括 PID、CPU 使用率、内存使用率等。

- **查看指定用户的进程：**

  ```
  ps -u <username>
  ```

  显示指定用户的所有进程。

- **查看某个进程的详细信息：**

  ```
  ps -p <PID> -f
  ```

  显示指定 PID 进程的详细信息。

- **查看进程树：**

  ```
  ps aux --forest
  ```

  以树形结构显示进程及其子进程。

| 命令          | 功能                                                         |
| ------------- | ------------------------------------------------------------ |
| **`netstat`** | 查看网络连接、端口使用情况、路由表等。用于查看网络端口的连接状态。 |
| **`lsof`**    | 列出系统中所有打开的文件，包括网络连接和设备文件。用于查找占用端口的进程。 |
| **`ps`**      | 显示当前运行的进程及其资源使用情况。用于查看进程及其状态。   |

---

### `carbonyl`

```sh
carbonyl -f=240 http://127.0.0.1:8080  用于像素级浏览网页
```

## 配置开发环境

#### 1.配置go语言开发环境

```sh
Golang的官网下载地址：https://golang.org/dl/
下载golang代码解压放至 /usr/local/go
然后按照图示在$USER目录下 .bashrc中添加以下内容即可
```
![image-20250210102836479](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250210102836479.png)

Go语言测试代码如下: go run main.go

```go
// 在主文件中必须引入main的包
package main
 
import "fmt"
 
//通过找到该main()方法进行执行程序
func main() {
        fmt.Printf("Golang env Test-2025-01-09!\n")
} 
```



#### 2.配置zsh

<font color="red">Ubuntu中若是更换zsh，需要重新配置相关bashrc文件</font>

vscode配置zsh教程[vscode如何美化命令行？zsh工具安装教程分享-猿码集](https://www.yingnd.com/vscode/10955.html)

vscode终端配置教程[VSCode连接linux,配置zsh+p10k终端,踩坑经验 - 知乎](https://zhuanlan.zhihu.com/p/721436015)

vscode在settings.json添加下面这句话即可

```json
        "terminal.integrated.fontFamily": "MesloLGS NF",
```



#### 3.拉取字节相关go框架

## Go语言命令

### go mod init


1. **`go mod init github.com/cloudwego/biz-demo/gomall/hello_world`**
   这条命令用于初始化一个新的 Go 模块，`github.com/cloudwego/biz-demo/gomall/hello_world` 是该模块的路径。执行后会在当前目录下生成一个 `go.mod` 文件，内容类似于：

   ```
   go复制编辑module github.com/cloudwego/biz-demo/gomall/hello_world
   
   go 1.xx // 这里的 Go 版本会根据你的 Go 环境确定
   ```

   这个 `go.mod` 文件用于管理该模块的依赖项，使其能够被其他 Go 项目导入。

   例如，在 `go.mod` 里定义了：
   
   ```
   module github.com/cloudwego/biz-demo/gomall/hello_world
   ```
   
   那么在代码中 import 该模块的包时，需要使用：
   
   ```
   import "github.com/cloudwego/biz-demo/gomall/hello_world/somepackage"
   ```
   
   如果 `module` 设置得不正确，import 时可能会找不到包。
   
   如果你用的是其他路径，比如：
   
   ```
   go mod init myproject
   ```
   
   那么在其他 Go 代码里 import 时就应该使用：
   
   ```
   import "myproject/somepackage"
   ```
   
   但如果你后来把 `myproject` 代码上传到 GitHub（比如 `github.com/yourname/myproject`），其他人下载代码后可能会因为 import 路径不匹配而出错，需要手动修改 `go.mod` 里的 `module` 名称。

- `github.com/cloudwego/biz-demo/gomall/hello_world` 是人为指定的路径，并没有强制要求必须这样写，但推荐使用项目的实际 Git 仓库路径作为 `module` 名，以方便代码管理和导入。
- 如果项目不会托管到 GitHub，而是仅用于本地开发，可以随意起名，如 `mylocalproject`。
- 采用 `github.com/...` 这样的路径有助于团队协作和代码复用，尤其是当项目要被其他开发者

### go get


1. **`go get -u github.com/cloudwego/hertz`**
   这条命令用于获取并更新 `github.com/cloudwego/hertz` 这个 Go 依赖包。

   - `go get` 用于下载指定的 Go 依赖包。
   - `-u` 选项表示更新到最新的次要版本或补丁版本。

   运行后，它会：

   - 下载 `hertz` 库及其依赖项到本地 `GOPATH/pkg/mod` 目录（如果 Go Modules 启用，则存入 `go.mod`）。
   - 更新 `go.mod` 文件，记录 `hertz` 及其版本信息。
   - 可能更新 `go.sum` 文件，该文件记录了依赖包的哈希值，以确保依赖的完整性。



### go mod tidy

`go mod tidy` 是一个 **清理和同步 Go 依赖** 的命令。它会：

1. **添加缺失的依赖**（如果 `go.mod` 里有 import 但没有在 `go.sum` 里记录，会自动下载）。
2. **删除未使用的依赖**（如果 `go.mod` 里声明了某些依赖但代码里未使用，会移除）。
3. **更新 `go.sum` 文件**，确保所有依赖的版本和哈希值正确。

### go work use .

- **作用**：将当前目录（`.`）作为 Go 工作区（Workspace）的一部分。

- **背景**：Go 1.18 引入了 **Go Workspaces（`go.work`）**，用于管理多个模块。

- 具体功能

  ：

  - 如果当前目录是一个 Go Module（包含 `go.mod`），那么 `go work use .` 会将其加入 `go.work` 文件。
  - 这样可以让多个 Go 模块（多个 `go.mod`）协同工作，而不需要每次修改 `GOPATH`。
  - 适用于多模块项目，如微服务架构。

## IDL工具-CWGO

接口定义语言工具，常见的有protobuf和thrift两种工具。字节提供的是cwgo。
cwgo 是 CloudWeGo All in one 代码生成工具，整合了 kitex 和 hz 工具的优势。
**生成代码的脚手架类似于thrift。**主要有thrift和protobuf
cwgo官网[cwgo/README_CN.md at main · cloudwego/cwgo](https://github.com/cloudwego/cwgo/blob/main/README_CN.md)

**工具特点**

- 支持生成工程化模板

  cwgo 工具支持生成 MVC 项目 Layout，用户只需要根据不同目录的功能，在相应的位置完成自己的业务代码即可，聚焦业务逻辑。

- 支持生成 Server、Client 代码

  cwgo 工具支持生成 Kitex、Hertz 的 Server 和 Client 代码，提供了对 Client 的封装。用户可以开箱即用的调用下游，免去封装 Client 的繁琐步骤。

- 支持生成关系型数据库代码

  cwgo 工具支持生成关系型数据库 CURD 代码。用户无需再自行封装繁琐的 CURD 代码，提高用户的工作效率。

- 支持生成文档类数据库代码

  cwgo 工具支持基于 IDL (thrift/protobuf) 生成文档类数据库 CURD 代码，目前支持 MongoDB。用户无需再自行封装繁琐的 CURD 代码，提高用户的工作效率。

- 支持生成命令行自动补全脚本

  cwgo 工具支持生成命令行自动补全脚本，提高用户命令行编写的效率。

- 支持分析 Hertz 项目路由和(路由注册)代码的关系

  cwgo 支持通过分析 Hertz 项目代码获取路由和(路由注册)代码的关系。

- 支持回退为 kitex、Hz 工具

  如果之前是 kitex、Hz 工具的用户，仍然可以使用 cwgo 工具。cwgo 工具支持回退功能，可以当作 kitex、Hz 使用，真正实现一个工具生成所有。

![image-20250210100954812](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250210100954812.png)

安装cwgo工具

```go
# Go 1.15 及之前版本
GO111MODULE=on GOPROXY=https://goproxy.cn/,direct go get github.com/cloudwego/cwgo@latest

# Go 1.16 及以后版本
GOPROXY=https://goproxy.cn/,direct go install github.com/cloudwego/cwgo@latest
```



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

## 服务注册与服务发现

### ETCD

ETCD 是一个高可用的键值存储系统，通常用于分布式系统中作为配置管理和服务发现的工具。它是由 CoreOS 开发的，并作为 Kubernetes 中的关键组件之一。

##### 主要特点：

1. **一致性**：ETCD 使用 Raft 协议来确保数据一致性。无论你读写哪个节点，都会返回一致的数据。
2. **高可用性**：ETCD 通过多个节点（一般为奇数个）组成集群来提供高可用性，即使部分节点宕机，仍然能够保证系统的正常运行。
3. **强一致性和线性化读写**：ETCD 的数据模型提供强一致性和线性化读写，保证所有操作都是按照提交的顺序执行。
4. **分布式锁**：ETCD 支持分布式锁，用于协调分布式系统中的多个进程之间的访问控制。
5. **简洁的键值存储**：ETCD 采用简单的键值对（key-value）的存储方式，支持对数据的原子操作。
6. **事件通知**：ETCD 支持对键值对的变化进行监控，可以用于通知客户端系统数据变化，如配置文件变更等。

##### 使用场景：

- **配置管理**：ETCD 常被用来存储和管理分布式系统的配置文件，支持动态配置的修改。
- **服务发现**：在微服务架构中，ETCD 可以用来进行服务注册和发现，帮助系统在不同节点间进行通信。
- **分布式锁和协调**：在多个进程或多个系统之间，ETCD 可用于管理分布式锁，协调不同进程的执行顺序。
- **Kubernetes** 使用 ETCD 作为其后端存储系统来保存所有集群的状态数据，包括配置信息和服务状态等。

##### **数据存储：**

ETCD 的数据结构非常简单，基于键值对存储，每个键值对可以存储一个二进制数据。ETCD 在存储过程中支持原子操作，如事务（watch、compare-and-swap 等）和版本控制。

##### 安全性：

ETCD 支持通过 TLS 加密传输数据，确保数据在网络中传输时的安全性。此外，它还可以通过设置访问控制策略和认证机制，确保只有授权的客户端才能访问数据。

ETCD 通过强一致性和高可用性成为现代分布式架构中不可或缺的一部分，尤其在容器编排和微服务环境中广泛使用。

![image-20250211111748382](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250211111748382.png)

- 为什么需要服务注册与服务发现：未来高可用性与拓展方便，不至于一直去修改代码
- 如何去实现服务注册与服务发现：市面上很多注册中心 ETCD，Zookeeper，consul等等，如下图所示我们可以通过加一层中间层C，实现服务注册与服务发现
- 如何去选择注册中心：Kitex支持市面上常见的注册中心
- Kitex中如何选择服务注册与服务发现

如何去实现服务注册与服务发现：**核心点在于微服务的拆分**，这里C只提供注册实例的作用，当作一个注册中心，B服务增减实例都告知C并存储，A会定时向C获取（query查询）B的实例列表，A会在C的实例中选择一个直接去调用B

![image-20250220102550096](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250220102550096.png)

服务的实例信息如何存储，Kitex中可以和兼容目前主流注册中心，比如`consul, etcd, zookeeper`等等

关于怎么去选择注册中心-CAP理论 consistency一致性 availability可用性 Partition Toleration 分区容错性,三者只能实现其二

![image-20250220110521375](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250220110521375.png)

```go
docker compose up -d
go run . //注意修改配置中心的ip和端口
//需要匹配好对应的服务 demo_proto 以及request中输出对应message
```



### LB负载均衡器

在微服务架构中，**LB**（Load Balancer，负载均衡器）是一个重要的组件，主要负责在多个服务实例之间分配流量，确保系统的高可用性、性能优化以及有效的资源利用。

#### 1. **LB（Load Balancer，负载均衡器）的作用**

负载均衡器的主要作用是 **将客户端的请求均匀地分配到多个服务实例**，避免某个服务实例的过载。负载均衡的核心目标是提高系统的可扩展性、容错性和性能。其具体作用包括：

- **流量分配**：将传入的请求分配到多台服务器或服务实例中，防止单个实例过载。
- **高可用性**：当某个服务实例发生故障时，负载均衡器会自动将请求路由到健康的实例，从而实现高可用性。
- **性能优化**：通过合理的流量分配，确保系统性能的最大化，避免某一节点成为瓶颈。
- **故障恢复**：监控服务实例的健康状态，一旦某个实例不可用，自动从负载均衡池中剔除该实例，防止请求发往故障实例。
- **服务发现**：负载均衡器通常与服务注册中心（如 Consul、Eureka）配合工作，动态获取可用服务实例，并基于这些实例进行负载均衡。

#### 2. **LB的工作方式**

负载均衡器的工作方式通常有几种，常见的包括：

- **轮询（Round Robin）**：将请求均匀地按顺序分配给所有服务实例。
- **加权轮询（Weighted Round Robin）**：为每个服务实例分配一个权重值，根据权重决定分配的流量。
- **最少连接（Least Connections）**：将请求转发到当前连接数最少的服务实例。
- **IP哈希（IP Hash）**：基于客户端的IP地址，将请求分配到某个特定的服务实例，确保同一客户端的请求始终发送到同一实例。
- **健康检查（Health Checks）**：负载均衡器定期检查服务实例的健康状况，自动剔除不健康的实例。

#### 3. **服务注册（Service Registration）**

在微服务架构中，服务注册是指服务实例向一个服务注册中心报告自己的信息（如 IP 地址、端口号、健康状态等），这样其他服务或负载均衡器就能够发现并调用该服务。

- **服务注册**：每个服务实例会在启动时向服务注册中心（如 Consul、Eureka、Zookeeper）注册自己的信息。注册信息通常包括服务名称、实例IP地址、端口、健康检查URL等。
- **服务发现**：其他服务或负载均衡器可以通过服务注册中心查询到当前可用的服务实例列表，实现 **服务发现**。

#### 4. **LB 和 服务注册的联系**

负载均衡器和服务注册是微服务架构中两个密切相关的组件，二者协同工作以保证系统的高可用性和可靠性。

- **服务注册中心的作用**：服务实例向服务注册中心注册自己的信息，并更新自己的健康状态。服务注册中心充当服务发现的角色，提供了可用服务实例的信息。
- **负载均衡器的作用**：负载均衡器根据服务注册中心提供的服务实例列表来进行流量分配。当服务实例发生变化时（例如某个实例下线或新增实例），负载均衡器可以根据注册中心动态更新服务实例列表，并做出流量的重新分配。

#### 流程：

1. **服务注册**：当一个新的服务实例启动时，它向服务注册中心注册自己的信息。
2. **服务发现**：负载均衡器从服务注册中心获取到服务实例的最新列表。
3. **请求路由**：负载均衡器根据获取到的实例列表，将客户端请求路由到合适的服务实例。

#### 举个例子：

假设有一个微服务应用，服务 A 需要调用服务 B。负载均衡器会根据服务注册中心（如 Eureka）提供的服务实例列表（包含服务 B 的多个实例）来分配流量。当服务 B 中的某个实例不可用时，负载均衡器会自动将请求转发到其他健康的实例，确保服务的高可用性。

#### 5. **总结**

- **LB（负载均衡器）**：负责将流量分配到多个服务实例，优化资源使用，提供高可用性和故障恢复能力。
- **服务注册**：服务实例将自己的信息注册到服务注册中心，供其他服务或负载均衡器查询和发现。
- **联系**：负载均衡器依赖服务注册中心提供的可用服务实例信息来进行流量的合理分配。服务实例的健康状态和变更会动态影响负载均衡器的工作。

通过服务注册与负载均衡的结合，微服务架构能够在动态环境中实现灵活的流量管理和故障恢复，提高系统的弹性和可伸缩性。

## 配置管理

常见配置方式：文件配置，环境配置，配置中心

- 文件配置即是将配置写入文件：yaml， json， toml
- 环境配置即是类似于环境变量
- 配置中心类似于consul有配置中心统一对各个服务进行管理 

### 文件配置

cwgo提供的脚手架为yaml格式。go项目中使用配置文件需要预先写好一个对应的结构体 `conf/conf.go`负责解析所有配置文件 `dev online test`分别代表开发/上线/测试。`conf.go`中定义了Kitex相关配置，MySQL,redis以及服务注册与发现相关配置。首先通过环境变量GO_ENV  确定环境

![image-20250211113146553](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250211113146553.png)

### 环境配置

常见环境配置有如下几种：

- Linux原生的环境变量，例如在编译内核的时候，我们经常使用这种方式来完成环境变量的声明，例如所使用的架构或者编译工具等等
- 加载`.env`文件，我们将预先准备好的对应的环境变量写入`.env`文件，调用时直接加载即可
- Docker容器加载镜像或k8s加载镜像

我们可以将mysql的相关配置换成占位符%s, 这样在项目启动的时候，我们就可以通过环境变量来配置这四个选项，例如export

```sh
MYSQL_USER, 
MYSQL_PASSWORD, 
MYSQL_HOST, 
MYSQL_DATABASE。
```

另外，如果我们使用加载`.env`文件，需要额外引入其他的GO语言库`GoDotEnv`，还需要在docker-compose添加依赖。

docker-compose up启动镜像及相关配置，最后我们还需创建对应的数据库

``` 
docker-compose exec mysql bash
mysql -u root -p
show databases;
create database test;
```

![image-20250211113352723](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250211113352723.png)

![image-20250211160900575](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250211160900575.png)

```go
type Version struct{
    Verison string
}
var v Version
err = DB.Raw("select version() as version").Scan(&v).Error
if err != nil {
    panic(err)
}

fmt.Println(v)
```

### 配置中心

![image-20250211170925720](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250211170925720.png)  

## 使用ORM进行数据操作

- 使用标准库访问数据：标准库需要我们写原生的SQL语句，但是效率不高，实际开发中我们可能会选择第三方的ORM库进行开发.CloudWeGo的脚手架已经支持了GORM的代码生成
- 常见ORM
- 使用GORM进行增删改查  

环境改造和确认涉及到两个方面：`docker-compose`和`.env`文件

MYSQL_DATABASE创建数据库的时候可以指定数据库名称

**GORM**：作为 ORM 工具，它本身不提供数据库存储功能，只是简化了与 MySQL 等数据库交互时的事务管理和数据操作。处理复杂的查询和数据持久化。

## 编码指南

- 代码规范：Go语言本身错误码；protobuf错误码
- 错误码
- 日志实现
- 提交规范
- 版本命名规范

uber-go：Go语言编码指南；`golint go vet goimports`

HTTP状态码

![image-20250228152412249](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250228152412249.png)

日志库 ：hertz-contrib/logger

![image-20250228152806831](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250228152806831.png)

提交规范：约定式提交 conventional commit

```scss
feat(auth): add login functionality				//添新功能
fix(db): resolve connection timeout issue		//修复bug
docs(readme): update API usage examples			//修改文档
refactor(ui): refactor button component 		//重构代码
perf(api): optimize request handling			//性能优化
test(auth): add unit tests for login function	//添加测试
```

# 电商项目中间件

1. **MySQL**：存储电商平台的核心业务数据（如用户信息、商品、订单、支付信息等），保证事务的原子性。
2. **Redis**：缓存商品信息、用户会话、热门搜索等，提升系统的性能；也可以作为消息队列处理订单异步任务。
3. **ETCD**：配置管理、服务发现、分布式锁，确保微服务架构中的配置一致性和高可用性。
4. **GORM**：简化与 MySQL 的交互，提供 ORM 功能，便于进行数据库操作和事务管理。
5. **Zookeeper**：用于服务发现、分布式锁和协调，确保系统的高可用性和一致性。
6. **Kafka**：处理异步任务和消息通知，构建事件驱动架构。
7. **ETCD** 和 **Zookeeper** 都用于分布式系统中的数据一致性、服务发现、分布式锁等功能，它们主要的区别在于实现协议（Raft vs. ZAB）和生态系统的应用。ETCD 比较轻量，专注于简单的键值存储和强一致性，而 Zookeeper 提供更广泛的分布式协调功能，如选举机制等。
8. **Kubernetes** 是一个容器编排平台，虽然它本身并不提供配置存储或分布式协调功能，但它依赖 **ETCD** 来存储集群的状态数据。Kubernetes 通过容器化技术实现应用的自动化部署、扩展和管理，解决了大规模微服务架构中容器的编排和管理问题。
9. **ETCD** 是 Kubernetes 的核心组件之一，负责保存和协调 Kubernetes 集群的状态数据，因此 ETCD 对 Kubernetes 的高可用性至关重要。
10. 在 **微服务架构** 中，通常会使用 **ETCD** 或 **Zookeeper** 来管理分布式服务的配置和协调，而 **Kubernetes** 负责应用的容器编排和管理，三者在分布式系统中各自担任不同的角色。

### **MySQL**

- Mysql数据库是一款存放和管理数据的软件，介于应用和数据之间，通过一些设计，可以将大量数据变成一张张像excel的数据表，为应用提供创建、读取、更新、删除等核心操作。
- **数据页**是什么，Mysql将数据组织成excel表的样子，excel在磁盘上是`.xls`的文件格式。Mysql的数据表也类似，在磁盘上是个`.ibd`后缀的文件，保存的数据越多，这个`.ibd`文件越大，Mysql直接读取大文件速度也会很慢，所以Mysql将数据拆分成一个一个数据页，每个页大小16KB，这样我们读取部分表的数据的时候，只需要读取磁盘里面的几个数据页就行。
- **索引**是什么：数据页很多，查某条数据时，我们怎么要知道读哪些数据页，我们可以为每隔数据页键入页号，然后为每行数据加个序号，**序号就是所谓的主键**，按主键大小排序，将每个数据页里最小的主键序号和所在页的页号提取出来，放入到一个新生成的数据页中，并且给数据页加入层级的概念，这样我们就可以通过上层的数据页，快速缩小查找范围，加速查找数据页的过程，现在页和页之间就是一颗倒过来的树，这颗可以加速查找数据页的树就是我们常说的B+树索引。上面提到的时**主键索引**。按同样的思路，我们也可以为其他数据表的列去建立索引，比如用户表的名称字段，这样我们就能快速查找到名字为xx的用户有哪些，这就是所谓的**辅助索引**
- `Buffer Pool`：有了索引之后，数据还是在磁盘上，每次读写磁盘慢。**我们可以在磁盘和应用之间加一层进程内缓存，缓存里装的就是前面提到的16KB数据和索引页**，它就是所谓的`Buffer Pool`，读取数据的时候优先读`Buffer Pool`，有数据就返回，没数据再去磁盘里面读取，大大减少了读取磁盘的次数。文件读取默认会先将文件数据加载到操作系统的文件缓存中，同样都是缓存，为什么还需要`Buffer Pool`，因为进程自己维护的`Buffer Pool`可以自己定制更多的缓存策略，还可以实现加锁等各种数据表高级特性，所以已经有了`Buffer Pool`,我们就没必要继续使用操作系统自带的文件缓存了，**所以`Buffer Pool`通过直接IO模式绕过操作系统的缓存机制，直接从磁盘读取数据。**
- **自适应哈希索引**：就算有了Buffer Pool，要查到某个数据页，也依然要查找B+树，查询复杂度`O(lgN)`，为了优化我们可以使用查询复杂度为O(1)的哈希表，记录每个数据页的查询频率，对于热点数据页，我们以查询到的值为key，数据页地址为value构建哈希表。这个哈希表就是所谓的**自适应哈希索引Adaptive Hash Index**
- **Change Buffer**：大部分数据表，除了主键索引外，还会添加一些辅助索引。比如对用户名添加辅助索引。对于这类数据表的写操作，更新完主键索引的数据页之后，还需要更新辅助索引页，**这样读取辅助索引页的磁盘IO是无法避免的**。有了自适应哈希索引，读性能优化了，那么写性能的优化如下：**先将要写入的数据收集到一块内存里，等哪天磁盘的索引页正好被读入Buffer Pool的时候，再将写入数据应用到索引页中，通过这个方式，减少大量磁盘IO提升性能。这个将写操作收集起来的地方就是所谓的Change Buffer。它是Buffer Pool的一部分。**
- **Undo Log**：**数据库中事务的概念，也即让多行数据要么同时更新成功，要么同时更新失败，也就是所谓的原子性**，为了实现这一点，我们需要知道写数据时每行数据原来长啥样，方便对更新后的数据行进行回滚，因此有了Undo Log,更新Buffer Pool数据页的时候，会用旧数据生成Undo Log记录，存储到Buffer Pool中的特殊Undo Log内存页中 ，并随着**Buffer Pool的刷盘机制**，定时写入到磁盘的Undo Log文件
- **Redo Log**：上面提到的都是Buffer Pool相关的内容，本质上都是内存，如果内存数据只写了一半到内存中，数据库进程崩溃，一个事务里面的多行数据就没能做到同时更新，解决办法如下：我们将事务中更新数据行的操作都写入到Redo Log Buffer内存中，然后在事务提交的时候，进行Redo Log刷盘，将数据固化到磁盘中的Redo Log文件中即可，数据库进程崩溃重启之后，，就能通过Redo Log File找到历史操作记录重做数据，保证了事务里的多行数据变更要么都成功，要么都失败 。为什么不讲Buffer Pool中的数据直接写入磁盘：二者差异在于Redo Log File是有顺序追加写效率更高 ，Buffer Pool中的内存数据是随机分散在磁盘各处的。顺序写磁盘时随机写的性能的几十倍，所以很多存储系统写数据时，都会搞一个日志来记录操作，方便服务重启之后，进行数据对账，确保数据的一致性和完整性。这类操作就是所谓的**Write Ahead Logging（WAL）**
- **InnoDB**:我们将提到的内容分为两个部分，**一个内存中的自适应哈希、Buffer Pool以及Redo Log Buffer，另一部分是存放行数据和索引的`.ibd`文件以及Undo log、Redo log文件，他们共同构成了InnoDB存储引擎，并对外提供一系列的函数接口**。我们平时写的SQL语句最终都会转换成InnoDB提供的接口函数调用
- Server层:Server层和存储引擎层共同构成了Mysql数据库并且Server层和存储引擎层是通过接口函数进行解耦的，只需要实现接口函数，即可作为存储引擎与Server层对接。Mysql早期使用`MyISAM`，后期使用`InnoDB`
- **`binlog`为了防止删库跑路，Server层会将历史上的所有变更操作记录到磁盘上的日志文件中，这个日志文件就是所谓的`binlog`。**`binlog` 不止是用来和备份恢复误删数据的，在恢复中 `binlog` 和 `redo log` 有二阶段提交，用来确认数据到底有没有刷盘。另一方面 `redo log` 是`innodb` 提供的，也不是 MySQL 都有
- 数据库查询更新流程：首先不管是查询还是更新操作，客户端都会先跟Mysql建立网络连接，并将SQL发送到Server层，经过分析器解析SQL语法，优化器选择索引生成执行计划，最终给到执行器调用`InnoDB`的函数接口。
- 对于读操作，InnoDB存储引擎会先检查Buffer Pool中是否存在所需的B+树数据页，如果存在则直接返回数据，如果Buffer Pool中没有所需的数据页，则会从磁盘中读取相应的数据页加载到Buffer Pool中再返回数据。同时，如果查询到的数据是热点数据，还会将数据加载到自适应哈希索引中,加速后续的查询。
- 对于写操作：会先将数据写入Buffer Pool，并生成相应的Undo Log记录，以便在事务回滚时能够恢复到数据的原始状态，接下来，会将写操作记录到Redo Log Buffer,这些Redo log会周期性的写入磁盘的Redo log文件，即使数据库崩溃，已提交的事务也不会丢失，对于辅助索引的更新操作，InnoDB会将这些更新暂时存储再Change Buffer中，等到相关的索引页被读取到Buffer Pool时，在进行实际的更新操作，从而减少磁盘IO，提高写入性能，同时所有的变更都会记录到Server层的`binlog`中，以便进行数据恢复.



![image-20250212202857310](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212202857310.png)

![image-20250212203002747](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212203002747.png)

![image-20250212203437168](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212203437168.png)

![image-20250212203754076](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212203754076.png)

Buffer Pool刷盘机制

![image-20250212211027906](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212211027906.png)

刷盘的出发条件

![image-20250212211049487](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212211049487.png)

刷盘过程的优化

![image-20250212211215028](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212211215028.png)

**使用场景**：

- **业务数据存储**：MySQL 是电商项目中最常见的关系型数据库，存储各种结构化的业务数据，如用户信息、商品信息、订单数据、购物车数据等。
- **事务管理**：电商交易系统需要强一致性的保证，MySQL 支持 ACID 事务，确保数据的一致性和完整性，特别是在订单创建、支付、库存扣减等操作时非常重要。
- **复杂查询**：MySQL 支持复杂的 JOIN 查询、聚合查询和索引优化，适用于电商项目中对商品搜索、订单统计、用户行为分析等功能的需求。

**具体应用**：

- **用户管理**：用户注册、登录、信息管理等数据存储。
- **商品管理**：商品分类、库存、价格、促销活动等数据存储。
- **订单系统**：订单的创建、支付状态、物流跟踪等。
- **数据分析**：如销售报表、客户行为分析等。

![image-20250212160937276](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212160937276.png)

### **Redis**

- 查询内存的速度比查询磁盘要快，Mysql数据主要存放在磁盘里，如果能将Mysql中数据放到内存中，性能必然会大大提升，我们可以在商品服务中去构建一个类似于字典的数据库，在发生查询时优先去查询字典，没结果再去Mysql数据库中查询，再将结果放到内存字典中返回用户。像这种在服务内存放数据的内存就是本地缓存，有了本地缓存，真正打到Mysql的查询量就很少。查询大导`10k QPS` 即不难实现
- 为了保证系统高可用，商品服务经常不止一个实例， 如果每个实例都重复缓存一份本地内存，过于浪费内存条，更好的解决方案是将这部分字典内存抽出来，**单独做成一个服务，即是远程缓存服务**，多个商品服务去读写同一份远程缓存，会存在并发问题，解决办法可以是在单线程中对字典进行读写
- 优化：**支持多种数据类型**，目前缓存服务中只有map一个字典类型，`key value`都是字符串,我们可以对字段value进行扩展，除了字符串，还支持先进先出队列list和用于去重的set再加入可以做排行榜的zset
- **内存过期策略**:缓存服务支持的数据结构变多之后，塞到内存中的数据越来越多，我们可以给内存中的数据加个过期时间，数据过期就从内存删掉，可以很大程度延缓内存增长速度。交给调用方去判断，让用户通过`expire`命令指定哪些数据多久过期。
- **缓存淘汰**：如果大家都不设置过期时间，我们可以根据一些策略删除掉一些内存，比如可以将最近最少使用的内存删掉，也就是`Least Recently Use LRU`算法，不仅解决了内存数据过大的问题，还是的内存中的数据全是热点数据
- **持久化**：我们需要给远程缓存服务加入最大程度的持久化保证 。确保服务重启后，内存中不至于什么数据都没有，于是我们可以在远程服务里加个异步子进程，定期将全量内存数据，持久化到内存磁盘文件中。这种将内存数据生成快照保存到文件的方式就是所谓的RDB，它可以每隔几分钟记录下缓存服务的全量数据，类似于游戏的存档，这样即使进程挂了重启，通过加载快照文件就能恢复大部分数据。大部分是因为存档之后写入的数据可能会丢。全量保存数据过于耗时，我们可以在每次写入数据时，顺手将数据记录到文件缓存中，并每秒将文件缓存刷入磁盘，这种持久化机制叫做`AOF`，服务启动时，跟着文件重新执行一遍就能将大部分数据还原，最大程度上保证了数据持久化
- Redis（Remote Dictionary Server）远程字典服务。通过命令行` redis-cli`我们可以输入一些命令，读写redis服务器中的数据。Redis放在Mysql前挡一道查询只是基础用法，通过其他扩展插件可以实现高级用法：`RedisJson`可以支持复杂的`Json`查询和更新，即是内存版本的`MongoDB`，`RedisSearch`支持全文搜索等等。
- `Redis`单线程快的原因：在内存上执行/单线程减少了线程切换的开销/采用的IO多路复用，提高网络请求速度

![image-20250212111542429](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212111542429.png)

![image-20250212111927398](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212111927398.png) 

![image-20250212152047455](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212152047455.png)

**使用场景**：

- **缓存系统**：Redis 是一个高性能的内存数据库，广泛用于缓存热点数据，减少数据库的访问压力，提升系统性能。例如，可以将商品信息、用户信息、热门搜索词、订单状态等数据缓存到 Redis 中。
- **会话管理**：Redis 可以存储用户会话信息，用于实现用户的登录状态管理。它可以存储在用户登录后生成的 token，减少数据库负担。
- **队列系统**：Redis 支持列表数据结构，可以用作消息队列，在电商项目中，通常用于订单处理、消息通知等异步任务。例如，在用户下单后，订单信息可以通过 Redis 的列表发送到一个消费队列，由后台服务进行处理。

**具体应用**：

- **商品缓存**：缓存商品详情、热门商品、促销商品等，避免每次查询都访问 MySQL 数据库。
- **用户会话**：存储用户的登录状态和身份验证信息，避免每次都从数据库验证。
- **秒杀活动**：秒杀场景下，通过 Redis 实现商品库存和秒杀请求的高效管理。
- **异步任务处理**：使用 Redis 的列表和发布/订阅模式处理订单异步任务。

### **ETCD**

**ETCD** 是一个高可用的分布式键值存储系统，通常用于存储和共享配置数据、服务发现、分布式协调等任务。它是 **CoreOS** 提出的一个项目，是 **Kubernetes** 的核心组件之一，主要用于保存集群的状态数据。ETCD 基于 **Raft 算法** 来保证数据的一致性，并且支持分布式集群架构。

#### **ETCD原理**

**1. Raft 算法**

Raft 算法是 ETCD 的核心，它用于解决分布式系统中的一致性问题。Raft 提供了强一致性，确保所有的数据更新都会被一致地复制到所有节点。Raft 算法的核心要素有：

- **Leader 选举**：在一个 Raft 集群中，总有一个节点被选举为 **Leader**，Leader 负责处理所有的写请求（即数据修改）并将修改复制到其他节点。
- **日志复制**：所有的写操作都需要先被 **Leader** 节点执行，并将操作日志同步到集群中的其他节点。其他节点成为 **Follower**，它们只负责接收 Leader 节点的操作。
- **一致性**：所有节点（Leader 和 Follower）确保它们存储的日志是顺序一致的，因此任何时候，集群内的数据都会保持一致。
- **心跳机制**：Leader 节点定期发送心跳信号，告知其他节点自己依然存活，防止发生 Leader 失效后没有新 Leader 产生的情况。

**2. 集群架构**

ETCD 支持分布式部署，可以由多个节点组成一个集群。ETCD 集群的成员数通常为奇数，以便于更好地进行 Leader 选举和分区容忍。集群内的节点通过网络通信相互同步数据。常见的 ETCD 集群配置为 3、5 或 7 个节点。

**3. 强一致性和线性一致性**

ETCD 保证 **强一致性**，意味着在 ETCD 集群中的所有读取操作，都能获取到最新的写入数据。即使在网络分区或节点故障的情况下，Raft 算法也能保证集群内的数据一致性。

另外，ETCD 提供 **线性一致性**，即每次对 ETCD 的读取都能保证是最新的写入。

#### **使用场景**

- **配置管理**：ETCD 是一个高可用的键值存储系统，常用于存储和管理分布式系统中的配置。电商项目中的配置，如数据库连接信息、第三方 API 配置、微服务的路由信息等，都可以使用 ETCD 管理。
- **服务发现**：在微服务架构中，ETCD 可以作为服务注册中心，帮助各个微服务实例注册自己的信息，并且其他服务可以查询和发现其他服务的地址。
- **分布式锁**：ETCD 可以用来实现分布式锁，确保在多节点的环境中对共享资源的访问是互斥的。例如，在高并发的订单处理系统中，可以使用 ETCD 来防止多个请求同时处理同一个订单。

**微服务中具体应用**

- **微服务配置管理**：存储各服务的配置信息，比如数据库配置、接口限流策略等。
- **服务注册与发现**：在微服务架构中，使用 ETCD 注册服务，提供动态发现。
- **分布式锁**：避免在订单支付过程中多次扣款，使用 ETCD 锁住订单，确保只有一个请求能够成功修改订单状态。

#### ETCD/Zookeeper比较

- 协议和实现：ETCD基于Raft协议，Raft是一种一致性算法，用于确保分布式系统的日志一致性，ETCD通过Raft协议管理集群中的数据一致性，适合K8S等系统作为配置存储的后端。Zookeeper使用ZAB协议（Zookeeper Atomic Broadcast）用于保证事务的顺序性和一致性，在早期的分布式系统中有广泛应用。如HBase、Kafka。Zookeeper相较于ETCD更加成熟且被大规模分布式系统应用。
- **ETCD**：是一个 **轻量级的键值存储**，它更多是作为 **配置存储** 和 **服务发现** 的工具。它专注于存储少量关键数据（如配置、状态信息等）并保证高可用性和一致性。**Zookeeper**：更多的是一个 **分布式协调框架**，它不仅用于存储数据，还提供更丰富的分布式协调功能，如选举机制、分布式锁、事件监听等。Zookeeper 更适合用于 **协调多个服务的行为**，例如在大数据系统中，Zookeeper 用于管理分布式数据存储的元数据。
- ETCD更多的用于配置管理和服务发现以及容器化应用（如 Kubernetes）的 **集群状态管理**。**Zookeeper** 常用于大规模分布式系统中的 **分布式协调**，如 **分布式锁**、**选举机制**。以及 分布式事务管理** 和 **大数据系统**（如 HBase、Kafka 等）中的元数据存储和**高一致性要求的分布式系统**，尤其是需要对节点进行细粒度管理的场景。

### **GORM**

**使用场景**：

- **ORM (Object-Relational Mapping)**：GORM 是 Go 语言中常用的 ORM 框架，可以简化与数据库（如 MySQL）的交互。在电商项目中，GORM 用于将数据库中的表映射到 Go 语言中的结构体，使得数据的存取更加简洁和高效。
- **数据库迁移**：GORM 支持数据库的自动迁移，可以根据定义的结构体生成相应的数据库表，也支持结构体变更时的数据库更新。

**具体应用**：

- **简化数据库操作**：通过 GORM，开发者可以在电商项目中使用 Go 语言的结构体而不需要写繁琐的 SQL 语句，简化了与 MySQL 的交互。
- **自动数据库迁移**：GORM 自动迁移功能，简化了数据库表结构的变更和管理，适用于开发过程中数据库结构的频繁调整。
- **事务管理**：GORM 提供对事务的支持，能够确保订单的原子性，避免并发操作时数据出现不一致的情况。

### **Zookeeper**

在分布式系统中，多个节点协同工作，需要进行数据共享，状态同步和协调操作，都需要一致性和可靠性的支持。

- **Zookeeper可以实现配置管理**，各个节点可以从Zookeeper中获取配置信息，这样当配置变化时，所有节点能够即使感知并进行相应调整
- **Zookeeper可以用作命名服务，类似于分布式的文件系统**，它允许应用程序在Zookeeper上创建删除和查找节点，从而实现简单的命名空间管理
- **分布式锁，Zookeeper提供了分布式锁的支持，允许多个节点在共享资源上进行协调，避免并发访问冲突**
- 分布式队列，可用于多个节点之间传递消息和任务
- 分布式通知，Zookeeper的Watcher机制可以让客户端监视节点的变化，并在节点状态发生变化时接收通知，实现分布式的事件触发和通知机制
- **leader选举**，在Zookeeper集群中，ZAB协议用于选举leader节点，leader负责处理所有客户端的写请求，并将更改广播给其他follower，可以帮助分布式系统实现一致性和可靠性的数据管理，以及实现分布式节点之间的协调和通信，广泛应用于分布式数据库/缓存/计算灯各种分布式应用

<font color="red"> 1.Zookeeper类似于一个分布式锁的服务，它解决了多个应用进程服务访问共享资源的一个顺序性的问题</font>
<font color="red"> 2.Zookeeper为应用服务集群，提供了一个选举能力，业务服务A可以选举出一个leader，leader有资格去执行定时任务，当leader节点挂了之后，follower节点会选举成新的leader，继续执行定时任务，Zookeeper可以实现这样的一个选举能力</font>
<font color="red">3.Zookeeper设计初衷是为了解决分布式协调和管理问题，当然现在也可以用于实现配置中心，注册中心，早期开源的Dubbo就是利用Zookeeper来实现注册中心的</font>

![image-20250212091550699](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212091550699.png)

### **RPC**

- 将结构体转化成01二进制数组的过程就叫做序列化，反之则称为反序列化
- 对于一般的Client/Server架构，可以使用自己的RPC服务连接自己的公司服务器即可，对于浏览器browser，不仅要访问自家公司服务器，还需要访问其他公司的网站服务器，HTTP就是统一BS架构的协议（Browser Server）
- 现在很多软件都支持多端，比如百度网盘，既支持BS架构网页版，又支持CS架构客户端手机版，如果通信协议都使用HTTP，那么服务器只用一套就可以了，RPC就开始退居幕后，用于公司内部集群用于各个微服务之间通信
- HTTP1.1设计出来是用于做网页文本展示，header和body主要是以字符串为主，body使用json来序列化结构体数组，RPC定制化程度更高，可以使用体积更小的protobuf或其他序列化协议去保存结构体数据，同时也不需要像HTTP一样考虑各种跳转如302重定向跳转，性能会更好一些，公司内部微服务抛弃HTTP选择RPC的原因。GRPC底层都直接 使用HTTP2。HTTP2性能更优

### **Kafka**

**使用场景**：

- **异步消息传递**：Kafka 是一个高吞吐量的分布式消息队列，电商项目可以使用 Kafka 进行异步消息处理，如订单处理、库存扣减、消息通知等。Kafka 适合用于大规模的消息流转场景。
- **事件驱动架构**：电商平台可以通过 Kafka 构建事件驱动的架构，所有的系统事件（如订单创建、支付成功等）都可以通过 Kafka 发布和消费。

**具体应用**：

- **订单处理流程**：用户下单后，将订单信息发送到 Kafka 中，后台服务消费 Kafka 消息进行异步处理（如库存扣减、物流分配等）。
- **消息通知**：通过 Kafka 向用户发送订单状态、促销信息等通知。

### **Kubernetes (K8s)**

**Kubernetes (K8s)** 是一个开源的容器编排平台，用于自动化容器的部署、扩展和管理。Kubernetes 使得在大规模分布式系统中管理容器化应用变得更加简单和高效。Kubernetes 本身并不是一个配置存储或协调系统，但它依赖 **ETCD** 来存储集群的所有状态数据。

- **功能**：
  - **容器编排**：Kubernetes 主要功能是管理和调度容器化应用的生命周期，包括容器的部署、扩展、监控、健康检查等。
  - **自动化管理**：Kubernetes 提供了自动扩展、负载均衡、容错等功能，可以根据应用的需求自动调整资源。
  - **服务发现**：Kubernetes 提供内建的服务发现机制，容器可以通过 K8s 的 DNS 服务发现其他服务。
  - **存储管理**：Kubernetes 支持自动挂载存储卷，以便容器能够持久化数据。
- **常见用途**：
  - 管理容器化应用的部署和扩展。
  - 微服务架构中，作为服务管理、编排和容器编排的平台。
  - 自动化运维、弹性伸缩和容错。

### **Docker **

**Docker** 是一个开源的容器化平台，它可以将应用及其依赖打包到容器中，并提供一种标准化的方式来构建、打包、分发和运行应用。Docker 使得在不同环境之间迁移和部署应用变得非常简单。

**主要功能**：

- **容器化**：Docker 使得应用和其依赖打包成独立的容器，确保在任何环境中一致地运行。
- **环境隔离**：每个容器都在自己的独立环境中运行，减少了环境配置带来的问题。
- **应用部署**：Docker 支持将应用程序分解为多个服务，每个服务部署在不同的容器中（例如，一个服务可能是数据库，另一个服务可能是 Web 服务器）。

### Docker/K8S/Zookeeper/ETCD

#### 横向应用对比

![image-20250212143450445](https://adonkey.oss-cn-beijing.aliyuncs.com/picgo/image-20250212143450445.png)

#### **电商项目实例：秒杀活动**

在电商项目中，**Zookeeper**、**Docker** 和 **Kubernetes (K8s)** 各自有着不同的角色，尽管它们在某些场景下的功能可能会有重叠，但它们通常在项目中共同发挥作用，确保系统的高可用性、可扩展性、容错性和灵活性。假设我们正在开发一个电商平台，平台在某个特定时间会举行秒杀活动，许多用户同时抢购限量商品。这种场景需要在后台系统中协调库存、订单、支付等多个环节，并保证系统的高并发、高可用和数据一致性。

##### **1. Docker：容器化应用**

在这个电商项目中，我们使用 **Docker** 将各个微服务打包成容器。这些服务可能包括：

- **用户服务**：处理用户注册、登录、个人信息等。
- **商品服务**：管理商品的分类、详情、库存等。
- **秒杀服务**：处理秒杀活动的启动、商品库存的扣减、用户的抢购请求等。
- **支付服务**：处理支付操作，调用支付网关进行支付授权。
- **订单服务**：管理用户订单的生成、状态变化和查询。

**如何使用 Docker**：

- 每个微服务都在一个独立的 Docker 容器中运行，确保环境的一致性。
- 微服务的代码、配置和依赖都被打包在容器中，便于在不同环境（开发、测试、生产等）之间迁移和部署。
- 通过 Docker，开发团队可以在本地环境快速启动完整的电商系统，并且可以方便地进行测试和调试。

**Docker 的优点**：

- 提供一致的运行环境，消除了不同开发环境、测试环境和生产环境之间的差异。
- 便于服务的扩展和快速部署，例如秒杀活动时，可以快速启动更多的服务容器来应对高并发。

------

##### **2. Zookeeper：分布式协调与锁管理**

在电商项目的秒杀活动中，可能会有多个用户同时请求购买限量商品。为了解决 **库存超卖** 问题，我们需要 **分布式锁** 来协调多个服务对库存的访问。Zookeeper 是一种流行的分布式协调框架，适用于这种场景。

**如何使用 Zookeeper**：

- **分布式锁**：秒杀活动开始时，多个微服务可能会尝试同时减少库存。为了避免超卖，系统需要确保只有一个请求能够成功减少库存，而其他请求需要等待或失败。通过 Zookeeper，我们可以为每个商品设置一个分布式锁，确保只有一个服务能够对库存进行操作。
- **主节点选举**：假设系统中的多个服务实例同时启动处理秒杀请求，Zookeeper 可以用来进行主节点选举，确保只有一个节点处理秒杀逻辑，以避免重复扣减库存。
- **服务发现**：如果系统有多个微服务实例，Zookeeper 可以帮助这些服务进行注册和发现，确保秒杀服务能够正确地找到其他服务（如支付服务、订单服务等）。

**Zookeeper 的优点**：

- 提供强一致性的保证，确保秒杀服务之间的协调和同步。
- 通过分布式锁机制，有效防止库存超卖。
- 在多节点环境中，可以实现高可用和容错。

------

##### **3. Kubernetes (K8s)：容器编排与自动化管理**

当秒杀活动启动时，系统需要处理大量并发的请求，这时就需要 **Kubernetes** 来提供容器编排和自动化管理的功能。Kubernetes 可以根据负载自动扩展或收缩服务实例，确保系统的高可用性和性能。

**如何使用 Kubernetes**：

- **自动扩展**：秒杀活动时，用户访问量激增，Kubernetes 可以根据实际流量自动扩展秒杀服务的容器实例。例如，秒杀开始时，秒杀服务可能需要扩展到数百个容器实例来应对高并发请求。
- **负载均衡**：Kubernetes 内建的负载均衡机制可以将请求均匀地分配到多个容器实例上，避免单个实例过载。
- **健康检查与自愈能力**：Kubernetes 可以定期检查容器的健康状况。如果某个容器因异常退出或失败，Kubernetes 会自动重新启动该容器，确保系统持续可用。
- **服务发现**：在微服务架构中，Kubernetes 负责提供服务发现机制，秒杀服务可以自动找到支付服务、库存服务和订单服务的容器实例，并进行通信。

**Kubernetes 的优点**：

- 提供自动化部署、扩展、负载均衡等功能，减少人工干预。
- 在秒杀活动期间，自动扩展容器，保证服务可用性，避免因流量激增导致的服务崩溃。
- 通过自动恢复机制，确保容器异常时自动重启，保持系统高可用。

------

##### **总结：Docker、Zookeeper 和 Kubernetes 在电商项目中的协同工作**

1. **Docker**：负责将电商项目中的每个微服务（如秒杀服务、支付服务、订单服务等）打包为独立的容器，确保在不同环境下的一致性。通过 Docker，我们可以快速启动、测试和部署微服务。
2. **Zookeeper**：作为分布式协调框架，Zookeeper 负责处理分布式锁、主节点选举和服务发现等功能。在秒杀活动中，Zookeeper 确保库存操作的同步，避免多个请求同时修改库存，防止超卖。
3. **Kubernetes**：作为容器编排平台，Kubernetes 负责自动管理和调度容器的部署。它根据流量需求自动扩展容器，提供负载均衡，确保电商平台在秒杀活动期间能够高效处理大量并发请求，并保持高可用性。






