CreateTime:2015-10-08 10:27:14.0

###抽象类
    public abstract class AbstractFileSystem {}
    位置：
    org.apache.hadoop.fs.AbstractFileSystem


_This class provides an interface for implementors of a Hadoop file system (analogous to the VFS of Unix). Applications do not access this class; instead they access files across all file systems using FileContext. Pathnames passed to AbstractFileSystem can be fully qualified URI that matches the "this" file system (ie same scheme and authority) or a Slash-relative name that is assumed to be relative to the root of the "this" file system ._
**提供了一个抽象的文件接口，不能被实例化，但是通过其获取文件系统的文件内容,通过“抽象的文件路径”可以构造一个合格的URI提供文件访问，
类似与linux的/    比如 hdfs://master:9000/**

 

###变量：
```
Statistics  statistics    //统计
```
###方法
```
 //判断名字是否可用
public boolean isValidName(String src) 

//访问文件并检查权限
public void access(Path path, FsAction mode)  
//public URI getUri,getFsStatus,get......
public void checkPath(Path path) // ??  

public Path getHomeDirectory()  // return Path  =>获取到一个用户(hadoop用户)的目录

//ACL方法：
setAcl  getAcl  modifyAcl ....
//Attr方法
setAttr  getAttr  modifyAttr
//其他方法:  略
//重要方法:
public static AbstractFileSystem createFileSystem(URI uri, Configuration conf)//里面构造了一个文件系统
//从一个URI按照Conf的规则获取一个文件系统   => 实现的是方法createFileSystem
public static AbstractFileSystem get(final URI uri, final Configuration conf)

//(实际动过都由之类重新实现了，但调用时可直接调用)
 public final FSDataOutputStream create(final Path f,
    final EnumSet<CreateFlag> createFlag, Options.CreateOpts... opts)
public final void rename(final Path src, final Path dst,       final Options.Rename... options)
//抽象方法：
public abstract FsServerDefaults getServerDefaults() throws IOException; 
public abstract boolean setReplication(final Path f, final short replication)
public abstract void mkdir(final Path dir, final FsPermission permission,    final boolean createParent)

public FSDataInputStream open(final Path f)
public abstract FSDataInputStream open(final Path f, int bufferSize)
public abstract FSDataOutputStream createInternal(Path f,EnumSet<CreateFlag> flag, FsPermission absolutePermission,int bufferSize, short replication, long blockSize, Progressable progress,ChecksumOpt checksumOpt, boolean createParent);


public abstract void setPermission,setOwner,set......

//文件完整性抽象
public abstract FileChecksum getFileChecksum(final Path f)
//获取文件状态
public abstract FileStatus getFileStatus(final Path f)


//补充   ACL是什么  =>  访问控制列表  =》也就是权限

```