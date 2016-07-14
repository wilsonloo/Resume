# 以下是个人的代码库描述

## https://github.com/wilsonloo/yy_idol（私有）
使用多进程架构的C++ 游戏服务端， 例如有常规的账号服务器、网关服务器、中心服务器、逻辑服务器、数据库服务器。该库有两个分支：

#### master 分支：是在windowns下开发、运行版本，参考广州要玩的《王者屠龙》服务器端设计；当前处理基础的多进程架构外，实现了角色属性、活动、军团、邮件、成就、战斗技能系统、好友系统、物品系统、任务系统、lua脚本嵌入、宠物（伙伴）、世界boss 等这些模块的基础代码；master分支的设计目标是架构及实现基本的游戏服务端基本框架，特点是充分使用了boost进行开发，尤其基于是boost::asio进行框架的运行（例如基于任务列表的多路复用的网络io、定时器回调），整体逻辑使用单线程；开发复杂度较大。

#### probuf-based分支：从master分支稳定后产生，主要用于 广州百田《YY-偶像大师》项目的具体应用，主要有以下几个新的特性：<br/>
1. 采用windows下vs进行开发编写coding，在ubuntu下编译运行（产生这一情况的原因是，随着更多第三方库的引入，windows下的 win_socket.h/windowns.h 的包含问题很是恶心，虽然自己有一套 wrapper.h <br/>可以处理这些文件的包含顺序问题，但是一些第三方库总是要求自己必须被首次include，我当时同时使用了boost::asio、redis、mysql等网络相关的第三方库，经常需要确保头文件包含顺序；后来厌烦了，直接变更开发方式）；<br/>
2. 引入redis（master的中心服务器主要有两大工作内容：【消息转发】和【全局消息缓存】，这就产生了单点问题；【消息转发】是无状态服务，挂掉重启即可恢复服务；但是【全局消息缓存】则需要面临数据的可恢复性，而redis本身就是基于内存的消息缓存，还能引入订阅-发布模式、数据落地操作）。redis 在《YY-偶像大师》中有一项实际实现：网关服务器的信息缓存；网关服务器fep是需要进行负载均衡的，随着玩家数量的改变，fep的数量也会动态变更，master分支的做法是由center进行统一管理（复杂度很高，需要维护与各种服务器间的处理）；每次新的fep开放后，都会向redis登记其基本的ip、port 和 玩家数量信息，断开则由center进行删除记录或者在redis上添加超时机制使自动删除；而那些依赖于fep的负载情况进行逻辑处理的服务，如登陆服务器、逻辑服务器，则在启动时从redis拉取这个fep列表、并使用redis的订阅-发布机制监听fep信息的变更。redis的这种机制简化了两个问题：消息缓存 和 基于发布-观察模式的消息处理。<br/>
3. 网络消息协议使用protobuf 和 二进制两种方式并存。<br/>
一、服务器内部之间采用二进制方式通信，原因是一般服务器代码维护的都是同一语言、同一帮人，协议多变性较少；<br/>
二、与客户端相关的通信，使用protobuf（一个比较大的体会是，《YY-偶像大师》客户端用的是flash，使用二进制方面，他们比较复杂；后来客户同事全体离职后，我用golang开发机器人代替客户端，golang的网络消息协议又得参照master的重写一遍，还有一次是使用UE4-VR在设计一款居家装修效果浏览的客户端，虽然是也是C++，但必须共享master的消息定义头文件）... 也就是说二进制方式比较固定、容易受制于某一框架；使用protobuf后，只需要各种客户端只需要依照消息原形文件 proto进行转换即可，无需进行人员间、编程语言之间沟通耦合。<br/>
三、不管使用protobuf 还是 二进制，都需要使用外围进行封装，至少得包含实际逻辑消息的长度；由于使用了混合机制，就进入了标记字段，其中包含以下标记位：<br/>
![网络消息协议格式](http://www.wilson-loo.com/wordpress/wp-content/uploads/2016/07/网络消息协议格式.jpg)
	

	消息有两部分组成：消息头 + 消息体<br/>
	<br/>
	消息头Header：<br/>
	其中消息头长度为 4 = sizeof(char) * 4 也就是一个长度为32位 的int<br/>
	高15位为控制字段: 	<br/>
			PACKET_FLAG_NONE			= 0x00000000,<br/>
			PACKET_FLAG_HAS_PROTOBUFF		= 0x40000000, // 实际逻辑包通过 proto buff 打包<br/>
			PACKET_FLAG_DYN_PROTOBUFF		= 0x20000000, // 动态protobuf，需要使用protobuf的反射机制处理<br/>
			PROTO_LANG_TYPE_CXX				= 0x00020000, // 传输proto 语言（CXX）<br/>
			PROTO_LANG_TYPE_GO				= 0x00040000, // 传输proto 语言（golang）<br/>
			PROTO_LANG_TYPE_JAVA			= 0x00080000, // 传输proto 语言（JAVA）<br/>
	低17位为实际消息的长度（可通过 0x1FFFF 掩码获取）<br/>
尤其是以后运维时，平台可能通过php进行平台活动的添加，那就使用了protobuf通信，并且消息类型使用字符串，这样就不需要和游戏开发商商量定制消息ID号，并且会设置 PACKET_FLAG_DYN_PROTOBUFF 标记，让游戏内部使用lua脚本进行动态消息的动态处理；<br/>
如果消息还需要打包的话，还会添加 PACKET_FLAG_ZIP_ENCODED 标记用表明需要进行压缩解压等等；PACKET_FLAG_HAS_PROTOBUFF 在fep中得到了广泛的引用，因为连接到fep的处理客户端外还有内部的其他服务器，这里就需要根据该标记进行区别处理咯。<br/>

## https://github.com/wilsonloo/chat_architeture
基于golang + protobuf + redis 聊天服务器<br/>
该项目的设计动机是，充分学习golagn 和 redis的好处。主要表现为<br/>
一、golang 是学习成本很低、开发效率很高的语言，其很好的一个特性是 channel 和 协程；channel能够很好地替代c++里的消息队列维护，编写很简单、效率高；而协程则可以很好的分派任务，而不必向c++那样需要特别关注线程的能效，考虑会不会太多线程等。而世界聊天则主要利用redis的发布-订阅机制，只用于世界聊天和跨服聊天，暂不支持小部分玩家间的聊天（因为每次订阅都需要家里新的tcp连接）；订阅的时候是聊天服务器向 聊天数据库redis 订阅本服服务的id，和世界服id，在通过客户端等级的 聊天UDP 端口进行聊天消息发放。<br/>
二、另外，golang的修改编译运行效率较高，方便编写了机器人，进行聊天压力测试。<br/>
三、添加了简单的开服工具<br/>
四、Player 分支，模拟机器人客户端用和 https://github.com/wilsonloo/yy_idol（私有）进行逻辑测试；<br/>
五、设计图：<br/>
![聊天服务器设计](http://www.wilson-loo.com/wordpress/wp-content/uploads/2016/07/聊天服务器设计.jpg)

<br/><br/>
## https://github.com/wilsonloo/utility（私有）
个人辅助工具：<br/>
	auto_link.h ------------- windowns下自动链接对应库服辅助类（参考自boost的库自动链接）<br/>
	bitmap_recorder.hpp ----- 位图（位大小可为一位或一个字节）<br/>
	console.hpp ------------- 终端输入处理，将输入的数值用std::vector 维护，由客户提供线程进行驱动<br/>
	data_dump.hpp ----------- 内存数据可视化显示<br/>
	fixed_size_object_pool.h- 固定大小的对象池<br/>
	message_block.hpp ------- 消息块<br/>
	message_queue.hpp ------- 基于message_block 的消息队列<br/>
	mini_dump.h ------------- windowns下的core dump 生成工具<br/>
	read_write_array.hpp ---- 读写队列（双缓冲池）<br/>
	singleton.hpp ----------- 单体模板<br/>
	task_base.hpp ----------- 任务队列<br/>
	uuid_generator.hpp ------ uuid 生成器<br/>
	varlen_struct.hpp ------- 边长结构体<br/>
	winsock_wrapper.h ------- windowns下的 winsock(2).h 与 windowns.h 的包含顺序辅助<br/>

## 以下只是封装，或者备份
### https://github.com/wilsonloo/evl_logger 基于log4cplus 的log<br/>
### https://github.com/wilsonloo/zlib123 压缩解压
### https://github.com/wilsonloo/mysql_helper 
### https://github.com/wilsonloo/luabind_with_vs_solution_and_libs 可编译的luabind

## https://github.com/wilsonloo/evl_net
基于boost::asio 的跨平台网络库
设计目标：提供tcp_server 和 tcp_client 的有状态封装类，两者都通过 tcp_session 维护一个会话，实际的IO操作有tcp_session 负责，tcp_server 和 tcp_client 主要负责事件的处理监听和触发。

## https://github.com/wilsonloo/asio(私有)
golang版本的网络库<br>
设计目标：在golang下能够快速进行网络通信，并维护相关的session状态<br>
设计框架：参考 https://github.com/wilsonloo/evl_net，也有主要tcp_server、tcp_connector、tcp_session<br>
命名特点：参考boost::asio<br>

## https://github.com/wilsonloo/asio_based_framework
基于 https://github.com/wilsonloo/asio 的通信框架<br>
设计目的：供 https://github.com/wilsonloo/chat_architeture 的Player分支进行网络通信<br>
特点：提供基本的消息类型（带前导标记、前导长度的消息结构），消息解包封包适配器<br>

## https://github.com/wilsonloo/asio_based_framework_usage
基于 https://github.com/wilsonloo/asio_based_framework 的 tcp服务端、客户端 使用例子

## https://github.com/wilsonloo/wgnet_framework
https://github.com/wilsonloo/asio 的原始版本

## https://github.com/wilsonloo/my_log
golang版本的log框架

## https://github.com/wilsonloo/serversample
基于Beego的OAuth2认证服务端范例
注：该项目取自网络，非本人原创
设计目标：为了以后网站应用能快速使用

## https://github.com/wilsonloo/go-app-scatter
基于golang的 服务器网络反应性能抓取工具
注：该项目取自网络，非本人原创

## https://github.com/wilsonloo/go-app-scatter-plugin
https://github.com/wilsonloo/go-app-scatter 的库版本

## https://github.com/wilsonloo/struct_nav（私有）
基于golang的居家装修效果浏览服务端<br/>
设计目的：可使用VR设备进行装修效果的浏览、可交互应用<br/>
目标效果：用户可以进行家具的替换、交互、上传模型、材质功能；可以添加好友、并邀请好友在同一房间内进行效果预览、共享家具模型、房间内聊天、有天气系统、地理坐标模拟装修成功后的各种效果浏览；<br/>
设计实现：CS模式开发，UE4负责客户端的可视化实现；服务端用golang进行玩家帐号信息的维护，并提供文件上传共享功能<br/>
其他：其实有两种CS实现方式，一是利用UE4自带的服务器、客户端交互机制，好处是简化服务端的开发，因为玩家间交互只限于房间内的一小群玩家，我假设是否可以参考掌机游戏的那种联机方式，机选一台服务器作为服务器，或者说每个客户端即是服务器；不过目前鉴于当前个人对UE4的属性程度来看，不知道能否实现；这种方式服务端就几乎只需要提供额外的文件上传共享机制即可，不足之处是使用UE4服务器不太熟悉，估计走的弯路较多；第二种方式是服务端完全手工写，房间架设在服务器上，要求客户之间的交互完全依赖于中心服务器，是否可以考虑在每个客户机器上架设一个手工写的服务器，客户之间通过其他途径通知各个客户端房间的存在，这种方式适合于局域网内、和面对面的客户（因为一般装修效果的浏览是在设计师工作地点）<br/>
当前服务端用golang实现，不过不管是用什么语言实现服务端，都必须考虑分布式系统的云存储问题、共享授权问题。golang的授权方式可采用 https://github.com/wilsonloo/serversample 加以修改，云存储估计就需要采用当前成功的产品了，貌似golang的云存储还不是特别成熟。<br/>

https://github.com/wilsonloo/struct_nav_client_helper
基于golang 的文件下载工具<br/>
设计目的：为了配合https://github.com/wilsonloo/struct_nav 服居家装修效果之模型共享下载而开发的一个工具<br/>
实现设想：客户在使用UE4客户端进行资源的上传、下载会新开一个进程进行事件的操作，操作结果通过进程间通信告知UE4展现给客户。<br/>

## https://github.com/wilsonloo/golang_smtp_client
基于golang的 smtp 客户端<br/>
设计目的：了解smtp的通信流程<br/>

## https://github.com/wilsonloo/whome
使用golang重构的 个人网站<br/>
设计目标：能够用golang快速开发网站<br/>

## https://github.com/wilsonloo/next_game（私有）
yy-idol《yy-偶像大师》服务端 https://github.com/wilsonloo/yy_idol 的原形<br/>
设计目标：相同于 yy_idol<br/>

https://github.com/wilsonloo/cocos2d-X-HotCity（私有）
使用 cocos2d 编写的rpg客户端<br/>
注：该项目取自网络，非本人原创<br/>
项目目标：配合服务端 next_game 进行游戏的登陆、注册、场景跳转、物品使用、技能效果 的测试工作<br/>

## https://github.com/wilsonloo/color-compile（私有）
gcc 下的编译颜色提示<br/>
设计目标：编译时warning，error，undefined reference to 等文本行会有不同的颜色标记，便于快速识别<br/>
