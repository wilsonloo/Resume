本目录记录了我的golang相关的开发套件

## 网络通信套件
  https://github.com/wilsonloo/galang_packages.git <br/>
  这是基于事件驱动的网络通信框架
  
## 协议搭建 protobuf
  https://github.com/google/protobuf/releases 下载 windowns 的 protoc.exe 这是主要的关键程序（linux 相同）
  https://github.com/golang/protobuf 获取针对 golang的 protobuf 插件 protoc-gen-go.exe 需要由golang 编译产生
  再使用 protoc-gen-go.exe 将 协议.proto 文件 生成 协议.go 文件
