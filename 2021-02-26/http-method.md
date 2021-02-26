# HTTP 的方法有哪些？

常用的方法：GET、POST、PUT、DELETE、OPTIONS、HEAD。

## GET

一般用于**数据读取，非幂等**操作。

敏感参数不应通过GET传递，否则可能会暴露在某些记录中。

## POST

一般用于**数据提交**操作，使用比较宽泛。

请求数据被包在请求体中，一般在修改资源或创建资源时使用。

## PUT

一般用于**数据提交**操作，是幂等的。

## DELETE

一般用于**数据删除操作，且是幂等的。

## HEADER

一般用于**服务器性能/状态探测**操作。与GET类似，但是服务器不会返回资源内容。

## OPTIONS

一般用于**资源访问权限嗅探**操作。如CORS时先发送OPTIONS方法的请求。





## Reference

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods

https://itbilu.com/other/relate/EkwKysXIl.html