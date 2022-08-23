title: 设备收发包之netif_receive_skb
date: 2022-06-20 23:04:18
tags: [Linux]
---

在设备驱动收包之后，会通过netif_receive_skb将收取的包，按照注册的协议回调，传递到上层进行处理；

<!--more-->

```c
/* 将skb传递到上层 */
static int __netif_receive_skb_core(struct sk_buff *skb, bool pfmemalloc)
{
    struct packet_type *ptype, *pt_prev;
    rx_handler_func_t *rx_handler;
    struct net_device *orig_dev;
    bool deliver_exact = false;
    int ret = NET_RX_DROP;
    __be16 type;

    /* 记录收包时间，netdev_tstamp_prequeue为0，表示可能有包延迟 */
    net_timestamp_check(!netdev_tstamp_prequeue, skb);

    trace_netif_receive_skb(skb);

    /* 记录收包设备 */
    orig_dev = skb->dev;

    /* 重置各层头部 */
    skb_reset_network_header(skb);
    if (!skb_transport_header_was_set(skb))
        skb_reset_transport_header(skb);
    skb_reset_mac_len(skb);

    /*
        留下一个节点，最后一次向上层传递时，
        不需要在inc引用，回调中会free
        这样相当于少调用了一次free
    */
    pt_prev = NULL;

another_round:

    /* 接收设备索引号 */
    skb->skb_iif = skb->dev->ifindex;

    /* 处理包数统计 */
    __this_cpu_inc(softnet_data.processed);

    /* vlan包，则去掉vlan头 */
    if (skb->protocol == cpu_to_be16(ETH_P_8021Q) ||
        skb->protocol == cpu_to_be16(ETH_P_8021AD)) {

        /*
            这里改了三层协议，protocol指向ip等
            another_round不会再走这里
        */
        skb = skb_vlan_untag(skb);
        if (unlikely(!skb))
            goto out;
    }

    /* 不对数据包进行分类 */
    if (skb_skip_tc_classify(skb))
        goto skip_classify;

    /* prmemalloc */
    if (pfmemalloc)
        goto skip_taps;


    /* 下面两个是未(指定)设备的所有协议传递的上层传递 */


    /* 如抓包程序未指定设备 */
    /* 进行未指定设备的全局链表对应协议的skb上层传递 */
    list_for_each_entry_rcu(ptype, &ptype_all, list) {
        if (pt_prev)
            ret = deliver_skb(skb, pt_prev, orig_dev);
        pt_prev = ptype;
    }

    /* 如抓包程序指定了设备 */
    /* 进行指定设备的协议链表的skb上层传递 */
    list_for_each_entry_rcu(ptype, &skb->dev->ptype_all, list) {
        if (pt_prev)
            ret = deliver_skb(skb, pt_prev, orig_dev);
        pt_prev = ptype;
    }

skip_taps:
#ifdef CONFIG_NET_INGRESS
    if (static_key_false(&ingress_needed)) {
        skb = sch_handle_ingress(skb, &pt_prev, &ret, orig_dev);
        if (!skb)
            goto out;

        if (nf_ingress(skb, &pt_prev, &ret, orig_dev) < 0)
            goto out;
    }
#endif
    skb_reset_tc(skb);
skip_classify:

    /* 不支持使用pfmemalloc */
    if (pfmemalloc && !skb_pfmemalloc_protocol(skb))
        goto drop;

    /* 如果是vlan包 */
    if (skb_vlan_tag_present(skb)) {
        /* 处理prev */
        if (pt_prev) {
            ret = deliver_skb(skb, pt_prev, orig_dev);
            pt_prev = NULL;
        }

        /* 根据实际的vlan设备调整信息，再走一遍 */
        if (vlan_do_receive(&skb))
            goto another_round;
        else if (unlikely(!skb))
            goto out;
    }

    /* 如果有注册handler，那么调用，比如网桥模块 */
    rx_handler = rcu_dereference(skb->dev->rx_handler);
    if (rx_handler) {
        if (pt_prev) {
            ret = deliver_skb(skb, pt_prev, orig_dev);
            pt_prev = NULL;
        }
        switch (rx_handler(&skb)) {
            /* 已处理，无需进一步处理 */
        case RX_HANDLER_CONSUMED:
            ret = NET_RX_SUCCESS;
            goto out;
            /* 修改了skb->dev，在处理一次 */
        case RX_HANDLER_ANOTHER:
            goto another_round;
            /* 精确传递到ptype->dev == skb->dev */
        case RX_HANDLER_EXACT:
            deliver_exact = true;
            /* 正常传递即可 */
        case RX_HANDLER_PASS:
            break;
        default:
            BUG();
        }
    }

    /* 还有vlan标记，说明找不到vlanid对应的设备 */
    if (unlikely(skb_vlan_tag_present(skb))) {
        /* 存在vlanid，则判定是到其他设备的包 */
        if (skb_vlan_tag_get_id(skb))
            skb->pkt_type = PACKET_OTHERHOST;
        /* Note: we might in the future use prio bits
         * and set skb->priority like in vlan_do_receive()
         * For the time being, just ignore Priority Code Point
         */
        skb->vlan_tci = 0;
    }

    /* 设置三层协议，下面提交都是按照三层协议提交的 */
    type = skb->protocol;

    /* deliver only exact match when indicated */
    /* 未设置精确发送，则向未指定设备的指定协议全局发送一份 */
    if (likely(!deliver_exact)) {
        deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
                       &ptype_base[ntohs(type) &
                           PTYPE_HASH_MASK]);
    }

    /* 指定设备的，向原设备上层传递  */
    deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
                   &orig_dev->ptype_specific);

    /*  当前设备与原设备不同，向当前设备传递 */
    if (unlikely(skb->dev != orig_dev)) {
        deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
                       &skb->dev->ptype_specific);
    }

    if (pt_prev) {
        if (unlikely(skb_orphan_frags(skb, GFP_ATOMIC)))
            goto drop;
        else
            /*
                使用pt_prev这里就不需要deliver_skb来inc应用数了
                func执行内部会free，减少了一次skb_free
            */
            /* 传递到上层*/
            ret = pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
    } else {
drop:
        if (!deliver_exact)
            atomic_long_inc(&skb->dev->rx_dropped);
        else
            atomic_long_inc(&skb->dev->rx_nohandler);
        kfree_skb(skb);
        /* Jamal, now you will not able to escape explaining
         * me how you were going to use this. :-)
         */
        ret = NET_RX_DROP;
    }

out:
    return ret;
}
```
