---
layout: post
title: Road_to_https 既有系统的https的改造
key: 20161209
tags: 架构 互联网 基础系统
---
### 研究目标    
-  整理 https 相关的基本知识；  
-  梳理现存网站全面转向 https 的相关行动事项；

<!--more-->   
 
### 主要内容
  
-  相关名词和基本知识；  
-  简明的行动项清单；    
-  重点技术项的细化阐述，包括证书，性能改善等；    
-  一些行动建议；    
  
## 引子   
如果你对这个课题还没有什么**感性认识**，为了避免一开始就掉入过于枯燥、抽象的技术名词的泥沼，建议先试试以下两个事情：  

- 使用chrome 或 Firefox 桌面（非手机端）浏览器，打开下面几个网站：  
第一组：<http://github.com>  ，<http://wallstreetcn.com> 或 <http://qiniu.com> ;   
第二组：<http://data.stats.gov.cn/> 或 <http://www.cbrc.gov.cn/>    
可以从地址栏上看到明显的返回结果的差异。  
![github地址栏](http://p1wq6a0o4.bkt.clouddn.com/20161209github.png-m640 "github地址栏")  
  
  
- 尝试访问 [第三方SSL 测试平台](https://www.ssllabs.com/ssltest/index.html)   
检测前面域名，也可以看到不同的测试结果返回。  
![sslLABS测试wallstreetcn.com结果](http://p1wq6a0o4.bkt.clouddn.com/20161209wallstreetcn.png-m640 "sslLABS测试wallstreetcn.com结果") 
  

**建议**  

>如果现在对整个事情的全貌有兴趣，你可以现在先阅读[第二部分 开始行动](#section-7)  


## 一 基础知识   

###  1.1 部分名词
   
- CSP:  
  (Content Security Policy) ,http 响应头。用来告诉浏览器，在当前页面采用何种内容安全策略。 可减轻内容注入的风险，降低应用执行的权限，提升安全的 “ 厚度 ” 。但不是防御内容注入的首选。    
    
  **参考**： 

>[Content Security Policy Level 3 的W3 官方文档][^CSPl3]      
  

>*sample from w3.org：*   
  Given a page with the following Content Security Policy:  

```
  
  Content-Security-Policy: child-src https://example.com/  
   
```  
  >Fetches for the following code will all return network errors, as the URLs provided do not match child-src's source list:   

```   
  <iframe src="https://not-example.com"></iframe>
  <script>
   var blockedWorker = new Worker("data:application/javascript,...");
  </script>
 
```



- Ocsp:  
  在线证书状态检查协议 (RFC6960 ) ，用来向 CA 站点查询证书状态，比如是否撤销。通常情况下，浏览器使用 OCSP 协议发起查询请求， CA 返回证书状态内容，然后浏览器接受证书是否可信的状态。 

- HSTS ：  
(HTTP  Strict Transport Security)  a way for sites to elect to always use HTTPS 。 http 协议响应头。要求浏览器通过 https 来访问网站。
避免用户试图用 "http://" 访问时，通过 Web Server 的 301/302 重定向到 "https:// " 方式（节省一个 RTT ）。
  **需要浏览器支持**  
  
- RTT ：  
(round trip time ) （网络协议）交互时间  
  
- SPDY :  
（ means  SPEEDY ）。 Google 开发的，为了改善 Http1.x 的 TCP 应用层协议。 修改了 HTTP 的请求与应答在网络上传输的方式，当使用 SPDY 的方式传输， HTTP 请求会被处理、标记简化和压缩。 关键特性已被吸收到了 HTTP/2 协议中。    
  

###  1.2 规避已知的风险  

#### § 漏洞及风险

- 规避已知的 HeartBleed 漏洞的涉及协议及产品版本    
2014 年 4 月漏洞暴露。      
对 2014 年 4 月前发布的基础系统，都要上官网确认是否存在安全漏洞，确认安装了必要的安全补丁。  
  
- 明确知道存在漏洞的基础系统（部分）      
centos  6 系列（ 6.5 ， 6.6 及以后，不含 7 以后） 涉及到 HeartBleed , 还包括 Shellshock, PODDLE 漏洞。   

**参考**： 

>[Cent OS 官方对 HeartBleed ,PODDLE的安全说明] [^HeadBleed]  

- 存在漏洞的协议    
OpenSSL 版本为 1.0.1 至 1.0.1f （含）。较新版本（ 1.0.1g 以上）及先前版本（ 1.0.0 分支以前）不受影响。 OpenSSL 最新版本 1.1.0c （ 2016-11-10 发布） 
SSL3.0 （ POODLE 攻击）， TLS1.0 （ 如 RC4 和 BEAST 攻击 ） 

**参考**：

>[Turn off SSL3.0 and TLS 1.0] [^SSL3]   

- 已知的加密算法的风险 
    + MD5  ， SHA － 1 ，都不再适合用于密钥交换和加密通信（一致性校验）  
    + RSA 算法需要使用 2048bit 或以上的密钥才能保证安全性（ 1024bit 密钥已经不再安全）  
    + ECC 算法需要 224bit 以上的密钥才能保证安全性（通常为 256bit ）  
    + RC4 （伪随机数流加密算法），已经在新版本 TLS 中被禁用   。  
        google 使用的 ChaCha20 已经在 RFC 7539 中标准化  ,性能较之AES-GCM更好        
    + TLS1.1 ， TLS1.2 尚未发现安全漏洞。  
    
**参考** :  

>[ 建议用 AES-GCM 取代RC4 ] [^AES-GCM]   
>[ 通俗版椭圆曲线算法理论 ][^pediy6014]     
   
#### § 针对协议与算法的常见攻击

- BEAST:  
    ( Browser Exploit Against SSL/TLS)  
   利用块加密的特性，破解出密文中的某块明文，比如 cookies 中的 SessionKey 。
    TLS1.1 以后的版本已经修正了风险。但是攻击者可以通过 Downgrade Attack （参见：降级攻击）来试图迫使客户端降到 TLS1.0 来实施攻击。  
**参考**： 

>[ 一篇通俗易懂的 BEAST 原理的 BLOG ][^BEAST]  

- Downgrade Attack 降级攻击  
   包括： 降级攻击一般包括两种：加密套件降级攻击 (cipher suite rollback) 和协议降级攻击（ version roll back ）。    
降级攻击的原理就是攻击者伪造或者修改 client hello 消息，使得客户端和服务器之间使用比较弱的加密套件或者协议完成通信。  
**参考**： 

>[ 利用 TLS_FALLBACK_SCSV 协议值来防止降级攻击 ][^TLS_FB]


- HeartBleed[^HeadBleed]：    
   Heartbleed 是存在于 OpenSSL 加密软件库内的一个严重漏洞。此弱点让受保护的信息可以被盗取，它们本来是受 SSL ／ TLS 互联网加密协议所保护的。 SSL ／ TLS 为互联网上的应用程序提供安全通讯及私隐，这些程序包括有网页、电邮、即时通讯（ IM ）及某些虚拟专用网络（ VPN ）。

- POODLE:  
（ Padding Oracle On Downgraded Legacy Encryption ）。是协议降级攻击 version roll back 方式的一种。迫使浏览器降级至 SSL 3.0 而进行的中间人攻击。假若攻击者能成功利用此漏洞，他们平均只需 256 个 SSL 3.0 请求便能看穿加密信息内的 1 个字符。

- TLS Renegotiation 重新协商攻击  
   包括： 
   加密套件重协商 (cipher suite renegotiation )、 协议重商攻击（ Protocol Renegotiation ）两类。  
  **最直接的保护手段是禁止客户端发起主动重新协商**   

**参考**： 

>[Understanding_the_TLS_Renegotiation ][^TLS_Ren]  
  
- XSS  攻击    
（ Cross-Site Scripting ）跨域脚本攻击   。在目标网站上执行非目标网站的原有脚本 。
既有利用浏览器漏洞，也有利用代码的漏洞进行脚本注入。通过 https 和响应头设置，可以减少
XSS 发生的可能。  
  **参见服务端响应头相关内容**       
 

## 二 开始行动    

###  2.1 首先要澄清的几个事项

#### § 把域名切换到 https ，只是切换工作的一小部分  
   - 部署证书、修改 WebServer 配置支持 https 只是开始    
   - 所需加载的资源，如 js ， css ，图片等都需要加入考量范畴。否则，依旧存在劫持的可能。

#### § https 的改造可以不考虑内部系统之间的调用  
   - 但是，如果是跨局域网的内部服务间的交互，我们需要考虑敏感信息在链路上的安全      
   
#### § 如果所需加载资源利用了第三方服务，如 CDN ，那么要做方案上的决策。  
-  信任 CDN ？ 把私钥交给 CDN ？  
    **相信 CDN 是安全的,这一行为本身就是不安全的。**
-  仅利用 CDN 的动态加速， TCP 代理，不缓存内容？    
    **这样就大大降低了 CDN 的价值。**
-   寻找类似 CloudFlare ，提供 Keyless SSL 服务的 CDN 服务商。[^CloudF]       
    **不用提供私钥，不用使用公共域名和证书**   
    **CloudFare 已经与百度合作**  

#### § 把资源先改成 http 和 https 都支持？？  
-  注意修改代码中的链接 , 协议无关地址（ Protocol-less URLs ）。      
``` href=\"http://...\"    改为   href=\"//...\" ```               
   **问题：存在不兼容的可能，上线前需要严格测试**

#### § https 下重建连接的代价成本要远比 http 高    
-  http/2 的头部压缩（ header compression ） , 连接多路复用（ one connection for parallelism ）等可以改善 TLS 带来的成本。    
   **目前平均每个页面有 80 个部件 (Assets) ，平均每个头 (Header) 有 1400bytes**  
-  http/2 可以完全兼容 http1.1  
**参考**:

>[HTTP2 的 FAQ][^HTTP2]  

#### § 必须关注证书采购的成本  
-  购买证书的投入，包括首次购买及续费成本    
-  未来运维管理的成本      


#### §  协议支持的，未必代表厂商（浏览器， Server 等）支持      
-  必须要用特定的配置（兼容）清单 去验证所有的修改            


### 2.2 主要行动Step by Step  

#### § Step1 申请（购买）CA证书
  
- 需要根据自身业务使用域名的实况来确定采购的证书类型和数量。  
  对现有域名资源进行适当的梳理和规划，可以节省证书和未来运维的成本。  
   
- DV(Domain Validation SSL Certificate ) ,OV( Organization Validation SSL Certificate ),EV(  Extended Validation SSL Certificate )  与保护的域名数量之间没有直接关联    

-  SAN SSL 证书   Subject Alternative Name certificates 又称为 UCC 证书 – Unified Communication Certificates.   
  可以保护若干数量的域名。超过默认数量，可能需要支付额外费用 。   （具体视厂商而定）    
    
-  通配符 证书   Wildcard SSL  Certificate    ， 也称为泛域名证书，  
    可以保护下一级的子域名，及其本身。    
    即：如要保护多个四级域名，需要购买针对三级域名的通配符型的证书。 

-  不同的 CA 证书供应商，价格差异较大。    

- 正在兴起的 **Let's Encrypt**免费证书[^Letsen] , [^acme-tiny]。 
 
**示例：**    

>如：需要保护类似 ＊ .weixin.example.com , ＊ .api.weixin.example.com ，以及上层的 example.com 和 ＊.example.com  
**【需要两张泛域名证书】**  


#### § Step2  生成和下载证书

- 证书供应商会都会提供管理和生成、下载证书的管理入口  
- 每张证书都是和特定的一个或者一组域名的保护相关联（生成）的    
- 将下载证书部署到服务端特定的位置     


#### § Step3 选择加密套件Cipher Suite

-  每个利用 https 的客户端都有一组支持的加密套件（ Cipher Suite ），  
   如果这组套件与服务器上的套件列表没有交集，则无法完成协商，握手失败。

-  每个套件都是一组算法存在的，其中一般包含：  

   * 密钥交换（ Key Exchange ）  
   * 证书签名认证（ Authentication ）  
   * 加密算法（ Encrytion ）  
   * 内容完整性（一致性）校验算法（ Message Authentication Code ，简称 MAC ）  

-  [iana.org 管理的完整套件列表 ][^iana]   

- openssl 不同版本支持的套件不同，可以通过命令查看：    
  ```  $ openssl ciphers -V | column -t ```   

  命令运行后，展现以下形式内容 , 其中第一列为套件的编号：  
  
  ```  
    0xC0,0x30  -  ECDHE-RSA-AES256-GCM-SHA384    TLSv1.2  Kx=ECDH        Au=RSA    Enc=AESGCM(256)    Mac=AEAD  
    0xC0,0x2C  -  ECDHE-ECDSA-AES256-GCM-SHA384  TLSv1.2  Kx=ECDH        Au=ECDSA  Enc=AESGCM(256)    Mac=AEAD  
    0xC0,0x28  -  ECDHE-RSA-AES256-SHA384        TLSv1.2  Kx=ECDH        Au=RSA    Enc=AES(256)       Mac=SHA384  
    0xC0,0x24  -  ECDHE-ECDSA-AES256-SHA384      TLSv1.2  Kx=ECDH        Au=ECDSA  Enc=AES(256)       Mac=SHA384  
    0xC0,0x14  -  ECDHE-RSA-AES256-SHA           SSLv3    Kx=ECDH        Au=RSA    Enc=AES(256)       Mac=SHA1  
    0xC0,0x0A  -  ECDHE-ECDSA-AES256-SHA         SSLv3    Kx=ECDH        Au=ECDSA  Enc=AES(256)       Mac=SHA1  
    0xC0,0x22  -  SRP-DSS-AES-256-CBC-SHA        SSLv3    Kx=SRP         Au=DSS    Enc=AES(256)       Mac=SHA1  
  ....  

  ```
    
**参考**：   

>-  非对称密钥交换算法 key Exchange 。      
建议优先使用 ECDHE ，禁用 DHE ，次优先选择 RSA 。  
-  证书签名算法 。    
由于部分浏览器及操作系统不支持 ECDSA 签名，目前默认都是使用 RSA 签名，其中 SHA1 签名已经不再安全， **chrome 及微软 2016 年开始不再支持 SHA1 签名的证书** [^SHA1]      
-  对称加解密算法。      
优先使用 AES-GCM 算法，针对 1.0 以上协议禁用 RC4 （rfc7465）。    
-  内容一致性校验算法。      
Md5 和 SHA1 都已经不安全，建议使用 SHA256 等安全哈希函数。    
  

#### § Step4  修改服务器配置   

- 修改 Nginx 配置 （包括指定证书的路径）
- 是否在 Nginx 的配置中加入不同的响应头，需要小心评估。    
  例如： 如下响应头可以自动将 http 资源请求，升级会 https 请求。  
  （需要保证资源路径相同，包括第三方资源可支持 https ）  

>W3.org上的例子 <https://www.w3.org/TR/upgrade-insecure-requests/> :       

```
  server {
    ...

    add_header Content-Security-Policy upgrade-insecure-requests;

    ...
  }

```


-  下载 openSSL 包

>示例 <https://github.com/diafygi/acme-tiny > : [^acme-tiny]     

```  
    server {  
      listen 443;  
      server_name yoursite.com, www.yoursite.com;   
      ssl on;  
      ssl_certificate /path/to/chained.pem;  
      ssl_certificate_key /path/to/domain.key;  
      ssl_session_timeout 5m;  
      ssl_protocols TLSv1.1 TLSv1.2;  
      ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;  
      ssl_session_cache shared:SSL:50m;  
      ssl_dhparam /path/to/server.dhparam;  
      ssl_prefer_server_ciphers on;   
      ...the rest of your config  
    } 
```

-  更多内容可参见后文 [重点行动项的进一步说明](#section-10)   

**参考**   

>[ 官方 Module ngx_http_ssl_module][^ngx_http]


#### § Step5 修改代码

- 采用 https 协议 "https://..." ，或者改为 协议无关的方式 “ //...”   
  改为协议无关 “//...” 方式，是种很有风险的方式，存在网络供应商篡改，导致资源路径失败的可能    
- 页面从存在任何不安全的链接（ http ），浏览器地址栏的小绿锁 就会消失。      
   **要去掉所有可能导致警告和绿锁消失的链接 **  

- 注意可能从配置文件 或者 数据库字段中生成的 地址  
     
- 注意所有的外链和外部接口    



**参考**： 

>- [Move a site with URL changes][^Move]  
>- [RFC3986][^RFC3986]  


#### § final Step  监控日志

-  必须关注和分析服务器日志，确定资源请求失败的客户端，或者资源本身。
-  监控日志，只是一轮改造的最后一步，更可能是新一轮迭代的开始。  

##  三 重点行动项的进一步说明

### 3.1 证书问题

-  通过 Nginx ，在同一 IP 上部署多个 HTTPS 站点，会存在证书冲突的问题   ，除了 SNI （ Server Name Indication ）外，利用 SAN 证书也可以解决这个问题      
   **Windows XP IE6~8 、 Android 2.x Webview 都不支持 SNI**   


### 3.2 服务端响应头  

**参考示例**
可以参考和研究其它网站的响应头，然后结合自身实况来决定怎么做  

>Google:          
  
```
  x-content-type-options: nosniff  
  x-frame-options: SAMEORIGIN  
  x-xss-protection: 1 ; mode=block  

```
  
>Twitter:    

```   
  strict-transport-security: max-age= 631138519
  x-frame-options: SAMEORIGIN
  x-xss-protection: 1 ; mode=block
```
  
>PayPal:  

```   
  X-Frame-Options: SAMEORIGIN
  Strict-Transport-Security: max-age= 14400
```

>Facebook 则使用了这些（配置了详细的 CSP ，关闭了 XSS 保护）:    

```  
  strict-transport-security: max-age= 60
  x-content-type-options: nosniff
  x-frame-options: DENY
  x-xss-protection: 0
  content-security-policy: default-src *;script-src https://*.facebook.com http://*.facebook.com https://*.fbcdn.net http://*.fbcdn.net *.facebook.net *.google-analytics.com *.virtualearth.net *.google.com 127.0 . 0.1 :* *.spotilocal.com:* chrome-extension://lifbcibllhkdhoafpjfnlhfpfgnpldfl 'unsafe-inline' 'unsafe-eval' https://*.akamaihd.net http://*.akamaihd.net;style-src * 'unsafe-inline' ;connect-src https://*.facebook.com http://*.facebook.com https://*.fbcdn.net http://*.fbcdn.net *.facebook.net *.spotilocal.com:* https://*.akamaihd.net ws://*.facebook.com:* http://*.akamaihd.net https://fb.scanandcleanlocal.com:*;
```
    
### 3.3 让交互更快的策略 
**参考建议**  

>采取任何优化策略前，先要对内部基础设施（包括web server，用户浏览器使用分布等），做清晰梳理。然后再寻找、评估和采用针对性的策略。

####  § TCP FAST OPEN       
  RFC7413  
  TFO 需要高版本内核的支持， linux 从 3.7 以后支持 TFO ，但是目前的 windows 系统还不支持 TFO ，所以只能在公司内部服务器之间发挥作用。
 


####  § OCSP Stapling      

  + OCSP 方式向 CA 站点检查证书状态请求， CA 返回证书状态（是否有效／可信／撤销等）的过程，会因为 CA 站点服务器的位置，网络的稳定性等因素，会额外消耗大量 RTT 的时间。

  + OCSP 装订 /( 封套 ). 是一个 TLS 证书状态查询扩展，作为在线证书状态协议的代替方法对 X.509 证书状态进行查询   。
  服务器在 TLS 握手时发送事先缓存的 OCSP 响应，用户只需验证该响应的有效性而不用再向数字证书认证机构（ CA ）发送请求。

  + 服务器端的支持情况：  
```
  Httpd ： 2.3.3          
  Nginx ： 1.3.7  
  IIS ： Windows Server 2008  
  HAProxy ： 1.5.0  
```

  + 浏览器的支持情况：主流浏览器都已获得支持。  

  + Nginx 开启配置的方式    
``` ssl_stapling on;ssl_stapling_file ocsp.staple;
```

**参考**:      

>[OCSP_Stapling,wikipedia ][^OCSP]      
  
  
####  § Session 复用（ Session  Resume ）的两种方式      
- Session Cache &  Session Ticket   
  通过在服务端缓存已有的 Session 握手信息来实现：避免再次计算和非对程密钥的交换，节省一个 RTT 交互时间和计算成本。  
  
| 策略 | 存储方式 | 存储容量 | 恢复方式 | 共享能力 | 优缺点与风险 |  
|:------:|:------|:------ | :------ |:------ | :------ |   
|session cache | 服务端内存 | 4K session/M  | 通过 Session id 查找 | 单机多进程间 | 浏览器支持度高，不支持分布式，消耗内存 |    
|session ticket| 服务端文件 | - | 通过 Session 本身（加密成 ticket ） | 可以在多个 Server 间共享 | 支持度略低，需维护 ticket 加解密的额外 key |    

- 为了保证 Session Tickets 的可复用，在 配置Nginx 时不要使用 web server 随机生成的保存文件。  

**参考**:  

>[nginx 官方 ssl_session_ticket_key][^ticket]      


### 3.4 APP周边相关 
#### § 原生功能的安全
  - 仅使用 MD5 的 token 是很容易被伪冒的。对于涉及资金的交易，显然不够；  需要同时利用［敏感内容先加密（如 利用 RSA 算法），同时完整交易信息签名验证（比如利用 appKey+appSecret+userName+tradeTime
，进行 MD5 或者 SHA256 ）。

  - 进一步提高安全性：   
  每天至少进行一次握手，交换和更新下 token （密钥）。  

#### § 内嵌H5 页面的安全
  - 可以选择高层或底层的不同方式  
  （如 android 的 HttpsURLConnection ，或 SSLSocket ）   
  **参考**    

>[Android 官方 : Best Practices for  Security and Privacy](https://developer.android.com/training/best-security.html ) [^Android]     
>[ios  官方： App Transport Security (ATS) ](https://developer.apple.com/library/content/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW33) [^ATS]   


### 3.5 测试与验证

####  § 工具

- 免费的第三方检测平台。  
  如：[ssl Labs](https://www.ssllabs.com/ssltest/index.html)  
 
## 五 结语

回到本文的最前面，如果你只是想让你的网站，在地址栏出现一个“小绿锁”（也许它会时隐时现），那么一切都比较简单。借助cloudflare 提供的服务，也许只要半个小时就看上去万事大吉了。  

但是，对于一个有一定规模的网站来说，向https 、 http/2 的切换，目标显然不仅是看上去有了个“小绿锁”。真正实现在安全层面的提升和切换，一定会是一个艰难和充满风险的过程。 

## 六 延伸阅读

### 6.1 来自其它公司的实践

- 大型网站的https实践(百度)

<http://op.baidu.com/2015/04/https-s01a01/>


- 淘宝的https探索

<http://velocity.oreilly.com.cn/2015/ppts/lizhenyu.pdf>


### 6.2  其他链接
- [nginx官方: Open Source Nginx 1.9.5  Released with  HTTP/2 support](https://www.nginx.com/blog/nginx-1-9-5/)     

- [nginx1.8.x 配置 https](https://segmentfault.com/a/1190000004232801)


----  
**参考索引** 

[^CSPl3]:[Content Security Policy Level 3 的W3 官方文档](https://www.w3.org/TR/CSP/)   

[^HeadBleed]:[Cent OS 官方对 HeartBleed ,PODDLE的安全说明](https://wiki.centos.org/zh/Security)   

[^SSL3]:[Turn off SSL3.0 and TLS 1.0](https://www.ssl.com/how-to/turn-off-ssl-3-0-and-tls-1-0-in-your-browser/) 


[^AES-GCM]:[ 建议用 AES-GCM 取代RC4 ](https://blogs.technet.microsoft.com/srd/2013/11/12/security-advisory-2868725-recommendation-to-disable-rc4/)     

[^pediy6014]:[ 通俗版椭圆曲线算法理论 ](http://www.pediy.com/kssd/pediy06/pediy6014.htm)  

[^BEAST]:[一篇通俗易懂的 BEAST 原理的 BLOG ](http://securityalley.blogspot.com/2014/07/ssltls-beast.html)  

[^TLS_FB]:[ietf: 利用 TLS_FALLBACK_SCSV 协议值来防止降级攻击 ](https://tools.ietf.org/html/draft-ietf-tls-downgrade-scsv-00#page-3)  

[^TLS_Ren]:[Understanding_the_TLS_Renegotiation](http://www.educatedguesswork.org/2009/11/understanding_the_tls_renegoti.html)   

[^CloudF]:[Cloudflare one-click SSL](https://www.cloudflare.com/ssl/)  


[^HTTP2]:[HTTP2的FAQ](https://http2.github.io/faq/)  

[^Letsen]:[Let's Encrypt 免费证书](https://letsencrypt.org/getting-started/)   

[^acme-tiny]:[issue and renew Let's Encrypt certificates](https://github.com/diafygi/acme-tiny) 

[^iana]:[iana.org 管理的完整套件列表](https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-4)  

[^SHA1]:[chrome 及微软 2016 年开始不再支持 SHA1 签名的证书](http://googleonlinesecurity.blogspot.jp/2014/09/gradually-sunsetting-sha-1.html)   

[^ngx_http]:[官方:配置 Module ngx_http_ssl_module ](http://nginx.org/en/docs/http/ngx_http_ssl_module.html)  

[^Move]:[Google:  Move a site with URL changes](https://support.google.com/webmasters/answer/6033049?hl=en&ref_topic=6033084&rd=1)   

[^RFC3986]:[RFC3986:(URI): Generic Syntax](https://tools.ietf.org/html/rfc3986#section-4.2)  

[^OCSP]:[OCSP_Stapling,wikipedia](https://en.wikipedia.org/wiki/OCSP_stapling) 

[^ticket]:[nginx官方：ssl_session_ticket_key](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_ticket_key)   

[^Android]:[Android官方: Best Practices for  Security and Privacy](https://developer.android.com/training/best-security.html) 

[^ATS]:[ios 官方： App Transport Security (ATS) ](https://developer.apple.com/library/content/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW33)    
