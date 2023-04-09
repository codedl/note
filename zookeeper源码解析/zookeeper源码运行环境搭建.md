---
title: zookeeper源码运行环境搭建
tags:
notebook: zookeeper源码解析
---
工欲善其事必先利其器，开始zookeeper的学习之旅前得把环境搭建好，接下来就要开始搭建zookeeper的运行环境了。
1. 到官网下载源码http://archive.apache.org/dist/zookeeper/，官网上有编译好的jar包，从GitHub上下载的源码需要自己编译，可能会因为网络环境导致编译失败。
2. 运行VerGen.java创建zk版本号文件Info.java，或者手动创建，将Info.java放到zookeeper-server\src\main\java\org\apache\zookeeper\version目录下
```java
package org.apache.zookeeper.version;

public interface Info {
    int MAJOR=3;
    int MINOR=4;
    int MICRO=14;
    String QUALIFIER=null;
    int REVISION=-1; //TODO: remove as related to SVN VCS
    String REVISION_HASH="9ce96740c2e1aea544fe94bb3abfbb620f268a2e";
    String BUILD_DATE="11/16/2022 14:09 GMT";
}
```
3. 编译整个项目，然后再编译zookeeper-jute模块，zk使用了protoc，需要在idea中安装Protobuf，注意得将idea内置的gRPC和Protocol Buffers禁用掉
4. 在conf目录下将zoo_sample.cfg文件复制为zoo.cfg，这是zk的配置文件，再将zookeeper-server下的resource标记为Resource Root目录，将conf下的log4j.properties拷贝过去。
5. 启动zk的主类是QuorumPeerMain，在org.apache.zookeeper.server.quorum类路径下，启动时设置zoo.cfg为main的启动参数，即可启动。补充下，可能会在代码中找不到某些类，因为这是protoc生成，需要手动拷贝下，从编译后得到的zookeeper-jute\target\classes\org\apache\zookeeper\data目录下拷贝就行了。