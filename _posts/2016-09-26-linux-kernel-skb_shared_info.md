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
用于skb clone的字段：   
**dataref** : This field contains the reference count for this skb. It is incremented each time the buffer is cloned.   
用于TCP 分段的字段：   
**nr_frags** : the number of fragments in this packet. This field is used by TCP segmentation.   
**frags** : the array of page table entries. Each entry is actually a TCP segment.   
用于IP 分片的字段：   
**frag_list** : This field points to the list of fragments for this packet if it is fragmented.      
# TODO GRO   


### TCP分段和IP分片之间的关系
 


