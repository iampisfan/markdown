# OpenWrt网络服务
**Jinhua Zhang
v1.0
2017.09.05**
<br>

## 1. preinit

## 2. ubus

ubus是OpenWrt中的进程间通信机制，类似于桌面版linux的dbus，Android的binder。ubus相当于简化版的dbus，ubus基于unix socket实现，socket绑定到一个本地文件，具有较高的效率；

unix socket是C/S模型，建立一个socket连接，server端和client端分别要做如下步骤：
1. 建立一个socket server端，绑定到一个本地socket文件，监听client的连接；
2. 建立一个或多个socket client端，连接到server端；
3. client端和server端相互发送消息；
4. client端或server端收到对方消息后，针对具体消息进行相应处理。

如下图所示：
<center>![](sockcet-pair.png)</center>

ubus同样基于这套流程，其中ubusd实现server，其他进程实现client，例如ubus(cli)、netifd、procd；
两个client通信需要通过server转发。

### 2.1. ubusd
ubusd作为ubus的server端，已经由OpenWrt实现好了，不需要做任何修改，下面来分析一下ubusd的工作流程；
1. 通过usock来创建server端socket，且socket bind到文件"/var/run/ubus.sock"，开启listen，等到client的连接；
2. 将socket添加到uloop中poll，触发条件是read，也就是说server socket可读，则触发poll回调server_cb，server_fd的回调函数是server_cb；                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                
```
static struct uloop_fd server_fd = {
	.cb = server_cb,
};
```
3. server_cb中通过accept来接受client的连接，套接字函数accept执行退出之后会创建一个新的socket，新的socket的fd为accept函数值，而旧的socket不变，也就是accept之后，server端存在两个socket了，旧的socket依然用来listen，新的socket与client建立pair，用于和client的通讯；
4. 根据新的socket的fd(int client_fd)来构建一个ubus_client对象，ubus_client的uloop回调函数是client_cb，将ubus_client插入到avl树中(struct avl_tree clients)，每个向ubusd注册的client都在对应到avl树中的一个ubus_client；
5. ubusd_send_hello，向client端的socket发送一个字符串"hello"；
6. 将ubus_client中的socket fd添加到uloop中去轮询监听，触发条件是read，回调函数是client_cb，也就是client发消息到server，则触发回调client_cb；

client_cb通过write或sendmsg来发消息给client，通过read或recvmsg来接收来自client的消息，ubusd_proto_receive_message根据接收的不同消息类型做不同的处理，如下所示：

| 消息类型                | 处理函数                    |
|:-----------------------|:---------------------------|
| UBUS_MSG_PING          | ubusd_send_pong            |
| UBUS_MSG_ADD_OBJECT    | ubusd_handle_add_object    |
| UBUS_MSG_REMOVE_OBJECT | ubusd_handle_remove_object |
| UBUS_MSG_LOOKUP        | ubusd_handle_lookup        |
| UBUS_MSG_INVOKE        | ubusd_handle_invoke        |
| UBUS_MSG_STATUS        | ubusd_handle_response      |
| UBUS_MSG_DATA          | ubusd_handle_response      |
| UBUS_MSG_SUBSCRIBE     | ubusd_handle_add_watch     |
| UBUS_MSG_UNSUBSCRIBE   | ubusd_handle_remove_watch  |
| UBUS_MSG_NOTIFY        | ubusd_handle_notify        |


### 2.2. ubus cli

OpenWrt为ubus实现了一个cli可执行程序，这个可执行程序的名称是"ubus"，shell script中大部分情况下都是通过ubus命令做跨进程通讯，这个应用场景下，ubus作为client端，发送消息到server端ubusd，ubusd再转发到另一个client端。

ubus支持的commands有list、call、listen、send、wait_for和monitor；
```
static struct {
	const char *name;
	int (*cb)(struct ubus_context *ctx, int argc, char **argv);
} commands[] = {
	{ "list", ubus_cli_list },
	{ "call", ubus_cli_call },
	{ "listen", ubus_cli_listen },
	{ "send", ubus_cli_send },
	{ "wait_for", ubus_cli_wait_for },
	{ "monitor", ubus_cli_monitor },
};
```

- list
命令的使用格式是：
```
ubus [-v] list [path]
```
命令用于列举出系统注册的ubus object和method；
例如：
```
root@OpenWrt:/# ubus list
dhcp
hostapd.wdev0ap0
hostapd.wdev1ap0
log
network
network.device
network.interface
network.interface.lan
network.interface.loopback
network.interface.wan
network.interface.wan6
network.wireless
service
session
system
uci
root@OpenWrt:/# ubus -v list network
'network' @26c90ada
        "restart":{}
        "reload":{}
        "add_host_route":{"target":"String","v6":"Boolean","interface":"String"}
        "get_proto_handlers":{}
        "add_dynamic":{"name":"String"}
```

- call
命令的使用格式是：
```
ubus call path method [message]
```
命令用于执行某个ubus client注册的某个的object的某个method；
例如：
```
root@OpenWrt:/# ubus -v call network.interface.lan status
{
        "up": true,
        "pending": false,
        "available": true,
        "autostart": true,
        "dynamic": false,
        "uptime": 1698,
        "l3_device": "br-lan",
        "proto": "static",
        "device": "br-lan",
        "updated": [
                "addresses"
        ],
        "metric": 0,
        "delegation": true,
        "ipv4-address": [
                {
                        "address": "192.168.1.1",
                        "mask": 24
                }
        ],
        ...
}
```

- listen
命令的使用格式是：
```
ubus listen [path]
```
设置一个监听socket并观察进入的事件；
例如：
```
root@OpenWrt:/# ubus listen &
root@OpenWrt:/# /etc/init.d/network restart
{ "ubus.object.remove": {"id":-369156789,"path":"network.interface.wan6"} }
{ "ubus.object.remove": {"id":10693651,"path":"network.interface.wan"} }
{ "ubus.object.remove": {"id":1946155556,"path":"network.interface.lan"} }
{ "ubus.object.remove": {"id":-696825625,"path":"network.interface.loopback"} }
{ "ubus.object.remove": {"id":-1930323617,"path":"network.interface"} }
{ "ubus.object.remove": {"id":-1647942339,"path":"network.wireless"} }
{ "ubus.object.remove": {"id":-886262541,"path":"network.device"} }
{ "ubus.object.remove": {"id":-1084952745,"path":"network"} }
{ "ubus.object.add": {"id":-22896841,"path":"network"} }
{ "ubus.object.add": {"id":-1554358219,"path":"network.device"} }
{ "ubus.object.add": {"id":-1777113328,"path":"network.wireless"} }
{ "ubus.object.add": {"id":1365477379,"path":"network.interface"} }
{ "ubus.object.add": {"id":1148732338,"path":"network.interface.loopback"} }
{ "ubus.object.add": {"id":1802312926,"path":"network.interface.lan"} }
{ "ubus.object.add": {"id":982976204,"path":"network.interface.wan"} }
{ "ubus.object.add": {"id":-1790820211,"path":"network.interface.wan6"} }
{ "network.interface": {"action":"ifup","interface":"lan"} }
{ "network.interface": {"action":"ifup","interface":"loopback"} }
{ "ubus.object.remove": {"id":-627066478,"path":"hostapd.wdev1ap0"} }
{ "ubus.object.remove": {"id":-208119521,"path":"hostapd.wdev0ap0"} }
{ "ubus.object.add": {"id":154802105,"path":"hostapd.wdev0ap0"} }
{ "ubus.object.add": {"id":-1230601970,"path":"hostapd.wdev1ap0"} }
{ "ubus.object.remove": {"id":-1230601970,"path":"hostapd.wdev1ap0"} }
{ "ubus.object.remove": {"id":154802105,"path":"hostapd.wdev0ap0"} }
{ "ubus.object.add": {"id":914025961,"path":"hostapd.wdev0ap0"} }
{ "ubus.object.add": {"id":488528099,"path":"hostapd.wdev1ap0"} }
```

- send
命令的使用格式是：
```
ubus send type [message]
```
发送一个消息；
例如：
```
root@OpenWrt:/# ubus listen &
root@OpenWrt:/# ubus send network.interface '{"action":"ifup","interface":"lan"}'
{ "network.interface": {"action":"ifup","interface":"lan"} }
```

- wait_for
命令的使用格式是：
```
ubus wait_for object [...]
```
等待某个事件；
例如启动netifd后，/etc/init.d/network会wait_for network_interface这个object的add事件；
```
ubus -t 30 wai_for network.interface
```

- monitor
监听，一般不用

下面以"ubus call"命令为例，来说明ubus client的工作流程。
1. ubus_connect做了三部分工作，a)构造结构体ubus_context，ubux_context表示client端的ubus上下文，包含client注册的object avl tree，client sock，msgbuf；b)创建client unix socket；c)connect to server socket。
2. ubus_lookup_id向ubusd发送消息UBUS_MSG_LOOKUP，查到path对应的object id；ubusd对应的处理函数为ubusd_handle_lookup，ubusd中维护了所有注册过的object的path avl tree，遍历path avl tree进行字符串匹配即可找到对应的object，然后通过ubusd_send_obj将object的UBUS_ATTR_OBJPATH、UBUS_ATTR_OBJID、UBUS_ATTR_OBJTYPE等信息发回给ubus cli，消息类型是UBUS_MSG_DATA。
3. ubus cli端ubus_lookup_id的回调函数是ubus_look_id_cb，所有ubus cli接收到消息后在ubus_lookup_id_cb中解析出来一个id号。
4. ubus_invoke向ubusd发送消息UBUS_MSG_INVOKE，消息中携带了信息UBUS_ATTR_OBJID和UBUS_ATTR_METHOD，ubusd对应UBUS_ATTR_METHOD的处理函数是ubusd_handle_invoke，ubusd根据OBJID找到注册的object，并发UBUS_MSG_INVOKE消息给该client进程，该client进程执行对应的method后，将执行结果发回给ubusd。
5. ubusd将method的执行结果发给ubus cli，ubus cli在回调函数receice_call_result_data将执行结果打印出来。

总体流程如下图所示：
<center>![](ubus-call.png)</center>

## 3. blob、json、uci

json即为JavaScript object Notation，用于异步应用程序中发送和接收信息使用，json是一种简单的数据交换格式，在某些方面，它的作用和xml非常类似，但比xml更为简单。
简单的说，json可以将JavaScript对象中表示的一组数据转换为字符串，然后就可以在函数之间轻松地传递这个字符串，或者在异步应用中将字符串从web客户端传递给服务器端程序；
常见api有：
json_init
json_cleanup
json_add_string
json_add_boolean
json_add_int
json_add_object
json_add_array
json_add_array_data
json_set_namespace
json_select
json_get_values

## 4. procd

## 5. netifd