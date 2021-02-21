# 简述 HTTPS 的加密与认证过程

## TLS1.2

### ECDHE的密钥交换方式

![KlHXKe](http://image.skyjerry.com/uPic/KlHXKe.jpg)

1. 客户端发送"Client Hello"，包含信息：客户端版本号、支持的密码套件、客户端随机数
2. 服务器收到“Client Hello”后，会返回一个“Server Hello”消息。把版本号对一下，也给出一个服务端随机数，然后从客户端的列表里选一个作为本次通信使用的密码套件。服务器为了证明自己的身份，就把证书也发给了客户端。因为服务器选择了 ECDHE 算法，所以它会在证书后发送“Server Key Exchange”消息，里面是椭圆曲线的公钥（Server Params），用来实现密钥交换算法，再加上自己的私钥签名认证，Server Hello Done
3. 客户端验证证书，确认服务端身份，按照密码套件要求，也生成一个椭圆曲线的公钥（Client Params），用“Client Key Exchange”消息发给服务器，然后使用Client Params、Server Params生成预主密钥，用客户端随机数、服务端随机数、预主密钥生成主密钥，根据主密钥生成会话密钥进行通信。向服务端发送"Change Cipher Spec"，再发送"Finished"，对之前所有握手消息做摘要，加密一下，让服务端验证。紧接着发送第一个数据包（经过了前两步已经过去1-RTT，此时可以开启**TLS False Start**，客户端可以不用收到服务端的ACK，而是立即将数据包发送到服务端，将TLS的握手时间降低为1-RTT）
4. 服务端向客户端发送"Change Cipher Spec"，再发送"Finished"，对之前所有握手消息做摘要，再加密一下，让客户端验证

### RSA的密钥交换方式（已经不是主流）

![CudZi4](http://image.skyjerry.com/uPic/CudZi4.jpg)

大体流程与ECDHE有两点不同：

第一个，使用 ECDHE 实现密钥交换，而不是 RSA，所以会在服务器端发出“Server Key Exchange”消息。

第二个，因为使用了 ECDHE，客户端可以不用等到服务器发回“Finished”确认握手完毕，立即就发出 HTTP 报文，省去了一个消息往返的时间浪费。这个叫“**TLS False Start**”，意思就是“抢跑”，和“TCP Fast Open”有点像，都是不等连接完全建立就提前发应用数据，提高传输的效率。

## TLS1.3

![GujYPe](http://image.skyjerry.com/uPic/GujYPe.jpg)

#### 强化安全：

- 伪随机数函数由 PRF 升级为 HKDF（HMAC-based Extract-and-Expand Key Derivation Function）；
- 明确禁止在记录协议里使用压缩；
- 废除了 RC4、DES 对称加密算法；
- 废除了 ECB、CBC 等传统分组模式；
- 废除了 MD5、SHA1、SHA-224 摘要算法；
- 废除了 RSA、DH 密钥交换算法和许多命名曲线。

#### 提升性能：

握手时间：1-RTT。

删除Key Exchange消息，客户端在“Client Hello”消息里直接用“supported_groups”带上支持的曲线，比如 P-256、x25519，用“key_share”带上曲线对应的客户端公钥参数，用“signature_algorithms”带上签名算法。

服务器收到后在这些扩展里选定一个曲线和参数，再用“key_share”扩展返回服务器这边的公钥参数，就实现了双方的密钥交换，后面的流程就和 TLS1.2 基本一样了。

握手时间：0-RTT。PSK实现。





## HTTPS如何保证数据完整性

对信息签名。保证机密性的同时，数据不被篡改。

签名过程：先通过摘要算法生成原文的摘要(一个字符串)，后通过公钥/私钥对摘要进行加密，生成的密文就是签名

验签过程：将加密的摘要通过公钥/私钥解密出摘要，然后，通过摘要算法生成原文的摘要，比对解密出的摘要和自己算出的摘要是否一致。



## HTTPS如何保证数据机密性

### 非对称加密

用公钥加密的数据只能用私钥解密，反过来用私钥加密的数据也只能用公钥解密。

DH、DSA、RSA、ECC。

DH和DSA都不安全，所以现在不常用。

RSA是最常用的，基于**整数分解**的数学难题，密钥长度2048位比较安全。

ECC是非对称加密的后起之秀，基于**椭圆曲线的离散对数**的数学难题，使用特定的椭圆曲线方程和基点生成公钥和私钥，子算法ECDHE用于密钥交换，ECDSA用于数字签名。虽然定义公钥和私钥，但不能直接实现密钥交换和身份认证，需要搭配DH、DSA等算法，形成专门的ECDHE、ECDSA。RSA比较特殊，本身支持密钥交换和身份认证。

### 对称加密

用同一个密钥进行加密和解密。比如AES、DES、Chacha20。目前DES已经不安全，被废弃。

## HTTPS如何保证双方身份不是假的

权威机构用自己的私钥对网站公钥签名，保证其完整性，并形成信任链。用户的客户端保存了几大权威机构的根证书，通过证书链层层验证，可以保证证书身份的真实性。