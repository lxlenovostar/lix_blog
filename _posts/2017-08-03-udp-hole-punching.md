---
title:  UDP hole punching
categories: P2P
---

### UDP Hole Punching 
Hole Punching技术是一种借助于公网上的服务器完成NAT穿越的   
解决方案。在基于UDP的Hole Punching场景中，终端A，B分别与   
公网上的服务器S建立UDP连接。当一个终端向服务器S注册时，   
服务器S记录下该终端的两对IP地址和端口信息，为描述方便，    
我们将一对IP地址和端口信息的组合称之为一个endpoint。一个    
endpoint是终端发起与服务器S通信的IP地址和端口；另一个    
endpoint是服务器S观察到的该终端实际与自己通信所用的IP    
地址和端口。我们可以把前一个endpoint看作是终端的私网IP   
地址和私网端口；把后一个endpoint看作是终端的私网IP地址    
和端口经过NAT转换后的公网IP地址和公网端口。服务器S可以    
从终端的注册报文中得到该终端的私网endpoint相关信息，     
可以通过对注册报文的源IP地址和UDP源端口字段获得该终端     
的公网endpoint。如果终端不是位于NAT设备后面，那么采用     
上述方法得到的两个endpoint应该是完全相同的。   
也有一些的NAT设备会扫描UDP报文的数据字段，寻找4字节的    
位域，将看上去很像IP地址的位域，改为与IP头一样的地址。    
为了避免这种行为的NAT设备对UDP报文数据的修改，应用程序    
可以采用直接对IP地址的值进行加密的方式骗过NAT设备的检查。

###  UDP Hole Punching的过程
当终端A希望与终端B建立连接时，Hole punching(也就是P2P    
的连接阶段)过程如下所示：     
1. 终端A最初并不知道如何向B发起连接，于是终端A向服务器   
S发送报文，请求服务器S帮助建立与终端B的UDP连接。

2. 服务器S将含有终端B的公网及私网的endpoint发给终端A，   
同时，服务器S将含有终端A的公网及私网的endpoint的用于    
请求连接的报文也发给B。一旦这些报文都顺利达到，终端A    
与终端B都知道了对方的公网和私网的endpoint。   

3. 当终端A收到服务器S返回的包含终端B的公网和私网的    
endpoint的报文后，终端A开始分别向这些终端B的endpoint    
发送UDP报文，并且终端A会自动“锁定”第一个给出响应的    
终端B的endpoint。同理，当终端B收到服务器S发送的包含    
终端A的公网和私网的endpoint的报文后，也会开始向终端    
A的公网和私网的endpoint发送UDP报文，并且自动锁定第    
一个得到终端A的回应的endpoint。由于终端A与B的互相向    
对方发送UDP报文的操作是异步的，所以终端A与B发送报文    
的时间先后没有严格的时序要求。     

### 终端位于同一个NAT设备后面
两个终端都位于同一个NAT设备后面，位于同一个内网中,是      
最“简单”的一种场景。    
在UDP punching之前的情况如下：
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2017-08-03-udp-hole-punching-1.png)   
如上图，终端A的内网endpoint是10.0.0.1:1000,经过NAT设备     
映射之后的endpoint是100.0.0.1:2000。终端B的内网endpoint      
是11.0.0.1:1000，经过NAT设备映射之后的endpoint是     
100.0.0.1:2010。服务器S的endpoint是111.0.0.1:9000。     
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2017-08-03-udp-hole-punching-2.png)   
如上图A1，终端A向服务器S请求终端B的公网和私网endpoint。       
如上图A3，终端A向终端B的公网endpoint发送报文，当NAT支持    
回环转换(hairpin translation)的情况下，同一个NAT环境下     
终端A和终端B才可以连通。但是终端A与B往对端私网endpoint      
发送的UDP报文是一定可以到达的，无论如何，私网报文采用         
最短转发路径，要比经过NAT转换来的快。终端A与B有很大的      
可能性采用私网的endpoint进行常规的通信。              



### 终端位于不同的NAT设备后面
最普遍的一种情景，两个终端分别位于不同的NAT设备后面，    
分属不同的内网。    
在UDP punching之前的情况如下：
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2017-08-03-udp-hole-punching-4.png)   
如上图，终端A需要通过服务器S的支持下与终端B连接。终端A的    
内网endpoint是10.0.0.1:1000,经过NAT A设备映射之后的endpoint   
是100.0.0.1:2000。终端B的内网endpoint是11.0.0.1:1000，经过    
NAT B设备映射之后的endpoint是101.0.0.1:2010。服务器S的     
endpoint是111.0.0.1:9000。 
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2017-08-03-udp-hole-punching-5.png)   
如上图的A1,终端A向服务器S发出请求报文与终端B进行连接。                 
如A2，服务器S将终端B的公网和私网endpoint信息发给终端A。           
如图A3，终端A向终端B的公网endpoint发送报文，这个时候            
终端A发给终端B的公网endpoint的报文在终端B向终端A发送            
报文之前到达终端B的NAT，终端B的NAT会认为终端A发过来的            
报文是未经授权的公网报文，会丢弃掉该报文。因此只有在             
B3，终端B向终端A的公网endpoint发送报文之后，A3这条通             
道才会畅通。如A4，终端A向终端B的私网endpoint发送报文，           
由于终端A和终端B不在一个内网，此通道将不可达。如B2,             
服务器S将终端A的公网和私网的endpoint反馈给终端B,如B3,            
终端B向终端A的公网endpoint发送报文，由于A3的时候建立              
终端A通过NAT A到NAT B的映射，此时终端B发往终端A的报文            
不会被NAT A丢失。如B4,和A4一样由于不在一个内网将发送              
报文失败。       



### 终端位于多层的NAT设备后面
终端位于两层NAT设备之后，通常最上层的NAT是由ISP提供，    
第二层的NAT是家用的NAT路由器之类的设备。    
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2017-08-03-udp-hole-punching-7.png)     
如上图所示，假定NAT C是由ISP(Internet Service Provider)      
提供的工业级的NAT设备，NAT C提供将多个下属的用户NAT或      
用户节点映射到有限的几个公网IP的服务，NAT A和NAT B作为      
NAT C的内网节点将把用户的家庭网络或内部网络接入NAT C的       
内网，然后用户的内部网络就可以经由NAT C访问公网了。从这        
种拓扑结构上来看，只有服务器S与NAT C是真正拥有公网可路由        
IP地址的设备，而NAT A和NAT B所使用的“公网”IP地址，实际上        
是由ISP服务提供商设定的（相对于NAT C而言）私网地址，同理       
隶属于NAT A与NAT B的客户端，相对与NAT A，NAT B而言，它们        
处于NAT A，NAT B的内网，以此类推，客户端可以放到多层NAT         
设备后面。客户端A和客户端B发起对服务器S的连接的时候，就会         
依次在NAT A和NAT B上建立向外的映射关系，而NAT A、NAT B要          
联入公网的时候，会在NAT C上再建立向外的映射关系。
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2017-08-03-udp-hole-punching-8.png)     
现在假定终端A和B希望通过UDP Hole punching完成两个终端间的      
P2P直连。最优化的路由策略是终端A向终端B的“伪公网”IP上发送       
数据包，即ISP服务提供商指定的私网IP，NAT B的“伪”公网        
endpoint，101.0.0.1:2000。由于从服务器S的角度只能观察到      
真正的公网地址，也就是NAT A，NAT B在NAT C建立的映射时的       
真正的公网endpoint：110.0.0.1:3000以及110.0.0.1:3010，       
所以非常不幸，终端A与B是无法通过服务器S知道这些“伪”公网的       
endpoint。而且即使终端A和B通过某种手段可以得到NAT A和      
NAT B的“伪”公网endpoint，我们仍然不建议采用上述的      
“最优化”的打洞方式，这是因为这些地址是由ISP服务提供商      
提供的，或许会存在与客户端本身所在的私网地址重复的      
可能性。（例如：NAT A的私网IP地址域恰好与NAT A在      
NAT C的“伪”公网IP地址域重复，这样就会导致打洞报文无法      
发出的问题）。        
因此终端别无选择，只能使用由公网服务器S观察到的终端       
A，B的公网地址和端口进行“打洞”操作，用于“打洞”的报文将        
由NAT C进行转发，这里NAT C是否支持回环转换非常重要，否则       
数据包将无法由NAT C转发给NAT A和NAT B，进而无法到达终端     
A和B。当终端A向B的公网地址（110.0.0.1:3010）发送UDP数据      
报文的时候，NAT A首先把数据包的源地址由A的私网endpoint       
（10.0.0.1:1000)转换为“伪”公网endpoint（100.0.0.1:2000）       
，现在报文到了NAT C，NAT C应该可以识别出来该数据包是要       
发往自身转换过的公网endpoint,如果NAT C可以给出“合理”        
响应的话，NAT C将把该数据包的源endpoint改为110.0.0.1:3000       
，目的endpoint改为101.0.0.1:2000，即NAT B的“伪”公网       
endpoint，NAT B最后会将收到的报文发往终端B。同样，       
由终端B发往A的数据包也会经过类似的过程。也有很多NAT设备                 
不支持类似这样的回环转换，但是已经有越来越多的NAT设备        
生产厂商开始加入对该转换的支持。      




