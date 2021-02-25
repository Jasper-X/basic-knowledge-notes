# 简述 JWT 的原理和校验机制

## 一、JWT组成

一个 JWT 实际上是一个字符串，由Header、Payload、Signature组成。

- Header是一个base64编码后的JSON对象，包含了JWT的元数据。
- Payload也是一个base64编码后的JSON对象，存放实际传递的数据，在payload未加密的情况下不能放敏感数据。
- Signature是对Header和Payload的签名，其中签名算法是在Header中指定的，服务器中存放的secret必须严格保密。

## 二、JWT原理

用户向服务器提交登录凭证，服务器根据用户信息生成 JWT返回给用户。之后的一段时间内用户与服务端通信都要通过 JWT 来进行用户认证。

## 三、与session机制的区别

- 服务器需要维护session状态信息，JWT不需要。
- session可以随时过期，JWT发出去无法销毁，除非在服务器做额外处理。
- session中的数据存放在服务端，而JWT是保存在字符串内部。

## 四、如何验证用户身份

JWT Signature中的部分是 JWT 生成时根据服务器的密钥生成的，所以一般情况下只要密钥不泄露，其他人就无法篡改payload。那么在验证 Signature 之后就可以认为 JWT 是可信的。



## Reference

https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html

https://cnodejs.org/topic/5b0c4a7b8a4f51e140d942fc