# Cookie 和 Session 的关系和区别是什么？

- Cookie保存在客户端，Session保存在服务端
- 一般Session是依赖Cookie机制的，即如果禁用Cookie，Session可能也会失效
- 由于Cookie保存在客户端，所以可能被窃取、欺骗，而Session保存在服务端，敏感数据放在Session中可以大大提高安全性