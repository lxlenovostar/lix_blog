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

### skb_shared_info 内存中的位置
This structure is placed at the end of the attached data buffer and pointed to by the **end** field    
in the *socket buffer structure(struct sk_buff)*, which points to the end of the data portion of the packet. 
```
 430 #ifdef NET_SKBUFF_DATA_USES_OFFSET
 431 static inline unsigned char *skb_end_pointer(const struct sk_buff *skb)
 432 {
 433     return skb->head + skb->end;
 434 }
 435 #else
 436 static inline unsigned char *skb_end_pointer(const struct sk_buff *skb)
 437 {
 438     return skb->end;
 439 }
 440 #endif
 441 
 442 /* Internal */
 443 #define skb_shinfo(SKB) ((struct skb_shared_info *)(skb_end_pointer(SKB)))
```
### skb_shared_info 的作用
1. IP分片
2. TCP分段
3. 记录SKB的clone次数 
4. TODO GSO

### TCP分段和IP分片之间的关系
The handling of TCP segments is more efficient than IP
fragments. IP fragmentation is not quite as common as it was in earlier days of the Internet.
Fragmentation is used when a network segment has a smaller MTU than the packet size. IP
fragmentation is necessary if the MTU of the outgoing device is smaller than the packet size. See
Chapter 9 for more details about IP fragmentation. TCP segmentation, however, is far more
common because it is the underlying mechanism for the transport of streaming data that occurs
in most network traffic. 

# TODO TCP分段的实现   
# TODO GRO   

