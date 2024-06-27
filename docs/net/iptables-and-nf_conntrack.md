# iptables 和 nf_conntrack 的纠缠

## 楔子
部署 Nginx 服务器时，为了业务安全，通常会开启 iptables 设置防火墙规则，以提高业务安全性。

故事，就从这里开始。

## 初见
压测，是任何系统上线前必不可少的检测步骤。压测刚开始时，请求量不高，服务器的 CPU、内存等资源的使用率、负载指标，以及请求失败率、延时等业务指标一切正常。但随着压测进行，请求量上涨，此时观察到请求失败率出现突增。

检查服务器资源指标，使用率、负载依然保持低水位，说明不是资源瓶颈导致的请求超时。对比服务端 access 日志和压测客户端日志，发现超时请求在服务端日志中没有记录。通过 dmesg 命令检查系统日志，发现有如下记录

    nf_conntrack: table full, dropping packet 

于是，问题定位清楚了：
1. nf_conntrack 是内核中的连接追踪模块，用于记录每个连接的状态信息
2. iptables 规则中使用了 -state 参数，对所有入包都进行了状态记录
```bash
# iptables -L -v
ACCEPT     all  --  any    any     anywhere             anywhere            state RELATED,ESTABLISHED
```
3. nf_conntrack 是一个开链的 hash 表，每个 hash 节点存储一个双链表。相同 hash 值的连接放到同一个双链表中，数据结构见示例如下
![image](https://github.com/alucard-Arutoria/alucard-arutoria.github.io/assets/51264332/e7126b69-59f6-4da1-b3db-95544e777dd5)
4. nf_conntrack 存在最大包数限制，默认值受到资源和操作系统版本的影响。取值规则如下：
    1. 名词解释
        ```
        系统内存
        指系统能识别的内存总量，也就是通过free命令查看到的总内存。下述最大内存数

        nf_conntrack_buckets
        hash 表大小（即节点数）。

        ARCH
        指操作系统位数，取值32或64
        ```
    2. CentOS6.10 x86_64
        ```
        # 系统内存小于 1GiB 时
        nf_conntrack_buckets = 最大内存数/16384/(ARCH/32)
        nf_conntrack_max = nf_conntrack_buckets * 4

        # 系统内存大于 1GiB 时
        nf_conntrack_buckets = 16384
        nf_conntrack_max = 65535
        ```
    3. CentOS7.9 x86_64
        ```
        # 系统内存小于 1GiB 时
        nf_conntrack_buckets = 最大内存数/16384/(ARCH/32)
        nf_conntrack_max = nf_conntrack_buckets * 4

        # 系统内存大于 1GiB 时
        nf_conntrack_buckets = 16384
        nf_conntrack_max = 65535

        # 系统内存大于 4GiB 时
        nf_conntrack_buckets = 65535
        nf_conntrack_max = 262144
        ```
5. 当连接数大于 nf_conntrack_max 时，为保证系统安全，会进行随机丢包。
6. 需要说明的是，当服务端主动断开时，连接会处于 time_wait 状态，并存活 2MLS，此时连接依然会占用 conntrack 空间。

**参考**

[你真的了解nf_conntrack么？](https://blog.51cto.com/u_15293891/3290232)

[nf_conntrack 调优](https://www.haxi.cc/archives/nf_conntrack%E8%B0%83%E4%BC%98.html)

## 再相逢
