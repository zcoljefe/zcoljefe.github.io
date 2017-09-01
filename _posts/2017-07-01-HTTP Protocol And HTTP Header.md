---
layout: default
title:  "http协议格式详解"
date:   2017-07-01 16:16:16
categories: java
---
虽然网络已然十分普及了，但是数据在网络中是如何传输的？以什么样的方式传输的？如何防止跨站点请求伪造？这些都涉及到http请求报文，在报文中我们将能看到许多在浏览器上看不到的信息。
<!--more-->
**HTTP 请求数据**  
![]({{ site.baseurl }}/images/13-27-09.jpg)

**请求实例：**  
```
GET /siamese/airsearch/international/search.htm HTTP/1.1
Host: 10.6.108.98:8801
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Referer: http://10.6.108.98:8801/siamese/index.htm
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8
Cookie: JSESSIONID=F62F7A4ED2A91446B0552AABF659D9ED; userType=1
```

**HTTP 响应数据**  
![](/images/13-31-09.jpg)

**响应实例：**  
```
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
Pragma: No-cache
Cache-Control: no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Content-Type: text/html;charset=UTF-8
Content-Language: zh-CN
Transfer-Encoding: chunked
Date: Tue, 14 Mar 2017 05:53:25 GMT
```