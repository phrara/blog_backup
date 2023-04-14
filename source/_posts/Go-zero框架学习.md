---
title: Go-zero框架学习
abbrlink: '0'
tags: 
    - 微服务
    - web
    - RPC
categories: 
    - 微服务
    - RPC
---
# go-zero-study
Just for studying go-zero framework

## 安装 go-zero & goctl Cli
```shell
# Run the following command under your project:
    go get -u github.com/zeromicro/go-zero

# for Go 1.16 and later
    go install github.com/zeromicro/go-zero/tools/goctl@latest
```


## API & RPC 语法
[官方API语法](https://go-zero.dev/cn/docs/design/grammar)

[Protobuf语法1](https://www.jianshu.com/p/4e7f861020d8)<br>
[Protobuf语法2](https://www.liwenzhou.com/posts/Go/Protobuf3-language-guide-zh/#autoid-0-1-0)

## 生成api业务代码 ， 进入"服务/cmd/api/desc"目录下，执行下面命令
```shell
goctl api go -api *.api -dir ../  --style=goZero
```
> 已存在的文件将不再生成，不会造成原文件被覆盖的问题，若需重新生成，须先删除原文件

### 目录结构
```shell
user_api:
│  user.go   # 启动文件
│
├─api
│      user.api    # API 源文件
│
├─etc
│      user-api.yaml    # 配置文件
│
└─internal
    ├─config
    │      config.go    # 配置管理
    │
    ├─handler
    │      loginHandler.go
    │      routes.go    # 路由
    │
    ├─logic
    │      loginLogic.go   # 业务逻辑
    │
    ├─svc
    │      serviceContext.go    # 相关依赖注入 
    │
    └─types
            types.go
```

## 生成rpc业务代码
> 需要安装下面3个插件
       protoc >= 3.13.0 ， 如果没安装请先安装 https://github.com/protocolbuffers/protobuf，下载解压到$GOPATH/bin下即可，前提是$GOPATH/bin已经加入$PATH中
+ protoc-gen-go ，如果没有安装请先安装 

```shell
    go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
```
+ protoc-gen-go-grpc ，如果没有安装请先安装

``` shell
    go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```
+ 如果有要使用grpc-gateway，也请安装如下两个插件 , 没有使用就忽略下面2个插件

```shell
    go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest  
     
    go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest
```
1. goctl >= 1.3 进入"服务/cmd/rpc/pb"目录下，执行下面命令
```shell
    goctl rpc protoc *.proto --go_out=../ --go-grpc_out=../  --zrpc_out=../ --style=goZero
   
# 去除proto中的json的omitempty
# mac: 
    sed -i "" 's/,omitempty//g' *.pb.go
# linux: 
    sed -i 's/,omitempty//g' *.pb.go
```
2. goctl < 1.3 进入"服务/cmd"目录下，执行下面命令
```shell
   goctl rpc proto -src rpc/pb/*.proto -dir ./rpc --style=goZero

# 去除proto中的json的omitempty
# mac: 
   sed -i "" 's/,omitempty//g'  ./rpc/pb/*.pb.go
# linux: 
   sed -i 's/,omitempty//g'  ./rpc/pb/*.pb.go
```
### 目录结构
```shell
user_rpc:
│  user.go              # 启动文件
│  
├─etc
│      user.yaml        # 配置文件
│
├─internal
│  ├─config              # 配置管理
│  │      config.go
│  │
│  ├─logic
│  │      getUserLogic.go         # 业务逻辑
│  │
│  ├─server
│  │      userServer.go           # 自动实现 gRPC-Server 接口的子类
│  │
│  └─svc
│          serviceContext.go        # 依赖注入
│
├─pb
│      user.pb.go               # proto -> go
│      user.proto               # Protobuf3 文件
│      user_grpc.pb.go          # go-gRPC 
│
└─user
        user.go                 # 客户端调用接口
```
### RPC调试
**Go gRPC 调试工具**： [gRPC UI](https://github.com/fullstorydev/grpcui)
+ 需要以开发者模式启动服务，在`/etc/xx.yaml`配置文件中添加：
    ```yaml
    Mode: dev # 启动模式：开发模式
    ```
+ 使用如下命令，以开启调试界面
    ```shell
    grpcui -plaintext localhost:8082
    ```

## API 调用 RPC
+ 修改yaml配置文件，配置etcd地址，与服务名
+ 修改config.go文件，在Config结构体中添加RPC配置信息字段
+ 在ServerContext结构体中添加goctl生成rpc服务时生成的相关rpcClient
+ 在业务逻辑中调用相关rpc接口 

## 创建kafka的topic
```shell
 kafka-topics.sh --create --zookeeper zookeeper:2181 --replication-factor 1 -partitions 1 --topic {topic}
```
1. 查看消费者组情况
```shell
 kafka-consumer-groups.sh --bootstrap-server kafka:9092 --describe --group {group}
```
2. 命令行消费
```shell
 ./kafka-console-consumer.sh  --bootstrap-server kafka:9092  --topic looklook-log   --from-beginning
```
3. 命令生产
```shell
 ./kafka-console-producer.sh --bootstrap-server kafka:9092 --topic second
```