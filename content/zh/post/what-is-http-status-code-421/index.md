---

title: "一只笨妞对他的 HAProxy 配置做了一点优化，这是 SNI 路由发生的变化"
subtitle: "How I messed up with SNI routing by issuing a wildcard tls cert instead of the per-site certs bundle."

# Summary for listings and search engines
summary: "HTTP/2 的连接复用策略更为激进，当你观察到连接的 URI 匹配不上实际被路由往的后端时，可以注意下这个坑哦。"

# Date published
date: "2021-10-24T03:40:00Z"

# Date updated
lastmod: "2021-10-24T14:25:00Z"

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false

math: true

toc: true

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com/photos/jH5nGtqr7KI)'
  focal_point: "Smart"
  placement: 1  # Placement options: 1 = Full column width, 2 = Out-set, 3 = Screen-width
  preview_only: false
  alt_text: Bird's eye view of the historic Chatham County courthouse located in downtown Pittsboro, North Carolina.

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []

authors:
- admin

# 我的定义：tags 的提取按内文关键词的感觉走，反映实际内容主题。
tags:
- "Coalescing"
- "HAProxy"
- "HTTP/2"
- "Wildcard Cert"

categories:
- "Computer Network"
- "HAProxy"
- "中文"
- "坑"
- "折腾"
- "计算机网络"

---

{{< toc >}}

## How it started

1. 我一直在 HAProxy 配置中任何可以使用 ssl_fc_sni 采样的地方使用它，而不是 hdr(Host)。因为前者是 [L5 采样](https://cbonte.github.io/haproxy-dconv/2.4/configuration.html#7.3.4-ssl_fc_sni) ，后者是 [L7 采样](https://cbonte.github.io/haproxy-dconv/2.4/configuration.html#7.3.6-hdr) 。我期望在获得到足够进行路由的信息后就进行处理，而不是继续拆包下去（我觉得这样更有效率
2. 为了减少 HAProxy 匹配证书的压力，也减少 Let's Encrypt 那边的压力（尽管这两种压力根本不存在，主要是我想偷懒 XD）——我在已经确认正常部署的配置上用一张 wildcard 证书代替了原来的三张证书。
3. 客户端建立 HTTP/2 连接后，只要服务端证书能通过认证，后继请求无论 URI 是否不同都会重用这条连接——这是我简单总结的<q>人话版</q>，可能有不准确的地方，最好参考 [RFC 7540](https://datatracker.ietf.org/doc/html/rfc7540#section-9.1.1)

## How it's going

$\quad$$\quad$——幸运的是，这次部署的三个站点都是我需要立马配置的。在配置好第一个站点之后我立马发现打开第二个站点的行为很不对劲。因为我是在已经成功的部署上做的细微调整，所以我不至于在调试时误会到其他的方向上去。  
$\quad$$\quad$可惜当时时间很晚了，我只来得及用熟悉的手段比如查查实时流量日志之类，一番排查无果之后就选择了求助（学长永远的神）。我急着用所以倒是不后悔这样做，毕竟大大节省了时间，就是事后的成就感少了很多 QaQ……  
$\quad$$\quad$这里事后复盘一下，感觉可以得到一个日后排查问题的经验——对日志收集到一定的程度就该适可而止了，过于详细的日志只会增加研判的困难，扰乱自己的思路。  
$\quad$$\quad$~~像我这次在服务端排查问题却看到行为一如预期，这时就该切换到客户端这边看看（让我联想起以前玩 wireshark 的经历，同样也有很多现象在某一侧是很难观察到的，但是如果<q>一碗水端平</q>的话很快就能在另一侧发现端倪）。~~  
$\quad$$\quad$为了给这篇博客整理思路，复现问题时，我却能从服务端日志里非常明显地观察到请求被转发去了与 SNI 不同的后端。真是搞不懂自己那天到底是眼睛变漏勺了还是怎么回事……  

## Conclusion

$\quad$$\quad$造成这个问题的调整如下，新的配置用 `*.L1.domain.tld` 匹配 `site[1-4].L1.domain.tld`：
```diff
--- a/crt-list.txt
+++ b/crt-list-new.txt
@@ -1,4 +1 @@
-/etc/haproxy/certs/site1.domain.tld.pem [alpn h2,http/1.1] site1.domain.tld
-/etc/haproxy/certs/site2.domain.tld.pem [alpn h2,http/1.1] site2.domain.tld
-/etc/haproxy/certs/site3.domain.tld.pem [alpn h2,http/1.1] site3.domain.tld
-/etc/haproxy/certs/site4.domain.tld.pem [alpn h2,http/1.1] site4.domain.tld
+/etc/haproxy/certs/L1.domain.tld.pem [alpn h2,http/1.1] *.L1.domain.tld L1.domain.tld
```

{{< spoiler text="请先自己思考下这个问题哦，如果确认自己思考好了就来对答案吧！" >}}

**参考答案：**  
$\quad$$\quad$因为服务端提供的是 wildcard 证书，且 C/S 协商使用了 HTTP/2 连接，所以客户端（我的浏览器）在访问第一个站点之后再另开标签请求第二个站点时，满足了重用到第一个站点连接的条件。  
$\quad$$\quad$而这足够在我长期用惯的配置上造成问题，原因是服务端用 SNI 作为路由的判断依据，但服务端收到的 SNI 和客户端实际请求的 Host 是不同的（前者变成了被复用的上次连接的目标）。  
{{< /spoiler >}}

$\quad$$\quad$最终的解决方案是改回之前的部署方案，直接一口气签了三个证书（对不起 Let's Encrypt，但我真的很想要更多证书 XD）。之前签的 wildcard 证书还是保留下来了（仅仅两天之后就有了作用，现在被用来提供博客的证书服务~）

## Think now

{{< spoiler text="思考题1：根据上文提供的解决方案，如何修改以上的 `crt-list-new.txt` 文件内容呢？" >}}

**参考答案：**  
```
# The first declared certificate of a bind line is used as the default certificate,
# either from crt or crt-list option, which HAProxy should use in the TLS handshake if no other certificate matches.

# This certificate will also be used if the provided SNI matches its CN or SAN,
# even if a matching SNI filter is found on any crt-list.

# The SNI filter !* can be used after the first declared certificate to not include its CN and SAN in the SNI tree,
# so it will never match except if no other certificate matches.
# This way the first declared certificate act as a fallback.

# Refer to: https://cbonte.github.io/haproxy-dconv/2.4/configuration.html#5.1-crt-list

/etc/haproxy/certs/L1.domain.tld.pem [alpn h2,http/1.1] !*
/etc/haproxy/certs/site1.L1.domain.tld.pem [alpn h2,http/1.1] site1.L1.domain.tld
/etc/haproxy/certs/site2.L1.domain.tld.pem [alpn h2,http/1.1] site2.L1.domain.tld
/etc/haproxy/certs/site3.L1.domain.tld.pem [alpn h2,http/1.1] site3.L1.domain.tld
```
{{< /spoiler >}}

{{< spoiler text="思考题2：除了上文中的解决方案，还有别的解决方案吗？这个解决方案为什么能生效？" >}}

**参考答案：**  
$\quad$$\quad$在该处使用 `hdr(Host)` 代替 `ssl_fc_sni` 做 SNI 路由（学长给的建议，我还未尝试过。如果有错欢迎指出）。   
$\quad$$\quad$以下是讨论记录，欢迎参与探讨：  
> (学长) ……主要是如果没有 connection reuse（或者叫 coalescing），SNI 和 HTTP Host 原本应该是一致的（否则就叫可以叫做 domain fronting）。结果 HTTP/2 允许共用（连接）了，那就只能根据最直接的 HTTP Host 来判断了  

**P.S.**  
$\quad$$\quad$个人不建议配置 web server 向客户端发送 421 状态码，因为有部分客户端是不响应这个状态码的（如果局限在浏览器圈子里估计这个问题并不会有多大影响）。  
$\quad$$\quad$——但就像我这次搬之前的配置过来小改结果改出问题一样，配置中没有必要最好不要使用那些可能会给未来埋坑的设置（注意 421 状态码回应是可以被缓存的，这同时可能影响原本通过连接复用压榨出的性能）。  
{{< /spoiler >}}

## Refer

1. https://levelup.gitconnected.com/multiplex-tls-traffic-with-sni-routing-ece1e4e43e56
1. https://notfalse.net/48/http-status-codes#421-Misdirected-Request-8212
1. https://halfrost.com/http2-considerations/
2. https://serverfault.com/questions/977848/disable-http-2-connection-reuse-across-domains
3. https://serverfault.com/questions/916724/421-misdirected-request

## License

Copyright 2021-present [IceCodeNew](https://blog.icecode.xyz).

Released under the [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/legalcode) license.
