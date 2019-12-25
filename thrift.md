# thrift
thrift是一个多语言支持的rpc框架，包含以下几个模块。
1. 类型系统。包括基本类型，容器，struct，异常，服务。
2. Transport。提供read，write，flush等接口，封装end2end的通信。
3. Protocol。封装了对语言对象的编解码功能。
4. Versioning。版本变更。提供兼容性测试接口。可以查询字段是否存在，接口是否实现。
5. RPC。提供rpc接口。包含通信协议，并发，异步操作等。



