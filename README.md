# mpRPC
**简介**: 基于protobuf和zookeeper实现的分布式网络通信RPC框架, 网络模块基于muduo实现，并使用消息队列管理异步日志系统。

## RPC框架的设计
RPC的全称是**远程过程调用**（Remote Procedure Call）。

### 单机聊天服务器的缺陷：
- **(1)**　受限于服务器硬件资源，单机聊天服务器所能承受的用户**并发量有限**；
- **(2)**　修改任意的代码，会导致整个项目**重新编译和重新部署**，耗费时间；
- **(3)**　在系统中由于需求不同，模块是**CPU密集型或IO密集型**，对硬件的**资源需求**也不同。

### 集群聊天服务器的缺陷：
- **(1)** 每一台服务器都是一个独立的聊天系统，只是水平扩展；
- **(2)** 修改任意的代码，需要重新编译和重新部署，并且需要在每台服务器上部署，更加耗费时间。

### 分布式通信RPC架构下的聊天服务器：
- **(1)** 所有服务器共同组成一个聊天系统，给客户端提供服务；
- **(2)** 根据用户需求不同，各服务器运行不同的模块，并且每个服务器都有一个备用机防止宕机，作为冗余；
- **(3)** 各服务器之间可进行通信，会利用分布式锁，服务器在RPC上注册服务，客户端从RPC查询该服务位于哪台服务器；

### 分布式通信RPC架构流程
![git-command.jpg](https://camo.githubusercontent.com/cbf64d67d1f3215d77429739fdbc7a5eaae59affad2555aadce99bfc582c5fbc/68747470733a2f2f696d672d626c6f672e6373646e696d672e636e2f32303231303431383232343833323539372e706e673f782d6f73732d70726f636573733d696d6167652f77617465726d61726b2c747970655f5a6d46755a33706f5a57356e6147567064476b2c736861646f775f31302c746578745f6148523063484d364c7939696247396e4c6d4e7a5a473475626d56304c334e6f5a5735746157356e6548566c5356513d2c73697a655f31362c636f6c6f725f4646464646462c745f3730)

## protobuf
protobuf 主要是作为整个框架的**传输协议**。看一下整个框架对于传输信息的格式定义：
``` proto
message RpcHeader
{
    bytes service_name = 1; 	//类名
    bytes method_name = 2;		//方法名
    uint32 args_size = 3;		//参数大小
}

service UserServiceRpc
{
    rpc Login(LoginRequest) returns(LoginResponse);
    rpc Register(RegisterRequest) returns(RegisterResponse);
}
```
它定义了要**调用方法是属于哪个类的哪个方法以及这个方法所需要的的参数大小**。

### protobuf安装步骤：
https://github.com/google/protobuf
- 1、解压压缩包:unzip protobuf-master.zip
- 2、进入解压后的文件夹:cd protobuf-master
- 3、安装所需工具:sudo apt-get install autoconf automake libtool curl make g++ unzip
- 4、自动生成configure配置文件:./autogen.sh
- 5、配置环境:./configure
- 6、编译源代码(时间比较长):make7、安装:sudo make install
- 7、刷新动态库:sudo ldconfig

## zookeeper
Zookeeper是在分布式环境中应用非常广泛,它的优秀功能很多,比如分布式环境中全局命名服务,服务注册中心,全局分布式锁等等。

### 分布式系统问题
服务的动态注册和发现，为了支持高并发，OrderService被部署了4份，每个客户端都保存了一份服务提供者的列表，但这个列表是静态的（在配置文件中写死的），如果服务的提供者发生了变化，例如有些机器down了，或者又新增了OrderService的实例，客户端根本不知道，想要得到最新的服务提供者的URL列表，必须手工更新配置文件，很不方便。

**解决办法**:
- 解除耦合，增加一个中间层 -- 注册中心它保存了能提供的服务的名称，以及URL。
- 首先这些服务会在注册中心进行注册，当客户端来查询的时候，只需要给出名称，注册中心就会给出一个URL。所有的客户端在访问服务前，都需要向这个注册中心进行询问，以获得最新的地址。

![git-command.jpg](https://img2018.cnblogs.com/blog/1672215/201906/1672215-20190616153201378-194806403.png)


- 注册中心可以是**树形结构，每个服务下面有若干节点**，每个节点表示服务的实例。

![git-command.jpg](https://img2018.cnblogs.com/blog/1672215/201906/1672215-20190616153210274-2041002168.png)


- 注册中心和各个服务实例直接建立Session，要求实例们定期**发送心跳**，一旦特定时间收不到心跳，则认为实例挂了，删除该实例.

![git-command.jpg](https://img2018.cnblogs.com/blog/1672215/201906/1672215-20190616153220003-32958374.png)

## RPC框架服务提供方
``` c++
    RpcApplication::init(argc, argv);

    //框架服务提供provider
    RpcProvider provide;
    provide.notify_service(new UserService());
    provide.run();
```
- **(1)** 首先 RPC 框架肯定是部署到一台服务器上的，所以我们需要对这个服务器的 ip 和 port 进行初始化;
- **(2)** 然后创建一个 porvider（也就是server）对象，将当前 UserService 这个对象传递给他，也就是其实这个 RPC 框架和我们执行具体业务的节点是在同一个服务器上的。RPC框架负责解析其他服务器传递过来的请求，然后将这些参数传递给本地的方法。并将返回值返回给其他服务器;
- **(3)** 让这个 provider 去 run 起来。

## RPC框架服务请求方
``` c++
//初始化ip地址和端口号
    RpcApplication::init(argc, argv);

    //演示调用远程发布的rpc方法login
    ik::UserServiceRpc_Stub stub(new RpcChannel());

    //请求参数和响应
    ik::LoginRequest request;
    request.set_name("zhang san");
    request.set_password("123456");

    ik::LoginResponse response;
    RpcControl control;
    //发起rpc调用，等待响应返回
    stub.Login(&control, &request, &response, nullptr);
```
- **(1)** 初始化 RPC 远程调用要连接的服务器;
- **(2)** 定义一个 UserSeervice 的 stub 桩类，由这个装类去调用Login方法.



