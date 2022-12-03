CreateTime:2017-11-30 14:11:47.0

## 安装


mac 和 win
 都是下载安装包直接安装
 
 
linux可选二进制包 和 源码编译安装

## 环境配置

GOROOT即安装目录

GOPATH可以有多个  [src.bin,pkg]

GOBIN可执行文件生成目录 不配置的话是第一个GOPATH下面的bin

## 基本命令

go run xxx.go  //必须包含package main包和 func main方法

go build xxx.go  //将XX文件编译到pkg目录下 如果是main包  会产生执行文件到  GOBIN目录下

go get 包名  获取包

go install //编译当前项目  将bin和pkg文件分别生成到对应位置