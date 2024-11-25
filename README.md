
# 目录


* [目录](https://github.com)
* [简介](https://github.com)
* [详情](https://github.com)
	+ [请求](https://github.com)
		- [DoH](https://github.com)
		- [DoT](https://github.com)
	+ [返回](https://github.com)
		- [DoH](https://github.com)
		- [DoT](https://github.com)
	+ [c\-ares的使用](https://github.com)
		- [打包](https://github.com)
		- [解析](https://github.com)


# 简介


DNS over HTTPS利用HTTP协议的GET命令发出经由JSON等编码的DNS解析请求。较于传统的DNS协议，此处的HTTP协议通信处于具有加密作用的SSL/TLS协议（两者统称作HTTPS）的保护之下。但是，由于HTTPS本身需要经由多次数据来回传递才能完成协议初始化，其域名解析耗时较原DNS协议会显著增加。[\[1]](https://github.com)DNS over TLS（缩写：DoT）是通过传输层安全协议（TLS）来加密并打包域名系统（DNS）的安全协议。[\[2]](https://github.com)本文基于[RFC 8484 \- DNS Queries over HTTPS (DoH)](https://github.com):[slower加速器](https://chundaotian.com)介绍DoH协议的详情。


此外，本文还基于Boost的Asio、Beast实现了DoH和DoT的解析器[ink19/Boost.dns](https://github.com)。


# 详情


DoH和DoT的请求包编码方式就是基于普通的DNS编码，具体可以参考这一篇博文[\[3]](https://github.com)。为了简便，本文使用了`c-ares`[\[4]](https://github.com)，对DNS的请求和回包进行解析。


## 请求


### DoH


DoH的请求基于HTTPS，其将DNS的请求包使用base64编码，放在了`dns`参数上，如：



```
GET /dns-query?dns=BAABAAABAAAAAAABBGhvbWUFaW5rMTkCY24AAAEAAQAAKQIAAAAAAAAA HTTP/1.1
Host: doh.pub
User-Agent: Boost.Beast/347
Accept: application/dns-message

```

其中需要注意的是`Accept`为`application/dns-message`。


### DoT


DoT是直接数据流传输，因此不需要使用base64编码，但是为了方便读写，需要在请求包前增加两个字节的请求长度。如：



```
std::vector req_buff;
uint16_t send_len = req.size();
send_len = htons(send_len);
req_buff.push_back(asio::buffer(&send_len, sizeof(send_len)));
req_buff.push_back(asio::buffer(req.data(), req.size()));

```

## 返回


### DoH


DoH的回包放在body中，为二进制。直接读取解析即可。


### DoT


DoT的回包和请求包类似，使用二进制，并有两个字节为前导长度。


## c\-ares的使用


### 打包


打包使用函数`ares_create_query`，签名为



```
#include 
 
int ares_create_query(const char *name,
                      int dnsclass,
                      int type,
                      unsigned short id,
                      int rd,
                      unsigned char **buf,
                      int *buflen,
                      int max_udp_size);

```

获取的buf，使用结束后需要使用`ares_free_string`释放。


`dnsclass`和`type`在中定义；`id`为16位，用于标记请求id；`rd`用于标识是否需要递归解析。


### 解析


解析使用`ares_parse_xxxx_reply`函数，比如A类型解析的签名为



```
#include 
 
int ares_parse_a_reply(const unsigned char *abuf, int alen,
                       struct hostent **host,
                       struct ares_addrttl *addrttls, int *naddrttls);

```

其中host需要使用`ares_free_hostent`进行释放。


使用方法为（注：该函数的使用方法在官方文档中描述的比较模糊，本文是在github中搜索ares\_parse\_a\_reply后，找到了node库使用该函数的方法[\[5]](https://github.com)）



```
struct hostent *hosts;
int ret = ares_parse_a_reply((const unsigned char *)rsp.data(), rsp.size(),
                              &hosts, NULL, NULL);

for (int i = 0; hosts->h_addr_list[i] != NULL; ++i) {
  uint32_t ip = ntohl(*(uint32_t *)hosts->h_addr_list[i]))
}
ares_free_hostent(hosts);

```



---



1. [DNS over HTTPS \- 维基百科，自由的百科全书](https://github.com) [↩︎](https://github.com)
2. [DNS over TLS \- 维基百科，自由的百科全书](https://github.com) [↩︎](https://github.com)
3. [自己动手实现DNS协议 \- 苍青浪 \- 博客园](https://github.com) [↩︎](https://github.com)
4. [c\-ares documentation \| c\-ares: a modern asynchronous DNS resolver](https://github.com) [↩︎](https://github.com)
5. [node/src/node\_cares.cc at 496be457b6a2bc5b01ec13644b9c9783976159b2 · kuno/node](https://github.com) [↩︎](https://github.com)



