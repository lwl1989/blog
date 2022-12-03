CreateTime:2017-07-18 20:27:24.0

###1. 编写Dockerfile
    文件详细看git地址，这里只说明坑的地方
    RUN 后面只能跟一句命令
    假如要执行多条命令，和shell一样，需要用 && 串联
    任何命令失败都可能导致第二步失败
     
    导出可用端口是  EXPOSE  ，而不是  EXPORT
     
    [后续会继续补充，因为是在公司编写的，在家写blog]
###2. 编译
    docker build path name
    在编译这一步可能出现很多第一步的错误造成的失败，shell不精通的我，只能不停的踩坑了

    另外，docker中假如有要执行的.sh是通过 ADD 加载进去，一定要注意文件编码。
    否则在docker运行中会报painc  157-159行左右  no such file。
###3. 运行
     运行主要关注在后台运行
     几个常用option：
        1. -v 挂载目录  HostMachineDir : Docker MachinePort
        2. -d 后台运行  dockerName
        3. -p 导出端口   HostMachinePort : DockerMachinePort 
        

    几个坑 ：
        1.文件没有权限   -t  selinux问题  文件没有权限  --privileged=true
        2.自动退出 
        3.docker rm  =>  正在运行 + -f
        未完待续