---
title: IPFS Windows
tags:
  - Decentralization
  - 区块链
  - P2P
categories:
  - 大创
  - 区块链
abbrlink: 3e9a9d51
date: 2022-10-09 22:17:50
---



# IPFS Windows

>   go-IPFS 项目在v0.14及其以后版本改名为Kubo

[github](https://github.com/ipfs/kubo.git)



## Install

[release](https://github.com/ipfs/kubo/releases)

+   下载安装包解压得kubo目录

![](IPFS-Windows/zip.png)

+   将kubo目录添加至环境变量，以便使用 `ipfs` 命令



## Config

```shell
$ ipfs init
```

此命令将会在你的用户文件夹根目录下(**C:/Users/abc/**)生成节点目录名.ipfs，接下来打开**config**，里面的是ipfs的一些基础配置数据，可以根据自己的需求修改里面的配置



## Run

```shell
$ ipfs daemon
```

访问 http://localhost:5001/webui 即可界面化操作

或者采用 go 语言API操作

```go
package core

import (
	"bytes"
	"io"

	shell "github.com/ipfs/go-ipfs-api"
)

type FileList shell.LsLink

type API struct {
	sh *shell.Shell
}

func NewAPI() *API {
	sh := shell.NewShell("localhost:5001")
	return &API{
		sh: sh,
	}
}

func (a *API) Upload(data []byte) (string, error) {
	hash, err := a.sh.Add(bytes.NewBuffer(data))
	if err != nil {
		return "", err
	}
	return hash, nil
}

func (a *API) Download(hash string) ([]byte, error) {
	rc, err := a.sh.Cat(hash)
	if err != nil {
		return nil, err
	}
	defer rc.Close()
	b, err2 := io.ReadAll(rc)
	if err2 != nil {
		return nil, err2
	}
	return b, nil

}

func (a *API) List(hash string) ([]*FileList, error) {
	ll, err := a.sh.List(hash)
	if err != nil {
		return nil, err
	}
	res := make([]*FileList, 0, 10)
	for _, v := range ll {
		l := FileList(*v)
		res = append(res, &l)
	}
	return res, nil
}

```

