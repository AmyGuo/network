#### 一、tcp队列
1. syns queue:半连接队列
tcp三次握手，第一步，服务端收到客户端发送的syn消息后，将连接信息放入syns queue，此时，双方连接尚未建立，称之为半连接
2. accept queue:全连接队列
tcp三次握手，第三部，客户端收到服务端发送的syn+ack消息后，向客户端发送ack消息，服务端接收到此消息后，将连接信息从syns queue拿出，放入accept queue，此时，经过三次握手，连接已经建立，称之为全连接

#### 二、队列溢出
既然是队列，那就会存在队列被填满的情况，我们称之为队列溢出
1. syns queue满
加入一段时间内，有大量的syn请求连接信息到达，如果后续连接建立处理不及时，或者有客户端方面恶意不处理后续连接，那么很快就会沾满syns queue，从而导致无法建立新的连接。

2. accept queue满
完成三次握手，则会触发连接信息的队列转移，加入此时，accept queue队列满，则会导致新建立的连接得不到维护保持，系统会根据设定的策略（tcp_abort_on_overflow）进行链接的直接抛弃（0）或者发送RST消息给客户端终止连接（1）（connetction reset by peer）

3. 查看队列溢出
命令：netstat -s | egrep "listen|LISTEN"
结果：
全连接队列溢出次数：6 times the listen queue of a socket overflowed
半连接队列溢出次数：6 SYNs to LISTEN sockets dropped

4. 查看队列使用情况
命令：ss -lnt
结果：
State  Recv-Q  Send-Q  Local Address:Port   Peer AddressLport
LISTEN    0       128         *:6379              *:*

Send-Q:监听端口上全连接队列最大的大小
Recv-Q:全连接队列当前使用量

#### 三、关于连接错误
connection reset：已关闭的连接上执行读操作触发
connection reset by peer：已关闭的连接上执行写操作触发

#### 四、关于RST消息
连接重置消息，用于连接的异常关闭
下面简单罗列几种可能触发RST连接关闭的情景：
1. 服务端接收到自身不存在端口的连接请求
2. 主动使用RST关闭，代替正常的四次挥手FIN消息关闭，主要用于特殊优化提升效率使用
3. 客户端或者服务端异常，无法继续正常的连接处理，发送RST终止连接操作
4. 处理TCP游离包信息
5. 长期未收到对方确认报文，经过一定时间或者重传尝试后，发送RST终止连接。
