---
layout: post
title: 关于在iOS中使用Google Protocol Buffer的使用体验
---


##关于在iOS中使用Google Protocol Buffer的使用体验

```
Google protocol Buffer  环境  
Google protocol Buffer 安装  
Google protocol Buffer 使用  
```
##Google protocol Buffer  环境  
   
Google protocol Buffer 安装前需要的环境有GCC，和autoreconf的，安装GCC的的直接使用Xcode安装即可。

安装autoreconf，使用mac下命令：
```
sudo brew install autoconf automake libtool
```

##Google protocol Buffer 安装 
Google protocol Buffer 的文档与地址是http://code.google.com/p/protobuf/

下载Protocol Buffers将下载解压后的文件存放至Applications目录下，进到ProtocolBuffers-2.2.0-Source目录看看会发现有个src目录。  
下载地址: http://code.google.com/p/metasyntactic/downloads/list

1. 在终端中切换到管理员身份，在终端下输入：su 然后输入密码，如果提示 su:Sorry，表明系统安全设置不允许，如果不想去更改，可以试着输入：sudo su,输入密码后如果看到sh-3.2#这种样式，表明成功。
	
	注：切换到管理员身份不是必须的，理论上所有命令都可以通过前面加sudo来执行。但我全部通过sudo来安装，在自己指定目录也能看到安装文件，也有protoc文件，但是提示命令无法识别，切换到文件所在目录也不行，没找到原因。
2. 用命令切换至ProtocolBuffers-2.2.0-Source目录下。
3.  使用：

```
   ./autogen.sh
```
4.在终端下输入
``` 
./configure (如果说没有权限，chmod +x configure)
```
如果不是管理员身份，需要输入：./configure - -prefix=$INSTALL_DIR 后面表示你要把protobuf安装的路径，需要是绝对路径。

6. 依次在终端下输入：

```
make
make check
make install
```
全部执行完后再输入protoc - - version检查是否安装成功。

**注： 使用make时会出现一些错误，这时需在message.m中的头文件中增加#include "istream"

##Google protocol Buffer 使用 
1. 生成Object-C代码
  创建一个Person.proto文件把该文件存放至想要的文件夹中，文件内容如下：
  
```
message Person {
required string name = 1;
required int32 id = 2;
optional string email = 3;

enum PhoneType {
MOBILE = 0;
HOME = 1;
WORK = 2;
}

message PhoneNumber {
required string number = 1;
optional PhoneType type = 2 [default = HOME];
}

repeated PhoneNumber phone = 4;
}
```

2. 在ProtocolBuffers-2.2.0-Source下创建这样一个子目录build/objc以便存放我们生成的classes

	现在执行命令:
```
protoc --proto_path=srcFolder --objc_out=desFolder
  srcFolder/Person.proto
```
成功后会在desFolder下生成Person.pd.h 和 Person.pb.m 两个Object-C文件

###添加编译环境
在Xcode中使用ProtocolBuffer
将步骤2中protobuf-obj/src/runtime/Classes目录导入到Xcode项目中，导入时，选中”Copy items into destination group‘s folder（if needed）“。 
 
导入位置选择项目根目录。导入完毕后，项目根目录下将会出现Classes目录，将该目录改名为ProtocolBuffers（注意最后的s）： mv Classes ProtocolBuffers
修改项目属性中”Build Settings–>Search Paths–>Header Search Paths”，将项目根目录“.”,"./ProtocolBuffer"添加到头文件搜索路径中去。  
这样ProtocolBuffer for Objective-C的工作环境就配置好了。

###工程中使用
1. 将步骤3中编译输出的Person.pb.h 和Person.pb.m添加到项目中
2. 将Person.pb.h 中的 #import <ProtocolBuffers/ProtocolBuffers.h> 改为#import”ProtocolBuffers/ProtocolBuffers.h”
3. 在需要使用的地方引入头文件：#import “Person.pb.h”

使用
```
    Person *person = [[[[[Person builder] setName:@"极致"]
                        setId:1]
                       setEmail:@"abc@163.com"] build];
    NSData *data = [person data];
     NSString *basePath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) objectAtIndex:0];

    NSString *path = [basePath stringByAppendingFormat:@"/person.data"];

    if ([data writeToFile:path atomically:YES]) {

        NSData *data = [NSData dataWithContentsOfFile:path];
        Person *person = [Person parseFromData:data];
        
        if (person) {
            NSLog(@"\n id %d \n name: %@ \n email: %@ \n",person.id, person.name, person.email);
        }
    }
```
输出打印的结果如下：
```
2014-08-16 11:46:15.502 protocol[18670:60b] 
 id 1 
 name: 极致 
 email: abc@163.com 
```

参考文章：
http://www.easymorse.com/index.php/archives/644  
