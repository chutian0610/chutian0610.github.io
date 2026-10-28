# linux命令dig

Linux dig命令用来从 DNS 域名服务器查询主机地址信息。
<!--more-->
## tldr 

```sh
## 简单查询
dig example.com

## 精简返回结果
dig +short example.com

## 使用指定的 DNS server进行查询
dig @8.8.8.8 example.com

## 仅展示查询的answer部分
dig +noquestion +noauthority +noadditional +noanswer example.com

## 查询指定的 DNS 类型(例如，MX = 邮箱)
dig example.com MX

## 获取IPv4 A and IPv6 AAAA 记录
dig example.com A
dig example.com AAAA

## 反向解析 IP 地址对应的域名
dig -x 8.8.8.8

## 跟踪整个查询过程
dig +trace example.com
```

## usage

以 baidu.com 为例展示下使用:

```
$ dig baidu.com                    

┌──────────; <<>> DiG 9.10.6 <<>> baidu.com
section 1
└──────────;; global options: +cmd
┌──────────;; Got answer:
section 2  ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28214
└──────────;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

┌──────────;; OPT PSEUDOSECTION:
│          ; EDNS: version: 0, flags:; udp: 4000
section 3  ;; QUESTION SECTION:
└──────────;baidu.com.                     IN      A

┌──────────;; ANSWER SECTION:
section 4  baidu.com.              315     IN      A       110.242.68.66
└──────────baidu.com.              315     IN      A       39.156.66.10

┌──────────;; Query time: 54 msec
│          ;; SERVER: 172.23.128.2#53(172.23.128.2)
section 5  ;; WHEN: Thu May 29 10:22:06 CST 2025
└──────────;; MSG SIZE  rcvd: 70
```

- 第一部分显示 dig 命令的版本和输入的参数。
- 第二部分显示服务返回的一些技术详情，比较重要的是 status。如果 status 的值为 NOERROR 则说明本次查询成功结束。
- 第三部分中的 "QUESTION SECTION" 显示我们要查询的域名。
- 第四部分的 "ANSWER SECTION" 是查询到的结果。
- 第五部分则是本次查询的一些统计信息，比如用了多长时间，查询了哪个 DNS 服务器，在什么时间进行的查询等等。

默认情况下 dig 命令查询 A 记录，上面显示的 A 即说明查询的记录类型为 A 记录。

| 类型 | 目的 |
|---|---|
| A | 地址记录，用来指定域名的 IPv4 地址，如果需要将域名指向一个 IP 地址，就需要添加 A 记录。 |
| AAAA | 用来指定主机名(或域名)对应的 IPv6 地址记录。 |
| CNAME | 如果需要将域名指向另一个域名，再由另一个域名提供 ip 地址，就需要添加 CNAME 记录。 |
| MX | 如果需要设置邮箱，让邮箱能够收到邮件，需要添加 MX 记录。 |
| NS | 域名服务器记录，如果需要把子域名交给其他 DNS 服务器解析，就需要添加 NS 记录。 |
| SOA | SOA 这种记录是所有区域性文件中的强制性记录。它必须是一个文件中的第一个记录。 |
| TXT | 可以写任何东西，长度限制为 255。绝大多数的 TXT记录是用来做 SPF 记录(反垃圾邮件)。 |

