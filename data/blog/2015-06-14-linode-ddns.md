---
title: '通过 Linode API 实现动态 DNS'
date: '2015-06-14'

tags: ['Linux', 'Linode', 'DDNS']
---

Linode 的 VPS 用了也有几年了，最近才发现它的 API 的实用性，其中一项就是可以简单又灵活的实现动态 DNS。

要使用 DNS 相关的 API 先决条件当然是得由 Linode 来托管你的域名，关于 Linode 的 DNS 托管可以看下[这里](https://www.linode.com/docs/networking/dns/dns-manager)。下面进入主题：

### **服务器端的配置**

进入账号后点击 DNS Manager 标签页，
![DNS Manager](/static/images/linode-ddns/1.png)
然后在下面的列表里点击自己的域名，进入该域名的编辑页面。

之后点击 A/AAAA Records 列表下的“Add a new A record”链接，新增一个域名和 IP 的对应关系。
![Add a new record](/static/images/linode-ddns/2.png)
Hostname 填想要配置成的动态域名，例如 home.example.com；IP Address 可以先随便写一个，比如 127.0.0.1，因为当动态域名配置好之后这个 IP 地址是会被自动更新的；最后的 TTL 应该设置成一个稍短的时间，因为一般来说 ISP 会比较频繁的更新你的 IP 地址，这样域名应该设置较短的存活时间以及时反映 IP 的变化。
接下来要实现在客户端(使用动态域名指向的 IP 的设备)周期性的更新刚刚配置的域名所对应的 IP。

### **客户端的配置**

Linode 提供了[不少 API](https://www.linode.com/api/dns)用以实现对 DNS 的查询和操作。想要使用这些 API 得先申请一个 API Key。

点击页面右上角的 my profile，然后点击 API Keys 标签页。
![API Key](/static/images/linode-ddns/3.png)
Label 处填写一个 API Key 的标签，比如 DDNS。Expires 选 Never，永远不过期。创建之后把 key 复制下来保存好。

接下来在浏览器里打开下面这个链接，用这个 API 来查看你所有的域名：

```
https://api.linode.com/?api_key=your-api-key&api_action=domain.list
```

api_key 要等于刚才申请的 key。返回的结果是一个 JSON 对象，找到你的域名对应的 DOMAINID，记好。再打开下面这个链接查看你的域名下的所有记录：

```
https://api.linode.com/?api_key=your-api-key&api_action=domain.resource.list&domainid=your-domain-id
```

api_key 和 domainid 都要换成你刚刚记下来的内容。在返回的结果里找到你的动态域名(home)对应的 RESOURCEID。最后这个 API 就是会把域名的 IP 更新为 API 调用的地址：

```
https://api.linode.com/?api_key=your-api-key&api_action=domain.resource.update&domainid=your-domain-id&resourceid=your-resource-id&target=[remote_addr]
```

照例把 api_key，domainid 和 resourceid 对应成刚才记下来的内容，最后的[remote\_addr]代表的就是 API 调用方(打开这个链接的电脑)的 IP 地址，不需要更改。最后的任务就是在客户端创建一个 cron job 来周期性的调用 Linode API：

```bash
crontab -e
```

打开当前用的 cron 任务文件。在文件里加一行：

```bash
*/30 * * * * /bin/echo `/bin/date`: `/usr/bin/wget -qO- --no-check-certificate https://api.linode.com/?api_key=your-api-key\&api_action=domain.resource.update\&domainid=your-domain-id\&resourceid=your-resource-id\&target=[remote_addr]` >> /var/log/linode_dyndns.log
```

每半小时更新一次动态域名对应的 IP。到这就大功告成了:)

P.S. 如果还没有 Linode 账号希望能用我的 referral 链接注册:P [注册 Linode](https://www.linode.com/?r=b5e79f5672ed45c37b58ea482f99d13d7f0d347e)
