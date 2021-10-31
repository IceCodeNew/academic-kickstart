---

title: "巧用 HAProxy 配置反代搭建软件镜像"
subtitle: "Set up mirror service by deploying reverse proxy in HAProxy"

# Summary for listings and search engines
summary: "—「为什么这么折腾呢？Caddy 几句话就能解决的事情」 —「因为 HAProxy（山）就在那里」"

# Date published
date: "2021-10-30T08:00:00Z"

# Date updated
lastmod: "2021-10-30T14:35:00Z"

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: true

math: true

toc: true

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com/photos/hsg538WrP0Y)'
  focal_point: "Smart"
  placement: 1  # Placement options: 1 = Full column width, 2 = Out-set, 3 = Screen-width
  preview_only: false
  alt_text: Going through the tunnel of mirrors, we are directed to our destination.

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
- "HAProxy"
- "Reverse Proxy"
- "Wildcard Cert"
- "反代"

categories:
- "HAProxy"
- "中文"
- "坑"
- "折腾"

---

{{< toc >}}

## Introduction

$\quad$$\quad$有时难免会需要需要在 [`the SICK BRIC country`](https://www.ted.com/talks/michael_anti_behind_the_great_firewall_of_china) 的服务器上做点开发，这时候就不得不处理 [`the SICK BRIC country`](https://www.ted.com/talks/michael_anti_behind_the_great_firewall_of_china) 特有的网络问题了。  
$\quad$$\quad$你也许会问：不是已经有这么多大厂或高校提供镜像服务了么，为什么不用？——然而这些镜像服务只提供了有限范围的镜像，当你需要的内容恰好不在上面时，你要花多长时间、用多少精力来推动新镜像服务的上线呢？  
$\quad$$\quad$另外，这些镜像服务设置的思路也和个人使用有些不同——对于这些有大量用户使用的镜像服务，往往会定期同步上游文件到本地来分发（这样既能减轻上游服务器压力，也能利用 CDN 资源对更复杂的用例和更广泛的用户提供更好的体验）。那么因为存储空间限制或者是授权问题等因素，有些资源的镜像服务可能根本就无法提供。这时候部署自己的反代，重新掌握主动权就显得有必要了。  
  
$\quad$$\quad$这篇博客总结了 19 年在 HAProxy 上实现反代镜像服务的一次尝试，当时因为没吃透文档栽在一个大坑上爬不出来。两年后的今天我心血来潮决心一雪前耻，在学长的帮助下终于攻克了难关。  
$\quad$$\quad$——请注意 HAProxy 在这项任务上并不具备优势，因为它的工作范围仅限于 HTTP header 这一部分[^1]。如果你不是为了学习研究而参考这篇博客，建议选择 Caddy 来满足您的需要。其 v2 版本性能已获得很大提升，配合简洁的配置语法和开箱包含的诸多强大插件，你可以轻松实现对响应体中内容的替换，如此实现的反代才算是功能完全。  
$\quad$$\quad$至于我选择 HAProxy 的理由？~~当然是因为 HAProxy 就在那里~~——其实是希望通过折腾积累经验，对 HAProxy 有更深刻的认识。另外我对于引入额外的组件这件事也很犹豫，如果能通过现有技术栈满足我的需要，我就会竭力避免引入新的技术栈。  

## Things you will need to handle

1. 在反代诸如 `archive.ubuntu.com` 等站点时，要注意上游服务器会检测到发生了 HTTPS 降级，并拒绝服务（所以你们什么时候提供 HTTPS 服务啊 \*\*\*)。  
——解决方案是在 HAProxy 配置里特判 Host，如果是反代域名则不做 301 跳转，也不发送 `Strict-Transport-Security` 头（话说回来这个问题可能也就对我这种安全偏执狂是问题了 hhh。有好多站点上根本不做 HTTPS 重定向的，甚至还拿同样的域名分出 HTTP 和 HTTPS 两个站来提供并不相似的内容——这里点名批评鸟哥）。
2. **提前评估服务被滥用的可能及影响**。一些必要且部署和后继使用时都不太麻烦的措施其实只要肯稍微停下来花点时间就能想到。认真对待这件事真的很重要——举例来说 [`the SICK BRIC country`](https://www.ted.com/talks/michael_anti_behind_the_great_firewall_of_china) 因为推行 IPv6，政府网站大量采用 IPv6 代理来完成推进工作。但是因为代理服务完全敞开没有做过滤，变成了黑帽 SEO 的天堂[^2]。类似这样的案例还有很多，其影响想必不用我再费笔墨去说明吧（什么是 [security by obscurity](https://en.wikipedia.org/wiki/Security_through_obscurity) 啊.战术后仰.webp）。  
_<small>我有一个绝妙的解决方案，可惜这里写不下了——看到这里可别急着打我，后文会附上的！</small>_
3. 替换响应体中的 `Location` Header，使其指向反代域名。举个例子来说，你通过反代愉快地访问到了嗖嗖快的 GayHub 站，但是当你点击下载链接时却发现是通过慢吞吞的 GitHub 站在下载文件……这可不太妙。不过真正想要实现功能完全的反向代理，光修改响应头其实是不足够的（我的用例毕竟是以加速下载为主，恰好就只需要修改上游发来的重定向请求就够了）。  
——解决方案是在后端使用 [`http-response replace-header`](https://cbonte.github.io/haproxy-dconv/2.4/configuration.html#4.2-http-response%20replace-header) 方法，通过正则表达式匹配模式并替换。在我后面给出答案之前（这个答案可是花了我一个小时才调整到完美），请先试着思考如下两个问题：`Location` Header 中可能出现 URL 的哪些部分？HAProxy 的正则匹配引擎支持哪些模式的匹配？  
_<small>上面的这两个问题就算是现在的我也不敢说能回答得清清楚楚，拿出来提问读者只是为了提醒大家在配置过程排查问题的思路（比如这里用来匹配的模式到底引擎支持不支持、你这里括号套来套去的部分到底在 HAProxy 里是第几组返回的结果……等等问题），以及不要把正则表达式写得太宽泛，以至于很容易错误匹配到不该匹配的东西上。</small>_
4. 配置 HAProxy 通过 HTTPS 和后端（被代理的服务）连接的细节。你需要：首先，找到一种配置语法可以支持随着被代理服务的改变而改变的后端 IP 地址（何况你也不应该假设这些服务的 IP 地址长期不变）；指明 HAProxy 需要通过 HTTPS 和对方通信，不然你只能得到一个被转发回来的重定向请求 XD；告诉 HAProxy 如何验证建立的 HTTPS 请求（这里坑了我两下，第二回是学长帮助我才得以解决）；最后，不要忘记了前言介绍部分提到的问题——既然有的上游服务会检测 HTTPS 降级，又不能给所有的后端服务都配置 HTTP，那该怎么办呢？  
——这么复杂的问题，解决方案请容我在后文详细展开 XD

## Solutions

$\quad$$\quad$以下从第 2 个问题起依次给出解决思路和配置片段。如果你对 HAProxy 也有一定了解，请在展开答案之前先试着自己思考一下答案吧！有任何疑问或者意见都欢迎在评论区进行交流~：

{{< spoiler text="对第 2 个问题的解决方案：" >}}
```
backend rp-mirror-https-be
…
  acl rp_white_domain             var(txn.ori_host)        dl.yarnpkg.com.reverse.proxy.tld
  acl rp_white_domain             var(txn.ori_host)        repo.mongodb.org.reverse.proxy.tld
  acl rp_white_domain             var(txn.ori_host) -m dom github.com.reverse.proxy.tld
  acl rp_white_domain             var(txn.ori_host) -m dom githubusercontent.com.reverse.proxy.tld
…
  http-request deny               errorfiles global.errorfiles if !rp_white_domain OR { var(txn.ip_striped_dom) -m ip 10.0.0.0/8 127.0.0.0/8 100.64.0.0/10 172.16.0.0/12 192.0.0.0/24 192.168.0.0/16 198.18.0.0/15 ::1/128 fc00::/7 }
…
```
{{< /spoiler >}}

#### _Think now_
{{< spoiler text="1. 从后端名称可以看出，HAProxy 配置中对收到的 HTTP 反代请求和 HTTPS 反代请求分别做了处理（这正是我解决问题 1 的思路）。那么当要反代的域名范围很广时，这份 ACL 条件列表就会因为被重复两遍占用很多空间，也造成阅读或修改配置时的不便。这时候应该怎么做呢？" >}}
> <small>答案可以在这篇博客中找到： https://www.haproxy.com/blog/introduction-to-haproxy-acls/ </small>
{{< /spoiler >}}
  
{{< spoiler text="2. `var()` 和 `txn.` 在这里分别起什么作用？`txn.` 的兄弟姐妹们都被用在什么场合呢？" >}}
> <small style="font-family:'Courier New'">The name of the variable starts with an indication about its scope. The scopes allowed are:  
> "proc" : the variable is shared with the whole process  
> "sess" : the variable is shared with the whole session  
> "txn"  : the variable is shared with the transaction (request and response)  
> "req"  : the variable is shared only during request processing  
> "res"  : the variable is shared only during response processing  
> This prefix is followed by a name. The separator is a '.'. The name may only contain characters 'a-z', 'A-Z', '0-9', '.' and '_'.  </small>
{{< /spoiler >}}
  
{{< spoiler text="3. 尝试给出为变量 `ori_host` 和 `ip_striped_dom` 赋值的 HAProxy 配置：" >}}
```
resolvers mydns
  nameserver                      quad9_1   9.9.9.9:53
…
frontend fe_https
  http-request set-var(txn.ori_host) ssl_fc_sni,lower
  http-request set-var(txn.striped_dom) var(txn.ori_host),regsub(\"(^.+)\.reverse\.proxy\.tld(:\d+)?\",\"\1\2\",i) if is_mirror
  http-request do-resolve(txn.ip_striped_dom,mydns,ipv4) var(txn.striped_dom)
```
{{< /spoiler >}}

{{< spoiler text="4. 除了对反代域名及其解析得到的 IP 进行校验外，你还有其他加固服务的思路吗？" >}}
> <small>举例来说，可以限制访客的 GeoIP 为仅限中国（甚至可以限制为仅开发机的 IP，如果条件允许的话）。实现思路可以参考以下几篇博客：
- https://www.haproxy.com/blog/use-geoip-database-within-haproxy/ 
- https://www.haproxy.com/blog/bot-protection-with-haproxy/ </small>
{{< /spoiler >}}

---

{{< spoiler text="对第 3 个问题的解决方案：" >}}
```
backend rp-mirror-http-be
  http-response replace-header    Location ^((?:https?:\/\/)?(?:[^@\/\n]+@)?[^\/\n:]+)(.*) \1.reverse.proxy.tld\2
```
{{< /spoiler >}}

---

{{< spoiler text="对第 4 个问题的解决方案：" >}}
```
backend rp-mirror-https-be
  http-request set-header         Host %[var(txn.striped_dom)]
  http-request set-dst            var(txn.ip_striped_dom)
  http-request set-dst-port       int(443)
  server rp-mirror                0.0.0.0:0 ssl sni var(txn.striped_dom) ca-file /etc/ssl/certs/ca-certificates.crt
``` 
{{< /spoiler >}}

#### _Think now_
1. 请指出上面的配置中的哪一部分解决了问题 4 中提出的哪一个问题（如果你可以动手实践看看删掉一部分配置会发生什么错误，想必会收获不少~）：  
<small>请参见文档[^3] [^4]</small>

## License

Copyright 2021-present [IceCodeNew](https://blog.icecode.xyz).

Released under the [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/legalcode) license.


[^1]: https://discourse.haproxy.org/t/rewrite-of-response-body/3995
[^2]: https://paper.seebug.org/1277/#1-ipv6
[^3]: https://cbonte.github.io/haproxy-dconv/2.4/configuration.html#4.2-http-request%20set-dst
[^4]: https://cbonte.github.io/haproxy-dconv/2.4/configuration.html#4.2-server
