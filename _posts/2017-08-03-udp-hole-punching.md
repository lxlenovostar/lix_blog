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
##TODO

### 终端位于不同的NAT设备后面
最普遍的一种情景，两个终端分别位于不同的NAT设备后面，    
分属不同的内网。    

### 终端位于多层的NAT设备后面
终端位于两层NAT设备之后，通常最上层的NAT是由ISP提供，    
第二层的NAT是家用的NAT路由器之类的设备。    



