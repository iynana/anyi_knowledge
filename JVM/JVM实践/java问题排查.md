+ top命令查看cpu占用
+ jstack查看进程
+ 排查GC日记
+ ss -lntp，排查服务是否启动，端口监听
+ arthas（阿尔萨斯）
	+ thread -b 找到阻塞线程（如果看门狗机制一直续约导致一堆进程阻塞，可以直接登录redis释放锁，然后事后对账补偿）
	+ thread -n 占用cpu高线程