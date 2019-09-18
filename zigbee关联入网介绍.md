# ZigBee设备入网流程

ZigBee设备入网有关联方式和直接方式两种，我所熟悉的是关联方式，这也是最常用的方式。
## 关联方式
![](https://i.imgur.com/rZnatKL.png)

### step1 设备发出Beacon Request
![](https://i.imgur.com/hoXLfvg.png)

设备会在预先设置的几个信道里面按照指定的顺序逐信道发出这个包，看到Dest PAN ID,Dest Address都是0xFFFF，说明这是个广播包，在这些信道里面的网络都会收到它。

### step2 route节点发出Beacon回复
![](https://i.imgur.com/kEqVwnU.png)
这个回复里面有五个关键的值
- Source PAN ID	：回复Beacon的这个设备所处网络的PAN ID
- Source Address：回复Beacon的这个设备所处网络的短地址
- Association Permit：关联许可是否开放
- Router Capacity：可否接入Route节点
- End Device Capacity：可否接入End Device

能收到入网设备发出的`Beacon Request`的网络都会回复`Beacon`，并且同一个网络里面能收到入网设备`Beacon Request`的FFD设备都会回复`Beacon`。这样一来，一般入网设备会受到多个`Beacon`回复。那么它会按照下列的顺序，并且结合这帧Beacon的Link Quality来进行下一步动作：
1. 	入网设备首先判断`Association Permit`是否开放，这个是需要协调器发出全网广播，通知所有route节点这个许可开放了。
1. 	如果关联许可是开放的，再根据自己所属的设备类型来判断`Router Capacity`、 `End Device Capacity`。
1. 	如果可以接入，再筛选最佳Link Quality的设备发出`Association Request`，这个时候就需要用Beacon里面的Source PAN ID和Source Address发出一个MAC层的单播包。

### step3 设备发出Association Request
![](https://i.imgur.com/Nw7m6Gl.png)

### step4 route发出Association Response
![](https://i.imgur.com/5vfZ0WG.png)

### step5 秘钥传输
![](https://i.imgur.com/ofLqdx5.png)

## step6 Device Announce
Device Announce的广播数据主要是通知全网相关节点有一个新设备进来了，给大家做个自我介绍，大家刷新下路由表这类的信息。并且可以看到此时的数据在NWK层加密了，就是用了上面的Transport Key传输的Standard Network Key。
![](https://i.imgur.com/RN3YD5p.png)