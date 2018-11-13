# HTTP几种请求方式的含义及应用场景

整理自https://stackoverflow.com/questions/27030649/explain-and-example-about-get-delete-post-put-options-patch-h

https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html

## GET

GET请求是向服务端请求获取某个或某些资源（resource），比如查询数据库单个或list数据，服务端成功的话，一般状态码返回200。

## POST

POST请求是用来向服务端请求新增资源（resource），处理成功的话，服务端一般返回状态码201。

## PUT

PUT请求一般是用来向服务端请求修改某个已存在的资源（resource）,服务端一般返回状态码200，204等。

## DELETE

DELETE请求一般是用来向服务端请求删除某个已存在的资源（resource），服务端一般返回200，202等。

## PATCH

PATCH请求一般是对某个资源做局部修改,如个别字段。

PUT和PATCH区别：PUT和PATCH都是用来修改服务端某个资源的，但是PUT和PATCH修改时提交的数据是不同的，PUT是将整个资源的信息都提交到服务端，包括修改的，未修改的都提交到服务端，而PATCH只提交已修改的字段到服务端。而服务端对PUT请求应该是整体替换，PATCH请求只修改提交的字段。所以PUT请求应该是幂等的，即多次提交同一个请求，结果是相同的。

来自Stack Overflow：

https://stackoverflow.com/questions/28459418/rest-api-put-vs-patch-with-real-life-examples

![image-20180919102403487](/Users/wangwangxiaoteng/Library/Application Support/typora-user-images/image-20180919102403487.png)

## TRACE

TRACE请求没有使用过。w3网站上是这么介绍的：

`The TRACE method is used to invoke a remote, application-layer loop- back of the request message. The final recipient of the request SHOULD reflect the message received back to the client as the entity-body of a 200 (OK) response. The final recipient is either the

origin server or the first proxy or gateway to receive a Max-Forwards value of zero (0) in the request (see section 14.31). A TRACE request MUST NOT include an entity.

TRACE allows the client to see what is being received at the other end of the request chain and use that data for testing or diagnostic information. The value of the Via header field (section [14.45](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.45)) is of particular interest, since it acts as a trace of the request chain. Use of the Max-Forwards header field allows the client to limit the length of the request chain, which is useful for testing a chain of proxies forwarding messages in an infinite loop.

If the request is valid, the response SHOULD contain the entire request message in the entity-body, with a Content-Type of "message/http". Responses to this method MUST NOT be cached.`

## OPTIONS

OPTIONS请求一般是客户端向服务端判断对某个资源是否有访问权限。

## HEAD

HEAD请求一般是用来获取某个资源的metadata信息，比如说某份报告的关键描述信息等。