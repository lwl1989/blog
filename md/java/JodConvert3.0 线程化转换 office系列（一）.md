CreateTime:2015-08-03 16:21:14.0

###&nbsp;&nbsp;&nbsp;&nbsp;在最近的项目中遇到问题，用户要上传office系列文档到项目中，然后我们要进行预览,那么问题来了，现在的word之类的是不支持预览的，2007以后的版本还行，03就是完全不行的。于是我去网上找资料，大谷歌是无敌的，大中国是要翻墙的......###
####搜到了一些相关资料####
   jodconvert
   openoffice
   swftools
####还有一些我用到的东西####
   RMI  其他的东西不方便透露   
###基本的思路是这样、我们由用户从servlet上传文件，接着开始对文件进行处理、上传，然后将上传的文件进行转换，最后上传至云端，接着就开始码代码的极端了###

###首先我们要准备好几个类:上传文件处理类、文档转换类（office to pdf）、pdf信息类、文档转换类2(pdf to swf)、数据库类等等###

首先进行文件上传

```
public String uploadFile() {
                //这里封装了 json
		if(userId.equals("")){
			return JsonReturn(false, null, "用户没有登录");
		}
		ArrayList<FileUpdateInfo> files=new ArrayList<FileUpdateInfo>();
		//JSONObject.
		if(list.size()<1){
			return JsonReturn(false, files, "没有文件上传");
		}
		try{
			for(FileItem item : list)
			{
				//获取表单的属性名字
				String name = item.getFieldName();
				
				//对传入的非 简单的字符串进行处理 ，比如说二进制的 图片，电影这些
				if(!item.isFormField())
				{
					//获取路径名
					String value = item.getName() ;
					//索引到最后一个反斜杠
					int start = value.lastIndexOf("\\");
					String filename = value.substring(start+1);
					if(filename.equals("")){ 
						FileUpdateInfo fileInfo=new FileUpdateInfo(filename, item.getSize(),"",false,"", "", name);
						fileInfo.setErrorMsg(2);
						files.add(fileInfo);
						continue;
					}
					String filePath=path+"/"+filename;
					String fileExt="";
					String fileSuffix=filename.substring(filename.lastIndexOf('.'));
					if(!fileSuffix.contains(fileSuffix)){
						FileUpdateInfo fileInfo=new FileUpdateInfo(filename, item.getSize(),fileExt,false,fileSuffix, filePath, name);
						files.add(fileInfo);
						fileInfo.setErrorMsg(4);
						continue;
					}
					//手动写的
					OutputStream out = new FileOutputStream(new File(path,filename));
					InputStream in = item.getInputStream() ;
					int length = 0 ;
					byte [] buf = new byte[1024] ;
					length = in.read(buf);
					fileExt=FileExtUtil.getExt(buf);
					out.write(buf, 0, length); 
	
					while( (length = in.read(buf) ) != -1)
					{			//在   buf 数组中 取出数据 写到 （输出流）磁盘上
						out.write(buf, 0, length);    
					}
					filePath=filePath.replaceAll("\\\\","/");
					FileUpdateInfo fileInfo=new FileUpdateInfo(filename, item.getSize(),fileExt,filename.substring(filename.lastIndexOf('.')), filePath, name);
					files.add(fileInfo);
					int fileId=insertDocument(fileInfo,userId);
					FileInfo info=new FileInfo(userId, filePath,fileId);
					//RMI写入本地转换进程
					InsertFileList(info);
					in.close();
					out.close();
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
		return JsonReturn(true, files, "结束");
	}
```
###上传完文件之后,我们将临时数据插入到数据库###
```
FileUpdateInfo fileInfo=new FileUpdateInfo(filename, item.getSize(),fileExt,filename.substring(filename.lastIndexOf('.')), filePath, name);
                    files.add(fileInfo);
                    int fileId=insertDocument(fileInfo,userId);

```
###上面我们返回了文件ID。接着我们使用了RMI进行了远程访问，当然，我这里是进行本地访问，将上传的文件加入到了队列里头###
```
	String rmiUrl="rmi://127.0.0.1:13355/FileAdd";
			ConvertList list=(ConvertList)Naming.lookup(rmiUrl);
			list.addQuery(fileInfo);
```

好了  这样我们的servlet端就已经处理完了。
下面我们来写文件转换的后台服务！