首先看看ThreadLocalRandom的用法
```java
ThreadLocalRandom uuidRandom=ThreadLocalRandom.current();
for (int i = 0; i < length; ++i) {
    System.out.println(uuidRandom.nextInt(62));
}
```
之后点进去看ThreadLocalRandom.current()函数
```java
public static ThreadLocalRandom current() {
    if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}
```
引用了一个UNSAFE
```java
private static final sun.misc.Unsafe UNSAFE;
```
看看UNSAFE类的初始化过程
```java
static {
    try {
        //首先获取一个UNSAFE实例
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> tk = Thread.class;
        //获取Thread类里面threadLocalRandomSeed,threadLocalRandomProbe,threadLocalRandomSecondarySeed变量在Thread实例里面偏移量
        SEED = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSeed"));
        PROBE = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomProbe"));
        SECONDARY = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```
