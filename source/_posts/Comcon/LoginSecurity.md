---
title: Something about authentication and authorization
date: 2019-06-05 23:48
tags: security
---

Wat the hack is [Oauth2](https://tools.ietf.org/html/rfc6749), [JWT](https://tools.ietf.org/html/rfc7519), [HTTPS](https://tools.ietf.org/html/rfc2818)??? Followings are my notes:
<!--more-->

## Oauth2
### 流程
主要流程如图：
![OauthProcess](/img/OauthProcess.png "Process")


### 四种验证方法

![Oauth4method](/img/Oauth4Methods.png "four ways of identification")


## JWT
主要流程(盗图)：![jwt](/img/JWT.png)
### 主要格式
    header.payload.signature

header结构：
```
{
    "typ": "JWT",
    "alg": "HS256"
}
```

payload 用于携带你希望向服务端传递的信息。你既可以往里添加官方字段（这里的“字段” (field) 也可以被称作“声明” claims）
例如iss(Issuer签发者)，aud（接受jwt的一方），jti（jwt唯一身份标识，主要作为一次性token）, sub(Subject面向的用户), exp(Expiration time)

也可以塞入自定义的字段，比如 userId:
```
{
    "userId": "yjqweqw0019-aq"//但一般不要敏感信息，因为会被看见
}
```

Signature结构

```
secret="mhh12121"//存在服务器上的密钥
data = base64( header ) + “.” + base64( payload )
signature = Hash( data, secret )//这里我们用HMACSHA256
```
假设我们的header 和paylaod如下
```
//header
{
    "typ": "JWT",
    "alg": "HS256"
}
//payload
{
    "userId": "yjqweqw0019-aq"
}
//服务器上的secret
secret="mhh12121"
```

可以通过这个网站![jwtio](https://jwt.io/)验证你的signature准确性
```
//我们的signature应该是这个
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJ5anF3ZXF3MDAxOS1hcSJ9.7jRvE_F8lriMHlTZPJulRPg_V66I6r-f7caMuI82u1o
```


## TLS (HTTPS=HTTP+TLS，TLS1.2=SSL3.3)

TLS (Transport layer security) mainly works in Application Layer. TLS is the 3rd edition of SSL(Secure Socket Layer) so that it can work on any application over TCP.

It provides securty over serverl aspects:

1. encryption(加密性)：指提交的数据不被获得
2. data Integerity(数据完整性)：指提交的的订单不被修改
3. Port Identification (端点鉴别，包括服务端和客户端)：指提交和接收的两边能互相确认身份



至于流程验证，为了方便了解，我们先简化SSL：

举例用Bob和Alice
#### 第一步：
##### 握手

一、 因为这是在TCP之上，所以先进行TCP链接,Bob发起链接，Alice接收，这里略过


二、 要验证Alice是真正的Alice：
    1. Bob会发送一个 SSL 验证报文，,下图是 **SSL hello**报文
    2. Alice 则用她的 **证书**进行回应，证书中包含了她的公钥
    3. Bob收到证书，因为 证书被 **CA** 证实过了，所以Bob会知道Alice的真实性
    4. 如果证书是真的，Bob会产生一个 **主密钥（MS）** 该密钥只用于这个SSL会话

三、 双方生成共同的密钥：
    1. Bob会用 Alice的公钥加密这个 **主密钥MS** 生成 **加密的主密钥(EMS)** 
    2. Bob发送 **EMS** 给Alice
    3. Alice收到 **EMS** 并用自己的 **私钥** 解开得到 **密钥（MS）**，这样双方都有一个共同的**密钥**


#### 第二步：
##### 密钥导出
现在双方有一个共同的密钥，已经可以用作通信，但前面说到，要保证3点加密性，数据完整性，端点鉴别
而且对于Alice和Bob每个人来说，这些都用不同的密钥才会更安全

为此，可以让Alice和Bob各生成4个密钥

1. E<sub>B</sub> 指Bob发送到Alice的数据的会话加密密钥
2. M<sub>B</sub> 指Bob发送到Alice的数据的会话MAC密钥
3. E<sub>A</sub> 指Alice发送到Bob的数据的会话加密密钥
4. M<sub>A</sub> 指Alice发送到Bob的数据的会话MAC密钥

每人通过MS生成4个密钥，其中两个用于加密数据，另外两个用于验证数据完整性


#### 第三步
##### 数据传输
但在数据传输中，经过TCP链接后，开始发送数据，一次发送了加密的数据，但另一个用于验证数据完整性的MAC去哪里了呢？
我们希望一次性就可以把加密数据和验证完整性的MAC都发送出去（即一次发送即可满足加密性和完整性）

为了解决这个问题，SSL将数据流分割成 [记录](#SSL-记录) ，对每一个记录附加一个MAC用于完整性检查，然后加密 **记录+MAC** ：
比如，从Bob发送开始， 产生这个MAC， Bob将数据连同密钥M<sub>B</sub>放入一个hash函数，再用自己CA会话加密密钥 **E<sub>B</sub>**,最后再传入TCP


然而，数据完整性仍然得不到保证，万一有中间人能抓取Bob发送的两个报文段，颠倒它们的顺序，调整TCP报文段的序号(seq)，将两个次序颠倒的报文段发送给Alice，
假设TCP报文段刚好封装一个记录（流式传输，所以不保证只含一个），Alice会这样做：
1. Alice端运行的TCP认为一切正常，传递这两个记录给SSL子层
2. SSL解密这两个记录
3. SSL会用每个记录中的MAC来验证完整性
4. SSL将解密的两条记录的字节流传给应用层，但实际上因为颠倒了报文，次序不正确。

解决如上问题，主要就要解决TCP的序号问题，所以就可以自己使用一个**序号计数器**：
Bob维护一个序号计数器，这个计数器不在记录中，而在MAC的计算中：即 
>>MAC=Hash(数据 + MAC密钥M<sub>B</sub> + 当前序号)

这样一来，以上颠倒了两个报文段的顺序，Alice解码发现顺序不对，就不会处理这两个报文



### SSL 记录
该记录如下图所示：
![SSL记录格式](/img/SSL.png)

主要由类型字段，版本字段，长度字段，数据字段和MAC构成，但其前三个字段是不加密的
类型字段：指出是握手报文还是有数据的报文，还被用于关闭SSL连接上（后面说到）

### 连接关闭时注意的问题

一般来讲，关闭连接就会由Client端，即Bob发起TCP FIN报文请求断开，但这个易遭到截断攻击（truncation attack)：
如果中间人过早地发送TCP FIN报文，Server端（Alice）会认为已经收到所有Bob的数据。

解决方案：SSL记录中的**类型字段**指明这个记录是否用于关闭连接。这里还要注意，虽然这个字段是明文，但接收方仍然可以用记录的MAC对它进行鉴别



***参考 《计算机网络自顶向下》 ***

### HTTPS（一般是443端口）

CA证书包含了：
1. 序号和过期时间
2. 姓名
3. 所有者公钥
4. 域名
5. 签发机构

流程如图![https](/img/HTTPS.png)


#### 问题：
1. 那么每次client端生成的key放在哪里呢？

改变环境变量SSLKEYLOGFILE，浏览器会从以下地址记录生成的对称密钥（linux）
```
export SSLKEYLOGFILE=~/tls/key.log
```
2. 被劫持咋办？
可以从图上的序号开始谈：
  1. 首先3处，可能 **有中间人拦截server端传去client端的请求吗，然后篡改证书？**
  如果中间人模拟一个自签名证书：
    浏览器会把这个自签名证书和系统证书匹配，匹配不上，会失败  
  如果中间人假冒颁发机构颁发证书：
    因为没有颁发机构的私钥，所以证书指纹不能对上，也会失败
  
  所以，唯一破解就是用户自己安装了一个未知证书，这样系统会认为中间人证书是信任的
  
  
  2. 接着6 处，即是被拦截，中间人没有server的私钥，无法解开

防止看不懂，还是新增一副图吧（盗图）：
![CA](/img/CA.jpeg)


3. 加密都用了啥？

而下面client**步骤4**则用的是**对称加密**传输的数据，再用**非对称加密** 加密 这个**经过对称加密的数据**（太绕）



## TLS1.3 

对比TLS1.2，TLS1.3在速度上有了很大的进步,
注意到之前TLS1.2在建立连接时用了4次RTT，即4次握手，但是TLS1.3缩减到了2次

### 做的修改
#### 速度加快

#### 加密算法删减
