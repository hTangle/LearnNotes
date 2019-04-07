### OutOfMemoryError
* 分配的内存太少
* 应用占用的太多

1. 堆内存溢出：通过-Xms,-Xms修改
2. 永久代溢出：方法区溢出了，存在大量的Class信息-XX:PermSize=64m -XX:MaxPermSize=256m
3. java.lang.StackOverflowError：栈溢出，-Xss