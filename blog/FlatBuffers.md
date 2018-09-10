---
layout: post
title: "Flatbuffers使用及其源码浅析"
date: 2018-4-18 19:56
comments: true
tags: 
  - 技术
---
#### 为什么要用flatbuffers [官网](https://google.github.io/flatbuffers/)
|                                                        | FlatBuffers (binary)  | Protocol Buffers LITE | Rapid JSON            | FlatBuffers (JSON)     | pugixml               | Raw structs           |
|--------------------------------------------------------|-----------------------|-----------------------|-----------------------|------------------------| ----------------------| ----------------------|
| Decode + Traverse + Dealloc (1 million times, seconds) | 0.08                  | 302                   | 583                   | 105                    | 196                   | 0.02                  |
| Decode / Traverse / Dealloc (breakdown)                | 0 / 0.08 / 0          | 220 / 0.15 / 81       | 294 / 0.9 / 287       | 70 / 0.08 / 35         | 41 / 3.9 / 150        | 0 / 0.02 / 0          |
| Encode (1 million times, seconds)                      | 3.2                   | 185                   | 650                   | 169                    | 273                   | 0.15                  |
| Wire format size (normal / zlib, bytes)                | 344 / 220             | 228 / 174             | 1475 / 322            | 1029 / 298             | 1137 / 341            | 312 / 187             |
| Memory needed to store decoded wire (bytes / blocks)   | 0 / 0                 | 760 / 20              | 65689 / 4             | 328 / 1                | 34194 / 3             | 0 / 0                 |
| Transient memory allocated during decode (KB)          | 0                     | 1                     | 131                   | 4                      | 34                    | 0                     |
| Generated source code size (KB)                        | 4                     | 61                    | 0                     | 4                      | 0                     | 0                     |
| Field access in handwritten traversal code             | typed accessors       | typed accessors       | manual error checking | typed accessors        | manual error checking | typed but no safety   |
| Library source code (KB)                               | 15                    | some subset of 3800   | 87                    | 43                     | 327                   | 0                     |

#### 环境配置
- 下载flatbuffers源码
```
git clone https://github.com/google/flatbuffers.git
```
- cmake （源码中已包含cmake,也可自行下载）
- 编译出flatc

进入 flatbuffers/
```
1.cmake -G "Unix Makefiles"
2.make
```
#### java层源码支持

 
```
1.可以将flatbufffers/java 中的源码copy进项目

 2.将源码打包成jar(此处选用mvn打包)
```



* 安装mvn

1.去官网下载mvn源码[mvn下载](http://mirrors.shu.edu.cn/apache/maven/maven-3/3.5.3/binaries/apache-maven-3.5.3-bin.tar.gz)

2.在.bash_profile中加入如下配置
```
export M2_HOME="/Users/wangyang/apache-maven-3.5.3"

export PATH="$M2_HOME/bin:$PATH"
```
3.进入flatbuffers/ 
>  本质是将flatbuffers/java目录下源码打成jar，但是此目录下没有配置对应的pom.xml

本人尝试自己编写pom文件失败后，最后发现flatbuffers/目录下有此配置文件，坑，在此目录下运行以下命令，会生成target文件夹，在target/文件中找到flatbuffers-java-1.9.0-sources.jar

```
mvn install 
```



#### scheme文件编写

```
namespace com.rolan.MyGame;//在java中相当于包名
include "mydefinitions.fbs";//可引入其它的schemas
attribute "priority";//属性

enum Color : byte { Red = 1, Green, Blue }//枚举

union Any { Monster, Weapon, Pickup }//联合体，类比于enum和 struct

struct Vec3 {//类似于table，但它针对不再添加修改的情形，一些常量，比table内存占用小访问速度更快，
  x:float;
  y:float;
  z:float;
}

table Monster {//table类似于object，只能在末尾添加新的字段，可以rename
  pos:Vec3;
  mana:short = 150;
  hp:short = 100;
  name:string;
  friendly:bool = false (deprecated, priority: 1);//不能删除不再使用的字段，可用deprecated标志
  inventory:[ubyte];//[types]: ubyte bool short int float double long
  color:Color = Blue;
  test:Any;
}

root_type Monster;//根节点
```
#### types
 括号里是别名
- 8 bit: byte (int8), ubyte (uint8), bool
- 16 bit: short (int16), ushort (uint16)
- 32 bit: int (int32), uint (uint32), float (float32)
- 64 bit: long (int64), ulong (uint64), double (float64)
#### 编译scheme文件

根据schme文件编译生成java entity文件
```
flatc -j[--java] sample_schema.fbs
```
根据schme文件编译生成c++ 头文件

```
flatc -c[--cpp] sample_schema.fbs
```
根据.proto文件生成.fbs(将protobuffer格式转为flatbuffer)

```
flatc --proto Person.proto
```
根据.fbs文件和.bin文件生成json

```
flatc -t schema.fbs -- binary.bin
```

根据schme文件和 json数据文件生成所对应的flatbuffers格式的数据文件

```
flatc -b[--binary] sample_schema.fbs sample_json.json
```
了解其它相关命令[官方文档](http://google.github.io/flatbuffers/flatbuffers_guide_using_schema_compiler.html)

#### 序列化
以下面这个scheme为例
```
namespace com.rolan.entity;
table PeopleList {
    peoples : [People];
}

table People {
    id : string;
    index : long;
    guid : string;
    name : string;
    gender : string;
    company : string;
    email : string;
    friends : [Friend];
}

table Friend {
    id : long;
    name : string;
}

root_type PeopleList;
```
java层序列化到文件
```
FlatBufferBuilder builder = new FlatBufferBuilder(0);
        int peopleList[] = new int[100];
        for (int i = 0; i < peopleList.length; i++) {
            int nameOffect1 = builder.createString("李四");
            int nameOffect2 = builder.createString("张三");
            int nameOffect3 = builder.createString("王五");
            int friends[] = new int[3];
            friends[0] = Friend.createFriend(builder, 3243544, nameOffect1);
            friends[1] = Friend.createFriend(builder, 2341423, nameOffect2);
            friends[2] = Friend.createFriend(builder, 1232432, nameOffect3);
            int index = builder.createString("23412243");
            int guid = builder.createString("xnvnsfsd");
            int name = builder.createString("赵钱");
            int gender = builder.createString("男");
            int company = builder.createString("阿里妈妈");
            int email = builder.createString("ye@163.com");
            int friendsVector = People.createFriendsVector(builder, friends);
            People.startPeople(builder);
            People.addId(builder, index);
            People.addIndex(builder, 3001);//001会被识别成int
            People.addGuid(builder, guid);
            People.addName(builder, name);
            People.addGender(builder, gender);
            People.addCompany(builder, company);
            People.addEmail(builder, email);
            People.addFriends(builder, friendsVector);
            peopleList[i] = People.endPeople(builder);
        }
        int peoplesVector = PeopleList.createPeoplesVector(builder, peopleList);
        PeopleList.startPeopleList(builder);
        PeopleList.addPeoples(builder, peoplesVector);
        int endPeopleList = PeopleList.endPeopleList(builder);
        builder.finish(endPeopleList);

        ByteBuffer bb = builder.dataBuffer();
        File file = new File(getCacheDir() + "/test.bin");
        try {
            FileOutputStream fo = new FileOutputStream(file);
            fo.write(bb.array(), bb.position(), bb.remaining());
            fo.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
```
java反序列化

```
 File file = new File(getCacheDir() + "/test.bin");
        try {
            byte[] bytes = Utils.toByteArray(file);
            ByteBuffer wrap = ByteBuffer.wrap(bytes);
            PeopleList rootAsPeopleList = PeopleList.getRootAsPeopleList(wrap);
            People peoples = rootAsPeopleList.peoples(1);//取第1个
            String company = peoples.company();
            int peopleLength = peoples.friendsLength();
            Toast.makeText(this,peoples.company(),Toast.LENGTH_SHORT).show();
        } catch (IOException e) {
            e.printStackTrace();
        }
```


参考：[http://coolpers.github.io/](http://coolpers.github.io/)
[https://www.jianshu.com/p/03a2e8918f8a](https://www.jianshu.com/p/03a2e8918f8a)