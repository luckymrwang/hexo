title: iptables中-j选项与-g选项的区别
date: 2022-04-19 14:05:06
tags: [Linux]
---

## 手册

```js
# man iptables
 
...
 
-j, --jump target
              This  specifies  the  target  of  the  rule; i.e., what to do if the packet matches it.  The target can be a user-
              defined chain (other than the one this rule is in), one of the special builtin targets which decide  the  fate  of
              the  packet  immediately,  or an extension (see EXTENSIONS below).  If this option is omitted in a rule (and -g is
              not used), then matching the rule will have no effect on the packet's fate, but the counters on the rule  will  be
              incremented.
 
-g, --goto chain
              This specifies that the processing should continue in a user specified chain. Unlike the --jump option return will
              not continue processing in this chain but instead in the chain that called us via --jump.
```
<!-- more -->

- `-j` 选项指定规则的目标

	目标可以是用户自定义链；内建目标；或扩展

- `-g` 选项将规则重定向到一个用户自定义链中

	与 `-j` 选项不同，从自定义链中返回时，是返回到调用 `-g` 选项上层的那一个 `-j` 链中
	
## 示例

使用iptables -S命令查看Fedora18中默认的防火墙配置

```js
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -j INPUT_direct
-A INPUT -j INPUT_ZONES
-A INPUT -p icmp -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
 
...
 
-A INPUT_ZONES -i p5p1 -g IN_ZONE_public
-A INPUT_ZONES -g IN_ZONE_public
 
...
 
-A IN_ZONE_public -j IN_ZONE_public_deny
-A IN_ZONE_public -j IN_ZONE_public_allow
```

- `-g` 选项

当报文由规则 `-A INPUT_ZONES -i p5p1 -g IN_ZONE_public` 将接口 `p5p1` 报文重定向到 `IN_ZONE_public` 时

从 `IN_ZONE_public` 返回后

不会继续进入其下一条规则 `-A INPUT_ZONES -g IN_ZONE_public`

而是直接返回到上层的 `-A INPUT -j INPUT_ZONES`

然后继续规则 `-A INPUT -p icmp -j ACCEPT`

## 计数器

使用 `iptables -L -v` 查看报文计数器

```js
# iptables -L -v
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
 2447  173K ACCEPT     all  --  any    any     anywhere             anywhere             ctstate RELATED,ESTABLISHED
    1    80 ACCEPT     all  --  lo     any     anywhere             anywhere            
  404 82313 INPUT_direct  all  --  any    any     anywhere             anywhere            
  404 82313 INPUT_ZONES  all  --  any    any     anywhere             anywhere            
    0     0 ACCEPT     icmp --  any    any     anywhere             anywhere            
  375 78428 REJECT     all  --  any    any     anywhere             anywhere             reject-with icmp-host-prohibited
 
...
 
Chain INPUT_ZONES (1 references)
 pkts bytes target     prot opt in     out     source               destination         
  375 78428 IN_ZONE_public  all  --  p5p1   any     anywhere             anywhere            [goto] 
    0     0 IN_ZONE_public  all  --  +      any     anywhere             anywhere            [goto]
 
...
 
Chain IN_ZONE_public (2 references)
 pkts bytes target     prot opt in     out     source               destination         
  375 78428 IN_ZONE_public_deny  all  --  any    any     anywhere             anywhere            
  375 78428 IN_ZONE_public_allow  all  --  any    any     anywhere             anywhere            
Chain IN_ZONE_public_allow (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     tcp  --  any    any     anywhere             anywhere             tcp dpt:ssh ctstate NEW
    0     0 ACCEPT     udp  --  any    any     anywhere             224.0.0.251          udp dpt:mdns ctstate NEW
Chain IN_ZONE_public_deny (1 references)
 pkts bytes target     prot opt in     out     source               destination 
```

从计数器可以看出链 `INPUT_ZONES` 中走了第一个 `goto` 的不会进入第二个 `goto`

而链 `IN_ZONE_public` 中第一个 `deny` 没有处理

其计数器与 `allow` 的相同


