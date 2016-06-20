---
layout: post
title:  "[RHCE7] RH254 Chapter 5 Managing DNS For Servers Learning Notes"
description: "此文章記錄學習 RHCE7 RH254 Chapter 5 Managing DNS For Servers Learning Notes 留下的內容"
date: 2016-06-14 09:00:00
published: false
comments: true
categories: [linux]
tags: [Linux, RHCE, RH254]
---

5.1 DNS Concepts
================

- DNS 是一種分散式的資料庫伺服機，不同的 DNS 之間可以彼此交換資料

- 早期的名稱解析機制：**/etc/hosts**

- DNS 功能：FQDN & IP 互轉

```bash
# 正向名稱解析
$ dig www.uuu.com.tw

# 反向名稱解析
$ dig -x 8.8.8.8
```

- 可能只有正向解析資料庫，而沒有反向解析資料庫

- DNS 有三種：**Master**、**Slave**、**Cache Only**

- DNS 回應最後正確的資料時，會包含 **TTL**

## 5.2.2 Anatomy of DNS lookups

- 會先到 **/etc/resolv.conf** 檔案找到 DNS server 資訊

- 從 master/slave DNS 所得到的資料是最正確的 => 權威伺服機 (用 dig 查詢結果會得到 flag **aa**)

> dig @dns1.uuu.com.tw. www.uuu.com.tw

> 詢問權威伺服機，會得到 flag `aa`；反之則不會

## 5.2.3 DNS resource records

- 簡稱 **RR**，只有在實際架設 master/slave DNS 時才會用到

### type

- **A**：把名稱對向 IP

- **AAAA**：同上，但用在 IPv6 上

- **CNAME**：使用已經被宣告過的 **A** 來指定同樣的機器

- **NS**：表示 name server 的位址

```bash
# 向 168.95.1.1 詢問 uuu.com.tw 的 dns server 在哪裡
$ dig @168.95.1.1 -t ns uuu.com.tw
```

- **MX**：指定 mail gateway，還需要額外搭配 **A** record 指定 ip (達成企業外部的郵件路由)

- **PTR**：把 ip 對向名稱

- **SOA**：指定 domain 的 master 主機之用，所需要設定的參數都是給 slave dns 用的

- **TXT**：原本為 DNS record 設定註解之用(`abc  IN  TXT  "this is a test server"`)，但現在多被拿來作為 spam filter 之用

- **SRV**：用來標示 domain 內有提供哪些服務

------------------------------------------------------

5.2 Configuring a Caching nameserver
====================================

## 5.2.1 Caching nameservers and DNSSEC

- DNSSEC：作數位簽章用

### Caching nameserver

- caching nameservers 讓原本要跑到網際網路的 DNS 查詢流量都留在速度相對快速的區域網路內

- 每筆存在 caching nameserver 的記錄都有 TTL，時間到了就會到網際網路上進行更新

### DNSSEC Validation

- DNSSEC(DNS Security Extensions)是為了防止 cache poisoning 的問題


## 5.2.1 Configuring and administering unbound as a caching nameserver

透過 **bind**、**dnsmasq**、**unbound** 都可以設定 DNS caching server，以下用 unbound 設定 caching nameserver + DNSSEC 作為示範

```bash
[root@server0 ~]# yum -y install unbound

# 1. 設定 listen interface
# 2. 設定 ACL
# 3. 設定 forward zone (找台強壯的 DNS server)
# 4. 設定不要作數位簽章檢查的 domain (optional)
[root@server0 ~]# vim /etc/unbound/unbound.conf
[root@server0 ~]# grep -Ev "^$|#" /etc/unbound/unbound.conf
server:
  .....
	interface: 0.0.0.0
	access-control: 172.25.0.0/24 allow
	domain-insecure: "example.com"
	.....
.....
forward-zone:
 	name: "."
	forward-addr: 172.25.254.254

# 檢查 unbound 設定
[root@server0 ~]# unbound-checkconf
unbound-checkconf: no errors in /etc/unbound/unbound.conf


# 啟動服務
[root@server0 ~]# systemctl enable unbound.service
[root@server0 ~]# systemctl restart unbound.service

[root@server0 ~]# dig @localhost www.google.com

# 把 cache dump 出來
[root@server0 ~]# unbound-control dump_cache
# 清除某一筆 cache
[root@server0 ~]# unbound-control flush www.google.com
# 清除整個 domain 的 cache
[root@server0 ~]# unbound-control flush_zone google.com
```

```bash
[root@desktop0 ~]# dig @server0 www.google.com
```


------------------------------------------------------

5.3 DNS Troubleshooting
=======================

DNS 解析的順序設定在 **/etc/nsswitch.conf** 中：

- `hosts:   files dns`：表示先找 **/etc/hosts** 再找 **/etc/resolve.conf** 中的 DNS server(也可以只留 dns)

DNS 查詢相關工具：

```bash
# 以下三個工具會忽略 /etc/hosts 的設定
$ host www.uuu.com.tw
$ nslookup www.uuu.com.tw
$ dig www.uuu.com.tw

# 以下兩個工具會先找 /etc/hosts 再找 DNS
$ getent www.uuu.com.tw
$ gethostip www.uuu.com.tw
```


```bash
# 強制使用 tcp 協定查詢
$ dig +tcp A example.com
```

### 過濾 spam mail 的方式

- postgrey

- dnsbl



------------------------------------------------------

Lab: Managing DNS For Servers
=============================
