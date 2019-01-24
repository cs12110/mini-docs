# ThreadLocal 笔记

关于 ThreadLocal 的使用笔记.

注意: **使用 ThreadLocal 每次执行完毕后,要使用 remove()方法来清空对象,否则 ThreadLocal 存放大对象后可能会 OMM.**

---

## 1. 基础知识

### 1.1 基础概念

使用 ThreadLocal 维护变量时,ThreadLocal 为每个使用该变量的线程提供独立的变量副本,所以每一个线程都可以独立地改变自己的副本,而不会影响其它线程所对应的副本.从线程的角度看,目标变量就象是线程的本地变量,这也是类名中"Local"所要表达的意思.可以理解为如下三点:

a. **每个线程都有自己的局部变量**

每个线程都有一个独立于其他线程的上下文来保存这个变量,一个线程的本地变量对其他线程是不可见的.

b. **独立于变量的初始化副本**

ThreadLocal 可以给一个初始值,而每个线程都会获得这个初始化值的一个副本,这样才能保证不同的线程都有一份拷贝.

c. **状态与某一个线程相关联**

ThreadLocal 不是用于解决共享变量的问题的,不是为了协调线程同步而存在,而是为了方便每个线程处理自己的私有状态而引入的一个机制,理解这点对正确使用 ThreadLocal 至关重要.

### 1.2 测试代码

```java
public class Bucket {
	public static void main(String[] args) {
		ThreadLocal<Long> threadLocal = new ThreadLocal<>();
		threadLocal.set(1L);
		System.out.println("ThreadLocal#get:" + threadLocal.get());
		threadLocal.remove();
		System.out.println("ThreadLocal#remove:" + threadLocal.get());
	}
}
```

测试结果

```java
ThreadLocal#get:1
ThreadLocal#remove:null
```

下面,我们一起来看看这个杀千刀的玩意.

---

## 2. `ThreadLocal#set` 方法

流程: 得到当前线程 -> 根据当前线程获取 ThreadLocalMap -> 保存值到 ThreadLocalMap

threadlocal 里面的内部实现 set 代码如下:

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

`ThreadLocal#createMap`方法代码如下

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

`ThreadLocal#getMap`方法代码如下

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

那么 t.threadLocals 又是什么玩意?

就是 ThreadLocal 里面的 Map,整这么多幺蛾子,我竟然看不懂为什么,我太渣了.

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

`ThreadLocalMap`:你可以理解为一个简易的 HashMap

---

## 3. `ThreadLocal#get` 方法

获取 ThreadLocal 里面的值.

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

`ThreadLocal#setInitialValue`方法代码如下

```java
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
```

`ThreadLocal#initialValue`方法代码如下

```java
protected T initialValue() {
    return null;
}
```

---

## 4. `ThreadLocal#remove` 方法

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

由上面的代码可以看出来,也是根据当前线程获取当前线程里面的 ThreadLocalMap,然后根据 key 移除 value.

---

## 5. 总结

可能有点绕,我尝试总结一下.

ThreadLocal 的操作都是: 获取当前线程(`Thread.currentThread()`) -> 对当前线程里面的 ThreadLocals(实际为`ThreadLocal.threadLocals`)属性值进行操作 -> `key` 为 `ThreadLocal` 的 `this` 对象,`value` 为用户 `set` 进来的值.
