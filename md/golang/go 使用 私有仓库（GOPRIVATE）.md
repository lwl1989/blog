CreateTime:2020-10-27 12:05:18.0

# 遇到的问题
直接使用go get ...添加私有仓库依赖时，会出现以下错误：

```
get "gitlab.com/xxx": found meta tag get.metaImport{Prefix:"gitlab.com/xxx", VCS:"git", RepoRoot:"https://gitlab.com/xxx.git"} at //gitlab.com/xxx?go-get=1
go get gitlab.com/xxx: git ls-remote -q https://gitlab.com/xxx.git in /Users/sulin/go/pkg/mod/cache/vcs/91fae55e78195f3139c4f56af15f9b47ba7aa6ca0fa761efbd5b6e2b16d5159d: exit status 128:
    fatal: could not read Username for 'https://gitlab.com': terminal prompts disabled
Confirm the import path was entered correctly.
If this is a private repository, see https://golang.org/doc/faq#git_https for additional information.
```
从错误信息可以明显地看出来，我们使用私有仓库时通常会配置ssh-pubkey进行鉴权，但是go get使用https而非ssh的方式来下载依赖，从而导致鉴权失败。

## proxy

如果配置了GOPROXY代理，错误信息则是如下样式：

	go get gitlab.com/xxx: module gitlab.com/xxx: reading https://goproxy.io/gitlab.com/xxx/@v/list: 404 Not Found

从错误信息可以看出，go get通过代理服务拉取私有仓库，而代理服务当然不可能访问到私有仓库，从而出现了404错误。

# 以前的做法（ version < 1.13）

在1.11和1.12版本中，比较主流的解决方案是配置git强制采用ssh。

	git config --global url."git@gitlab.com:xxx/zz.git".insteadof "https://gitlab.com/xxx/zz.git"

但是它与GOPROXY存在冲突，也就是说，在使用代理时，这个解决方案也是不生效的。

# 1.13以后的版本解决方案

## 1.13的新问题

在go1.13之后，go mod 已经开始成熟了（1.14已经开始推荐应用于生产环境）

在我们不改变方案的时候，ssh的解决办法在1.13版本之后已经不可用了。

```
get "gitlab.com/xxx/zz": found meta tag get.metaImport{Prefix:"gitlab.com/xxx/zz", VCS:"git", RepoRoot:"https://gitlab.com/xxx/zz.git"} at //gitlab.com/xxx/zz?go-get=1 verifying gitlab.com/xxx/zz@v0.0.1: gitlab.com/xxx/zz@v0.0.1: reading https://sum.golang.org/lookup/gitlab.com/xxx/zz@v0.0.1: 410 Gone
```

这个错误是因为新版本go mod会对依赖包进行checksum校验。

而私有仓库对sum.golang.org(代理的话，此域名也会更改)是不可见的，它当然没有办法成功执行checksum。

## 解决方案

1.13 新加入一个go的环境变量。 

	GOPRIVATE

可以使用：

	export GOPRIVATE=gitlab.com/xxx 
	or
	go env -w GOPRIVATE=gitlab.com/xxx

这里指定域名为私有仓库，单个go get遇到此域名下的所有依赖时，会直接通过git进行。

# 备注

如果是通过ssh公钥访问私有仓库，那么需要配置git拉取私有仓库时使用ssh。

如果本地git是带有端口号的，那么也需要重新代理过去（go的包名内，不能包含:）。

```
[url "git@github.com:"]
    insteadOf = https://github.com/
[url "git@gitlab.com:"]
    insteadOf = https://gitlab.com:2224/
```