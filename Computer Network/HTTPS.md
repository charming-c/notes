# HTTPS：确保 WEB 安全

## 一、HTTP 的缺点

HTTP 主要有如下不足，例举如下：

- 通信使用明文（不加密），内容可能会被窃听
- 不验证通信方的身份，因此有可能遭遇伪装
- 无法证明报文完整性，所以有可能已遭篡改

这些问题不仅在 HTTP 中出现，其他未加密的协议中也会存在类似的问题。

### 1. 通信使用明文可能会被窃听

由于 HTTP 本身不具备加密的功能，所以也无法做到对通信整体（使用 HTTP 协议通信的请求和相应内容）进行加密。即，HTTP 报文使用明文方式发送。

- TCP/IP 是可能被窃听的网络

- 加密处理防止被窃听

    目前正在研究的如何让防止窃听保护信息的几种对策中，最为普及的就是加密技术。

    - 通信的加密

        HTTP 协议中没有加密机制，但可以通过和 SSL（secure socket layer，安全套接层）或者 TLS（Transport Layer Security）的组合使用，加密 HTTP 的通信内容。

        用 SSL 建立安全通信线路以后，就可以在这条线路上进行 HTTP 通信了，称为 HTTPS。

    - 内容的加密

        还有一种将参与通信的内容本身加密的方式。由于 HTTP 协议中没有加密机制，那么就对 HTTP 协议传输的内通本身加密。即把 HTTP 报文里所含的内容进行加密处理。

        在这种情况下，客户端需要对 HTTP 报文进行加密处理后再发送请求。

        <img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231114162615505.png" alt="image-20231114162615505" style="zoom:50%;" />

### 2. 不验证通信方的身份就可能遭遇伪装

HTTP 协议中的请求和响应不会对通信方进行确认。也就是说存在“服务器是否就是发送请求中 URI 真正指定的主机，返回的响应是否真的返回到实际提出请求的客户端“等类似问题。

- 任何人都可以发起请求

    在 HTTP 协议通信时，由于不存在确认通信方的处理步骤，任何人都可以发起请求。服务器只要接收到请求都会响应，因此就会有很多隐患：

    - 无法确定请求发送至目标的 Web 服务器是否是返回响应的服务器。
    - 无法确定客户端是否是收到响应的客户端。
    - 无法确定正在通信的对方是否有访问权限。
    - 即使是无意义的请求服务器也会照单全收，无法阻止海量请求下的 Dos 攻击（Deny of Service，拒绝服务攻击）。

- 查明对手的证书

    虽然使用 HTTP 无法确定通信对方，但如果使用 SSL 则可以。SSL 不仅提供加密处理，而且还使用的了一种被称之为证书的手段，可用于确定方。

    <img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231114163751798.png" alt="image-20231114163751798" style="zoom:50%;" />

### 3. 无法证明报文完整性，可能已遭篡改

- 接收到的内容可能有误

    <img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231114163958963.png" alt="image-20231114163958963" style="zoom:50%;" />

    请求或响应在传输中，遭受攻击者拦截并篡改内容的攻击称为中间人攻击（Man-in-the-Middle attack，MITM）。

- 如何防止篡改

    虽然有使用 HTTP 协议确定报文完整性的方法，但事实上并不可靠、快捷。其中常用就是 MD5 和 SHA-1 等散列值校验的方法，以及用来确认文件的数字签名方法。

## 二、HTTP + 加密 + 认证 + 完整性保护 = HTTPS

把添加了加密及认证机制的 HTTP 称为 HTTPS。

### 1. HTTPS 是身披 SSL 外壳的 HTTP

HTTPS 不是新的应用层协议，只是 HTTP 通信接口部分用 SSL 和 TLS 协议代替而已。通常 HTTP 直接和 TCP 通信，当使用 SSL 时，则演变成先和 SSL 通信，再由 SSL 和 TCP 通信。

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231114165113215.png" alt="image-20231114165113215" style="zoom:50%;" />

SSL 是独立于 HTTP 的协议，所以不光是 HTTP 协议，其他运行在应用层的 SMTP 和 Telnet等协议均可配合 SSL 协议使用。可以说 SSL 是世界上应用最为广泛的网络安全技术。

### 2. 相互交换密钥的公开密钥加密技术（非对称加密）

- HTTPS 采用混合加密机制

    <img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231114170139770.png" alt="image-20231114170139770" style="zoom:50%;" />

### 3. 证明公开密钥正确性的证书

公开密钥加密开始有一些缺陷，无法证明公开密钥本身就是货正价实的公开密钥。比如正准备和某台服务器建立公开密钥加密方式下的通信时，如何证明收到的公开密钥就是原本预想的那台服务器发行的公开密钥，或许在公开密钥传输途中，真正的密钥已经被攻击者替换掉了。为了解决上述问题，可以使用有数字证书认证机构（CA，Certificate Authority）和其他机关颁发的公开密钥证书。

服务器会将这份由数字认证机构颁发的公钥证书发送给客户端，以进行非对称加密方式通信，公钥证书也被称为证书或者数字证书。接到证书的客户端可使用数字证书认证机构的公开密钥，对那张证书上的数字签名进行验证。

此处认证机关的公开密钥必须安全的转交给客户端，使用通信方式时，如何安全转交是一件很困难的事情，因此，多数浏览器开发商发布版本时，会事先在内部植入常用认证机关的公开密钥。

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231114171703528.png" alt="image-20231114171703528" style="zoom:50%;" />

- 可证明组织真实性的 EV SSL 证书

- 用以确认客户端的客户端证书（网银业务）

- 由自认证机构机构颁发的证书称为自签名证书

    如果使用 OpenSSL 这套开源程序，每个人都可以构建出一套属于自己的认证机构，从而给自己颁发服务器证书，但该服务器证书在互联网上无法当作证书使用。

### 4. HTTPS 的安全通信机制

以下是 HTTPS 的通信机制：

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231114172435797.png" alt="image-20231114172435797" style="zoom: 50%;" />

- 1. 客户端通过发送 Client Hello 报文开始 SSL 通信。报文中包含客户端支持的 SSL 的指定的版本、加密组件列表（所使用的加密算法以及密钥长度等）
- 2. 服务器可进行 SSL 通信时，会以 Server Hello 报文作为应答。和客户端一样，报文中包含 SSL 版本以及加密组件，服务器的加密组件内容是从客户端加密组件中筛选出来的。
- 3. 之后服务器发送 Certificate 报文。报文中包含公开密钥证书。
- 4. 最后服务器发送 Server Hello Done 报文通知客户端，最初阶段的 SSL 握手协商部分结束。
- 5. SSL 握手第一次结束以后，客户端以 Client Key Exchange 报文作为回应。报文中包含通信加密中使用的一种被称为 Pre-master secret 的随机密码串。该报文已用步骤 3 中的公开密钥进行加密。
- 6. 接着客户端继续发送 Change Ciper Spec 报文。该报文会提示服务器，在此报文之后的通信会采用 Pre-master secret 密钥加密。
- 7. 客户端发送 Finished 报文。该报文包含连接至今全部报文的整体校验值。这次握手协商是否能够成功，要以服务器能否正确解密该报文作为判定标准。
- 8. 服务器同样发送 Change Ciper Spec 报文。
- 9. 服务器同样发送 Finished 报文。
- 10. 服务器和客户端的 Finished 报文交换完毕之后，SSL 连接就算建立完成。当然，通信会受到 SSL 的保护，从此处开始进行应用层协议的通信，即发送 HTTP 请求。
- 11. 应用层协议通信，即发送 HTTP 响应。
- 12. 最后由客户端断开连接。断开连接时，发送 close_nofity 报文，上图做了省略，这步之后再发送 TCP FIN 报文来关闭与 TCP 的通信。

在以上步骤中，应用层发送数据时会附加一种叫做 MAC （Message Authentication Code）的报文摘要。MAC 能够查知报文是否遭到更改，从而保护报文完整性。

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231114175838819.png" alt="image-20231114175838819" style="zoom:50%;" />



- SSL 和 TLS 

    HTTPS 使用 SSL 和 TLS 两个协议。SSL 的前两个版本被发现有问题，主流版本是 SSL3.0。TLS 适宜 SSL3.0 为原型开发，当前主流是 TLS1.0，TLS 有时也称为 SSL。

- SSL 速度慢

    SSL 需要加密和解密步骤，所以通信慢，且过程中要计算需要消耗 CPU 和 内存等资源。