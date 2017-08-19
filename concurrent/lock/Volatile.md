
## volatile的特性
可见性：对一个volatile变量的读，任意线程总是能看到该变量的最后的值。
原子性：对任意单个volatile变量的读写具有原子性。

## volatile写-读建立的happens-before关系
通过volatile变量的写-读可以实现线程之间的通信。通过volatile的写-读与锁的释放-获取有相同的内存效果。

## 参考资料：
http://www.cnblogs.com/paddix/p/5428507.html