## JAVA
### 类加载器
1. 根类加载器:jre/lib
2. 扩展类加载器:jre/lib/ext扩展类库
3. 应用类加载器:用户自己定义的类
#### 双亲委派模型
* 调用类加载器的LoadClass方法进行类加载时（需要加锁synchronized），该类加载器会首先请求父类加载器，依次递归，如果所有的父类加载器都加载失败，则当前类加载器自己进行加载操作（避免用户自定义的类覆盖核心类库），如果找不到抛出NoSuchClass异常
* 如果需要实现自己的类加载器且不破坏双亲委派模型，需要继承ClassLoader类并重写findClass方法，如果需要破坏双亲委派模型，则重写loadClass方法
* loadClass的逻辑里如果父类加载器加载失败会调用findClass方法完成加载
```
protected Class<?> loadClass(String name, boolean resolve)throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

> IO：同步，阻塞
> NIO：同步非阻塞
> > Channel：流
> > Buffer：从流中读写数据
> > Selector：注册到很多个Channel上，监听每个Channel上发生的事件，根据情况决定Channel的读写，可以通过一个线程管理多个Channel
> > 阻塞模式下注册操作不允许

> AIO：异步，非阻塞
> IO操作完成以后再给线程发送通知。

### 红黑树和平衡二叉树的区别
* 平衡二叉树左右子树高度差不超过1，是严格的平衡二叉树，适合插入删除少但查找多的情况
* *红黑树是弱平衡二叉树，高度一般不小于平衡二叉树，适合搜索插入删除操作的情况下，根节点到叶子节点的最长路径不多于最短路径的两倍，插入删除查找复杂度都是O(logN),**最多3次旋转能够达到平衡**，

### Java反射机制
* 对于任意一个类，都能够获得这个类的所有属性和方法，运行时动态获取信息以及动态调用对象的方法的功能为反射
* java.lang.reflect:Field，Method，Constructor
* 通过三种方法获取字节码对象：
  1. Object.getClass()
  2. 静态属性.class
  3. 给定类的字符串名称Class.forName()-->ClassNotFoundException 