---
layout: default
title: 结构体 skb_shared_info  
---
## skb_shared_info 

### skb_shared_info 的定义
```
/* This data is invariant across clones and lives at
 * the end of the header data, ie. at skb->end.
 */
struct skb_shared_info {
    atomic_t    dataref;
    unsigned short  nr_frags;
    unsigned short  gso_size;
    /* Warning: this field is not always filled in (UFO)! */
    unsigned short  gso_segs;
    unsigned short  gso_type;
    __be32          ip6_frag_id;
    struct sk_buff  *frag_list;
    skb_frag_t  frags[MAX_SKB_FRAGS];
};
```
