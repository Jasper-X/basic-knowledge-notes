# 简述 RPC 的调用过程

1. Server A以接口或本地调用的形式调用Server B
2. Server A Stub处理调用payload(包括方法、参数等)，并进行组装和序列化，以网络消息的形式发送至Server B
3. Server B Stub收到消息后对消息进行反序列化和解析，根据消息内容调用本地服务
4. Server B本地服务返回执行结果给Server B Stub，Server B Stub将结果进行组装和序列化，返回给Server A
5. Server A Stub收到返回结果后对结果进行反序列化和解析，并将内容返回给调用者



## Reference

https://zhuanlan.zhihu.com/p/152586626