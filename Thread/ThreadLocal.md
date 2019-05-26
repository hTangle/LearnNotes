## ThreadLocal
### 实际通过ThreadLocal创建的副本存储在每个线程自己的ThreadLocals中，ThreadLocal.ThreadLocalMap threadLocals = null;
### 每个线程可以有多个threadLocals变量
### 在get之前需要set，否则会有空指针异常
### Entry是弱引用：保证ThreadLocal的生命周期与线程相同，方便GC
### 通过线性探测法解决Hash冲突
### 空间换时间的解决冲突办法
### 用于数据库连接和Session管理
ThreadLocal.java
```
public class ThreadLocal<T> {
    private final int threadLocalHashCode = nextHashCode();//标记不同的线程
    private static AtomicInteger nextHashCode =
        new AtomicInteger();
    
    public T get() {
        Thread t = Thread.currentThread();//获取当前线程
        ThreadLocalMap map = getMap(t);//获取当前线程对应的Map
        if (map != null) {//map不为空，返回value
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();//返回初始化值
    }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;//返回当前线程t的threadLocals  
    }
    static class ThreadLocalMap {
        private Entry[] table;//使用数组存储各个线程的变量
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
    }
    //不为空，设置键值对，为空直接创建map
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
    //初始时，在Thread里面，threadLocals为空，
    //当通过ThreadLocal变量调用get()方法或者set()方法，
    //就会对Thread类中的threadLocals进行初始化，
    //并且以当前ThreadLocal变量为键值，
    //以ThreadLocal要保存的副本变量为value，存到threadLocals。
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
}
```
