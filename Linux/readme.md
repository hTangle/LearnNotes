IO模型
1）同步阻塞
2）同步与非阻塞
3）IO多路复用：
	1）select：每次调用select，都需要把fd集合从用户态拷贝到内核态，每次调用select都需要在内核遍历传递过来的所有的fd，支持的文件描述符太少
	2）poll：pool使用的描述fd的集合不同，使用的是pollfd
	3）epoll：epoll_create,epoll_ctl和epoll_wait，epoll_create是创建一个epoll句柄；epoll_ctl是注册要监听的事件类型；epoll_wait则是等待事件的产生。
		○ epoll保证每个fd在整个过程中只会拷贝一次，不会随着fd数目的增长而降低效率（维护的是一个队列）
		○ 不轮询，只在epoll_ctl时把current挂一遍并为每个fd指定一个回调函数，当设备就绪时，唤醒等待队列上的等待者，就会调用这个回调函数（轮询就绪链表）
		○ epollfd数量上限很大
	
Linux常用命令
查看端口占用情况
	Lsof -i:8000
	Netstat -tunlp | grep 8000
	Ps -aux | grep tomcat
查看磁盘容量
	Df （-h -a）
	
git fetch --all 
git reset --hard origin/master
git pull
