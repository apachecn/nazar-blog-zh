# 具有绑定视图的 Intranet DNS 解析

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/Intranet-DNS-Resolution-with-BIND-Views>

有时，我会为 Oopsilon 网站写一篇文章。在写文章的时候，我喜欢检查它在网站上的显示是否正确，这包括在浏览器中加载它。这完全没问题，除了我的工作计算机和 web 服务器在同一个网络上，而且这个网络是内部的。

任何来自网络外部的人(换句话说，在我家外面)都可以毫无问题地查看这篇文章:向 Oopsilon 发出一个请求，它(目前)解析为`87.194.101.173`，并向该 IP 地址发出一个连接请求。我的防火墙将其转换为 web 服务器的内部 IP 地址`192.168.0.1`，并双向维护这种转换。

从网络内部来说，又是另外一个故事了。Oopsilon 像以前一样解析为`87.194.101.173`，但是当防火墙收到连接请求时，它会看到一个从内部网络到外部世界的连接，并立即返回网络。结果，防火墙拒绝连接请求，我最终无法看到我的文章。

### 标准绑定配置

网络内部的问题是由解析到外部 IP 的环路引起的。这是因为 BIND 配置了简单的 DNS 区域，如下所示:

#### oopsilon.zone:外部区域文件

```
IN      SOA     adhocbox.oopsilon.com. tf.oopsilon.com.  (
                                      2008042701 ; Serial
                                      28800      ; Refresh
                                      14400      ; Retry
                                      604800     ; Expire
                                      86400 )    ; Minimum

                        NS      adhocbox.oopsilon.com.
                        MX 10   oopsilon.com.

oopsilon.com.   IN      A       87.194.101.173
adhocbox        IN      CNAME   oopsilon.com.
www             IN      CNAME   oopsilon.com.
```

然后，BIND 被告知使用该区域来处理与相关域相关的请求，如下所示:

#### named.conf:绑定主配置

```
zone "oopsilon.com" IN {
        type master;
        file "oopsilon.zone";
        allow-update { 88.192.91.15; };
        notify yes;
};
```

上面的配置表明，域的任何请求都将由给定的区域文件提供服务。这包括来自局域网内部的请求，这些请求应该解析为 web 服务器的局域网地址。这可以通过使用两个而不是一个区域文件来解决。

### 特定于视图的绑定配置

对于外部请求，上面的区域文件就足够了:服务于外部 IP 是这些客户端所期望的。对于内部请求，可以使用单独的区域:

#### oopsilon.zone.int:内部区域文件

```
IN      SOA     adhocbox.oopsilon.com. tf.oopsilon.com.  (
                                      2008042701 ; Serial
                                      28800      ; Refresh
                                      14400      ; Retry
                                      604800     ; Expire
                                      86400 )    ; Minimum

                        NS      adhocbox.oopsilon.com.
                        MX 10   oopsilon.com.

oopsilon.com.   IN      A       192.168.0.1
adhocbox        IN      CNAME   oopsilon.com.
www             IN      CNAME   oopsilon.com.
```

为了在两个区域文件之间进行选择，可以在配置文件中设置一系列“视图”，其中每个视图都与一系列 IP 地址相匹配。这是通过在视图块中嵌套分区来实现的:

#### named.conf:带有视图的主配置

```
view "internal" {
        match-clients { 192.168.0.0/24; };
        zone "oopsilon.com" IN {
                type master;
                file "oopsilon.zone.int";
                allow-update { none; };
                notify no;
        };
};

view "external" {
        match-clients { any; };
        zone "oopsilon.com" IN {
                type master;
                file "oopsilon.zone";
                allow-update { 88.192.91.15; };
                notify yes;
        };
};
```

在上面的配置中，有两个视图:`internal`用于来自内部网络的客户端(`192.168.0.x`)，以及`external`用于其他所有人。一个视图中可以有任意数量的区域，但是在本例中，我只需要每个区域中有一个区域。

一旦这种配置到位，它的操作是自动的:局域网中的任何人都将收到网络服务器的局域网 IP，并能够浏览网站。网络之外的客户端将收到一个外部 IP，并且也能够看到网站。每个人都赢了。

*版权所有伊姆兰·纳扎尔<tf@oopsilon.com2008 年*

文章日期:2008 年 6 月 2 日