---
layout: post
title: "Protocol Buffers使用及其源码浅析"
date: 2018-4-24 20:30
comments: true
tags: 
  - 技术
---
#### 环境配置
* 下载Protocol Buffers源码 [官网](https://developers.google.com/protocol-buffers/docs/javatutorial)

```
https://github.com/google/protobuf.git
```

* 编译源码

1.  进入protobuf/目录下
2.  ./autogen.sh 
3.  ./configure 
4.  make
5.  sudo make install

* 执行完上面步骤后即可全局执行protoc命令

```
protoc --version  查看当前版本
protoc --help     查看命令
```

* 编写 .proto文件

```
syntax = "proto2";
package protocal;
option java_package = "com.rolan.proto";
option java_outer_classname = "PersonEntity";
message People {
    repeated Person person = 1;//可重复的，即数组
    message Person {
        required int32 id = 1;
        required int64 index = 2;
        optional string guid=3;
        required string name = 4;
        optional string gender=5;
        optional string company=6;
        optional string email = 7;
        repeated Friend friend = 8;

        message Friend {
            required string id = 1;
            required string name = 4;
        }

    }

}



```


* 编译出java文件

```
protoc -I=. --java_out=. addressbook.proto
```

* java层源码支持

1.可自行根据源码中protocol/java 编译出jar包
2.build.gradle引入

```
implementation 'com.google.protobuf:protobuf-java:3.5.1'
```
* 序列化

```
 PersonEntity.People.Builder people = PersonEntity.People.newBuilder();
        for (int i = 0; i < sampleSize; i++) {
            PersonEntity.People.Person.Builder person = PersonEntity.People.Person.newBuilder();
            PersonEntity.People.Person.Friend friend1 = PersonEntity.People.Person.Friend.newBuilder().setId("1111").setName("李四").build();
            PersonEntity.People.Person.Friend friend2 = PersonEntity.People.Person.Friend.newBuilder().setId("2222").setName("张三").build();
            PersonEntity.People.Person.Friend friend3 = PersonEntity.People.Person.Friend.newBuilder().setId("3333").setName("王五").build();
            person.setId(i)
                    .setIndex(i)
                    .setGuid("guid" + i)
                    .setName("name:" + i)
                    .setGender(i % 2 == 0 ? "男" : "女")
                    .setCompany("加油宝：" + i)
                    .setEmail("sd@.com")
                    .addFriend(friend1)
                    .addFriend(friend2)
                    .addFriend(friend3)
                    .build();
            people.addPerson(person);
        }
        PersonEntity.People build = people.build();
```
* 反序列化

```
byte[] bytes = Utils.toByteArray(file);
            long startTime = System.currentTimeMillis();
            PersonEntity.People people = PersonEntity.People.parseFrom(bytes);
            PersonEntity.People.Person person = people.getPerson(1);
```

