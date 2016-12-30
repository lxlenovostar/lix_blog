---
title: 自定义TCP包 
---

# 在无套接字情况下，发送TCP包 

## 目标
1. 不使用套接字进行数据包传输。
2. 在源端机器的内核态，组建一个TCP包并发送到目标机器。
3. 在目标机器的内核态对自定义TCP包进行处理。

### TCP/IP 参考模型
因特网协议栈中的层：
1. 应用层 
2. 传输层 
3. 网络层
4. 接口层

如下图中的*Data Flow*所示:   
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2016-10-20-IP_stack_connections.svg.png)   
当我们发送一个TCP包的时候，我们应该从最上层的*Application*层准备需要传输的信息，然后依次填充*Transport*、*Internet* 和 *Link* 的协议头部。

### 发送端实现
#### 发送端整体流程
```
243     /* 第一步 分配和设置sk_buff. */   
244     skb = alloc_set_skb();  
245     if (!skb)             
246         return -1;        
247                           
248     /* 第二步 拷贝TCP负载 */
249     err = copy_md5sum(skb);
250     if (err != 0) {       
251         kfree_skb(skb);   
252         return err;       
253     }
254                           
255     /* 第三步 组建TCP头部 */
256     err = build_tcphdr(skb);
257     if (err != 0) {       
258         kfree_skb(skb);   
259         return err;       
260     }
261    
262     /* 第四步 组建IP头部 */
263     err = build_iphdr(skb);
264     if (err != 0) {       
265         kfree_skb(skb);   
266         return err;       
267     }
268 
269     /* 第五步 计算TCP头和IP头中的checksum */  
270     iph = ip_hdr(skb);    
271     skb->ip_summed = CHECKSUM_NONE;   
272     skbcsum(skb, iph);
273 
274     /* 第六步 组建网络接口层的头部 */
275     err = build_ethhdr(skb); 
276     if (err != 0) {
277         return err;
278     }
279 
280     /* 第七步 发送SKB */
281     err = dev_queue_xmit(skb);
282 
283     return err;
```

#### 分配SKB
内核中使用套接字缓冲区(struct sk_buff)表示协议栈中的数据包。    
```
216 struct sk_buff * alloc_set_skb(void)
217 {
218     struct sk_buff *skb;
219
220     /* At least 32-bit aligned.  */
221     int size = ALIGN(sizeof(struct ethhdr), 4) + ALIGN(sizeof(struct iphdr), 4) + ALIGN(sizeof(struct tcphdr), 4) + ALIGN    (MD5LEN, 4);
222 
223     skb = alloc_skb(size, GFP_ATOMIC);
224     if (skb == NULL) {
225         printk(KERN_ERR"alloc skb failed.");
226         return NULL;
227     }
228 
229     /* Reserve space for headers and prepare control bits. */
230     skb_reserve(skb, size);
231 
232     return skb;
233 }
```
第221行调整size为了四字节边界对齐。第223行分配一个新的sk_buff实例。第230行skb_reserve通过移动skb的data和tail指针, 来调整skb的headroom。   

#### 拷贝传输信息
这里直接在内核态中将要传输信息拷贝到skb, 而非像套接字编程中将数据从用户态拷贝到内核态中。   
```
205 int copy_md5sum(struct sk_buff *skb)
206 {
207     uint8_t md5_result[MD5LEN] = {1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16};
208 
209     if (skb_tailroom(skb) < MD5LEN)
210         return -1;
211     else {
212         memcpy(skb_push(skb, MD5LEN), md5_result, MD5LEN);
213     }
214 
215     return 0;
216 }
```
这里我们需要将一个伪MD5值传递到对端。第209行用于判断内存空间是否可以容纳MD5值，第212行的skb_push用于向前移动skb的data指针，   
此时data和tail之间的内存空间就可以存放传输信息。

#### 组装TCP头部 
TCP的头部定义如下:   
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2016-10-20-tcp-header.jpg)   
```
184 int build_tcphdr(struct sk_buff *skb)
185 {
186     struct tcphdr *th;
187 
188     skb_push(skb, ALIGN(sizeof(struct tcphdr), 4));
189     skb_reset_transport_header(skb);
190 
191     /* Build TCP header and checksum it. */
192     th = tcp_hdr(skb);
193     th->source      = htons(SOURCE);
194     th->dest        = htons(DEST);
195     th->seq         = htonl(123);
196     th->ack_seq     = 0;
197     *(((__be16 *)th) + 6)   = htons(((sizeof(struct tcphdr) >> 2) << 12) | TCPCB_FLAG_FIN);
198     th->window = htons(560);
199     th->check = 0;
200     th->urg_ptr = 0;
201 
202     return 0;
203 }
```
第188行设置TCP头部所需要的内存空间，第189行用于设置skb的transport_header指针，这样做是为了方便后面的tcp_hdr函数使用。   
第193和194行用于设置端口，第195和196行用于设置序号和确认号，第197行用于设置TCP头长度和FIN标志位，    
第198行用于设置窗口值，第199行校验和先设置为0。

#### 组装IP头部 
IP的头部定义如下:   
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2016-10-20-ip-header.jpg)   
```
161 int build_iphdr(struct sk_buff *skb)
162 {
163     struct iphdr *iph;
164     
165     skb_push(skb, sizeof(struct iphdr));
166     skb_reset_network_header(skb);
167     
168     iph = ip_hdr(skb);
169     *((__be16 *)iph) = htons((4 << 12) | (5 << 8) | (RT_TOS(20) & 0xff));
170     iph->tot_len = htons(skb->len);
171     iph->frag_off = htons(IP_DF);
172     iph->ttl      = 64;
173     iph->protocol = IPPROTO_TCP;
174     iph->saddr    = in_aton(SOU_IP);
175     iph->daddr    = in_aton(DST_IP);
176     
177     return 0;
178 }
```
第165行设置IP头部所需要的内存空间，第169用于设置IP的版本、IHL和区分服务。第170行的tot_len为总长度。   
第171行设置IP_DF表示没有IP分段。第171用于设置生存期，第172行用于设置协议，IP的头检验和暂时不设置。   
第174和175行用于设置源地址和目标地址。   

#### 组建网络接口层的头部 
在网络接口层主要是获得本机发包网卡的MAC地址和网关的MAC地址。
```
124 int build_ethhdr(struct sk_buff *skb)
125 {
126     struct ethhdr *eth;
127     struct net_device *dev;
128     int err;
129    
130     dev = dev_get_by_name(&init_net, SOU_DEVICE);
131     if (!dev) {
132         printk(KERN_ERR"get device failed.");
133         return -3;
134     }
135 
136     skb->dev = dev;
137     skb->pkt_type  = PACKET_OUTGOING;
138     skb->local_df  = 1;
139     skb->protocol = htons(ETH_P_IP);
140 
141     eth = (struct ethhdr *)skb_push(skb, ETH_HLEN);
142     skb_reset_mac_header(skb);
143 
144     memcpy(eth->h_source, dev->dev_addr, ETH_ALEN);
145    
146     gate_addr = in_aton(GATE_IP);
147     err = get_mac(eth->h_dest, gate_addr, RT_TOS(20), skb);
148     if (err != 1) {
149         kfree_skb(skb);
150         if (net_ratelimit())
151             printk(KERN_ERR"get device mac failed when send packets.err:%d", err);
152             return err;
153         }
154 
155     return 0;
156 }
```
第130行获得网卡SOU_DEVICE在内核中的指针，用于获得MAC地址。第147行用于获得网关的MAC地址。   
第138行的local_df是指"don't fragment"，设置为1代表不会被分段。

### 目的端实现
因为没有在目地端建立socket，我们在netfiter中对包进行处理，netfilter中hook点的位置如下：
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2016-10-20-ip-hook.jpg)   
```
60 struct nf_hook_ops nf_in_ops = {
61     .list       = { NULL, NULL},
62     .pf     = PF_INET,
63 #if LINUX_VERSION_CODE >= KERNEL_VERSION(2, 6, 32)
64     .hook           = nf_in,
65     //.hooknum        = NF_INET_PRE_ROUTING,
66     .hooknum        = NF_INET_LOCAL_IN,
67 #else
68     .hook           = nf_in_p,
69     //.hooknum        = NF_IP_PRE_ROUTING,
70     .hooknum        = NF_IP_LOCAL_IN,
71 #endif
72     .priority       = NF_IP_PRI_FIRST,
73 };
```
hook函数如下：   
```
10 static unsigned int nf_in(
11 #if LINUX_VERSION_CODE >= KERNEL_VERSION(3, 13, 0)
12         const struct nf_hook_ops *ops,
13 #else
14         unsigned int hooknum,
15 #endif
16         struct sk_buff *skb,
17         const struct net_device *in,
18         const struct net_device *out,
19         int (*okfn)(struct sk_buff *))
20 {
21     unsigned short sport, dport;
22     struct iphdr *iph;
23     struct tcphdr *tcph;
24     char *tcp_payload = NULL;
25     size_t tcp_payload_len = 0;
26     int i;
27 
28     iph = ip_hdr(skb);
29     if (iph->protocol == IPPROTO_TCP) {
30         tcph = (struct tcphdr *)(skb->data + (iph->ihl << 2));
31 
32         sport = tcph->source;
33         dport = tcph->dest;
34         if (ntohs(sport) == 6880 && ntohs(dport) == 6880) {
35             printk(KERN_INFO "find a new packet");
36             tcp_payload = (char *)((unsigned char *)tcph + (tcph->doff << 2));
37             tcp_payload_len = ntohs(iph->tot_len) - (iph->ihl << 2) - (tcph->doff << 2);
38 
39             if (tcp_payload_len == 16)
40                 for (i = 0; i < tcp_payload_len; ++i)
41                     printk("%d:", tcp_payload[i]);
42 
43             return NF_DROP;
44         }
45     }
46     return NF_ACCEPT;
47 }
```
第29行到43行，我们对我们自己制造的TCP包进行处理。

### 结论
1. 使用此种方法对于传输少量数据可行，但是由于丢包的问题，需要增加ACK和重传机制。
2. 改为per-cpu模式话，可以用于局域网中测试网卡性能。  

### github地址
[github地址](https://github.com/lxlenovostar/kernel_test/tree/master/model/build_receive_tcp)
