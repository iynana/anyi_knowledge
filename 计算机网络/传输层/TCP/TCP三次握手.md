![[Pasted image 20260324015140.png]]
- C→S：SYN = 1，Seq = x
- S→C：SYN = 1，Seq = y，Ack = x + 1
- C→S：Seq = x + 1，Ack = y + 1（可以携带数据）

**为什么不是两次？**
防止滞留在网络中的（已经无效的）历史报文，被重新接收



