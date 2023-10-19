# Auth 身份认证

HTTP/1.1 使用的认证方式如下所示。

- BASIC 认证(基本认证)
- DIGEST 认证(摘要认证)
- SSL 客户端认证
- FormBase 认证(基于表单认证)
- 此外，还有 Windows 统一认证(Keberos 认证、NTLM 认证)。

## 一、基本（basic）认证

BASIC 认证(基本认证)是从 HTTP/1.0 就定义的认证方式。即便是 现在仍有一部分的网站会使用这种认证方式。是 Web 服务器与通信 客户端之间进行的认证方式。

### 1. BASIC 认证的认证步骤

![image-20231019171151907](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231019171151907.png)

- 步骤 1: 当请求的资源需要 BASIC 认证时，服务器会随状态码 401 Authorization Required，返回带 WWW-Authenticate 首部字段的响应。 该字段内包含认证的方式(BASIC) 及 Request-URI 安全域字符串 (realm)。

- 步骤 2: 接收到状态码 401 的客户端为了通过 BASIC 认证，需要将 用户 ID 及密码发送给服务器。发送的字符串内容是由用户 ID 和密码 构成，两者中间以冒号(:)连接后，再经过 Base64 编码处理。

    假设用户 ID 为 guest，密码是 guest，连接起来就会形成 guest:guest 这 样的字符串。然后经过 Base64 编码，最后的结果即是 Z3Vlc3Q6Z3Vlc3Q=。把这串字符串写入首部字段 Authorization 后， 发送请求。



## 二、摘要（digest）认证

从 HTTP/1.1 起开始有 DIGEST 认证，DIGEST 使用质询/响应（challenge-response）方式。challenge-response 方式是指：一开始一方会发送认证方式给另一方，接着使用从另一方获得的质询码（challenge）计算生成响应码，最后响应码返回给对方的进行认证的方式。DIGEST 认证提供高于 BASIC 认证的安全等级，但是和 HTTPS 客户端相比还是比较弱。

![image-20231019154739170](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231019154739170.png)

因为发送给对方的只是响应摘要及由质询码产生的就算结果，所以比起 BASIC 认证，密码泄漏的可能性降低。

### 1. DIGEST 认证的认证步骤

![image-AQ9jpQLoU-transformed](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-AQ9jpQLoU-transformed.png)

- 步骤1：请求需要认证的资源时，服务器会随着状态码 401 Authorization Required，返回带 WWW-Authenticate 首部字段的响应。该字段内包含质询响应方式所需的临时质询码（随机数：nonce）。
    - 首部字段 WWW-Authenticate 内必须包含 realm 和 nonce 这两个字段的信息。客户端就是依靠向服务器回送这两个值进行认证的。
    - nonce 是一种每次随返回的 401 响应生成的任意随机字符串。该字符串通常推荐由 Base64 编码十六进制数的组成形式，但实际内容依赖服务器的具体实现。
- 步骤2：接收到 401状态码的客户端，返回的响应中包含 DIGEST 认证必须的首部字段 Authorization 信息
    - 首部字段 Authorization 内必须包含 username、realm、nonce、uri 和 response 的字段信息。其中 realm 和 nonce 就是之前从服务器接收到的响应中的字段。
    - username 是 realm 限定范围内可进行认证的用户名。
    - uri（digest-uri）即 Request-Uri 的值，但考虑到经代理转发以后 Request-Uri 的值可能进行改变，因此先复制一个副本保存在 uri 中。
    - **response 也可以叫做 Request-Digest** ，存放经过 MD5 运算后的密码字符串形成响应码。
- 步骤3：接收到包含首部字段 Authorization 请求的服务器，会确认认证信息的正确性。认证通过后则返回包含 Request-Uri 资源的响应。并且这时会在首部字段 Authorization-Info 中写入认证成功的相关信息。

```java
//响应头：WWW-Authenticate  或者 请求头：Authorization
Digest username="用户名", realm="角色描述", nonce="服务端随机数(Base64或十六进制编码)", uri="请求uri部分", response="摘要（客户端计算、到时服务端也会计算一份看是否一致）", algorithm="摘要算法", cnonce="客户端随机数(Base64或十六进制编码)", opaque="服务端给的信息原封不动换回去即可", nc="使用次数", qop="auth|auth-int"


//响应头WWW-Authenticate 必须提供
opaque、algorithm

//响应头WWW-Authenticate 提供
qop，则请求头Authorization将这个参数原封不动的还回去


//摘要公式  secret=密钥key  data=需要摘要的数据   两个参数都是根据规则由WWW-Authenticate里面的参数进行构建
//由于摘要算法无需密钥即可生成，故充分利用将secret也作为被摘要的部分数据
KD(secret, data)=摘要算法(concat(secret, ":", data))   
secret（也称A1）=摘要算法(摘要算法(<username>:<realm>:<password>):<nonce>:<cnonce>)
secret（也称A1-sess形式）=摘要算法(<username>:<realm>:<password>)
data=<nonce>:<nc>:<cnonce>:<qop>:摘要算法(A2)
A2=<request-method>:<uri>

```

参数解释：

| 原英文         |       参数（请求头携带的参数名）       |                             作用                             |
| -------------- | :------------------------------------: | :----------------------------------------------------------: |
|                | （响应头-认证失败）WWW-Authentication  | 用来定义使用何种方式（Basic、Digest、Bearer等）去进行认证以获取受保护的资源、以及各种其他参数的添加 |
| digest-uri     |                  uri                   |                        当前请求的 uri                        |
|                |                username                |                            用户名                            |
|                |                 realm                  | 表示服务器中受保护文档的安全域（比如公司财务信息域、公司员工信息域），用来指示需要那个域的用户名和密码 |
| message-qop    |                  qop                   | 保护质量，包含 **auth**（默认）和 **auth-int**（增加了报文完整性检测）以及 **Token** 三种策略，可以为空，但是不推荐为空 |
| nonce          |                 nonce                  | 服务端向客户端发送质询时附带的一个随机数，这个数会经常发生变化。客户端计算密码摘要时将其附加上去，使得多次生成同一用户的密码摘要各不相同，用来防止重放攻击 = **官方建议每次请求都不同** |
| userhash       |                userhash                |               （选填）是否支持用户名使用 hash                |
|                |                 opaque                 | （选填）服务端给的数据一般是 Base64 或者十六进制编码，客户端请求认证时原封不动返回给服务端 |
|                |    （请求头-开始认证）Authorization    |              认证类型以及用于携带客户端认证信息              |
| nonce-count    |                   nc                   | nonce计数器，是一个16进制的数值，表示同一nonce下客户端发送出请求的数量。例如，在响应的第一个请求中，客户端将发送“nc=00000001”。这个指示值的目的是让服务器保持这个计数器的一个副本，以便检测重复的请求 |
|                |                 cnonce                 | 客户端随机数，这是一个不透明的字符串值，由客户端提供，并且客户端和服务器都会使用，以避免用明文文本。这使得双方都可以查验对方的身份，并对消息的完整性提供一些保护 |
| request-digest |                response                | 这是由用户代理软件计算出的一个字符串，以证明用户知道口令 == **认证凭证（服务端校验）** == **公式：MD5(MD5(A1):::::MD5(A2))** |
|                | （响应头-认证成功） Authorization-Info |             用于返回一些与授权会话相关的附加信息             |
|                |               nextnonce                |      下一个服务端随机数，使客户端可以预先发送正确的摘要      |
|                |                rspauth                 | 响应摘要，用于客户端对服务端进行认证 == **认证凭证（客户端校验）** == **公式：MD5(MD5(A1):::::MD5(A2))** |
|                |                 stale                  | 当密码摘要使用的随机数过期时，服务器可以返回一个附带有新随机数的401响应，并指定stale=true，表示服务器在告知客户端用新的随机数来重试，而不再要求用户重新输入用户名和密码了 |
|                |               algorithm                | 摘要算法，由服务端（响应头）进行指定，**默认MD5**，支持MD5、MD5-sess、SHA-256、SHA-256-sess、SHA-512-256、SHA-512-256-sess、token值 |