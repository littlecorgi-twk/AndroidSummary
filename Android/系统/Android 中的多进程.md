# Android 中的多进程

## 什么是 IPC

IPC，进程间通信，指两个进程间的数据交换的过程。但是首先搞明白，什么是进程以及为何要特殊的机制来通信。



### 什么是进程

一般的操作系统中，进程都对应了一个执行单元。比如在 Android 中，一般指一个应用程序（但是一个 APP 也可以设置多个进程），而操作系统为了保证应用程序执行的稳定，所以设置了进程隔离，也就是进程间没法直接的互相通信。

但是有些情况下，我们确确实实有进程间通信的诉求，所以这个时候就需要 IPC 这个概念。



## Android中开启多进程的方式



## 多进程传输数据的基础

### Serializable 接口

`Serializable`是 Java 提供的序列化接口，实现起来非常简单。

1. 让数据类实现这个接口
2. 设置一个 Long 常量 `serialVersionUID` 即可。

这样子就完成了。序列化和反序列化也可以轻松通过 `ObjectOutputStream`和`ObjectInputStream`实现。

```java
// 序列化
User user = new User(0, "jake", true);
ObjectOutputSteam out = new ObjectOutputStream(new FileOutputStream("cache.txt"));
out.writeObject(user);
out.close;

// 反序列化
ObjectInputStream in = new ObjectInputStream(new FileInputStream("cache.txt"));
User newUser = (User) in.readObject();
in.close();
```

#### serialVersionUID

`serialVersionUID` 随便指定就行，但是不指定，其实也能实现序列化。 这个 UID 只是为了辅助实现序列化的，序列化时会将 UID 写入序列化后的数据，反序列化时会进行校验，如果序列化数据的 UID 和指定的类的 UID 不匹配的话，会序列化失败，抛出 `java.io.InvalidClassException` 异常。

### Parcelable 接口

`Parcelable` 和 `Serializable`类似，也是个接口，但是它的实现要更复杂些，因为他默认提供了一个方法：`writeToParcel()` 以及一个接口 `Creator`，而接口`Creator`中又提供了一个方法 `createFromParcel()`。

看到这很好理解了， `writeToParcel()`中实现如何将数据序列化，`createFromParcel()`中实现如何将数据反序列化。

## Android中的 IPC 方式



### Binder



### AIDL 



### ContentProvider



### Messager



