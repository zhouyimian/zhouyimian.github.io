---
layout:     post
title:      JDK源码阅读—Object
subtitle:   
date:       2019-5-19
author:     BY KiloMeter
header-img: img/2019-5-19-JDK源码阅读—Object/53.png
catalog: true
tags:
    - 源码阅读
---

Object类中最开始是一个native方法

**private static native void registerNatives()**

![](/img/2019-5-19-JDK源码阅读—Object/registerNatives.png)

放在静态代码块中执行，被native修饰的方法，就是不用Java实现了，使用的是其他语言如C去实现，在Java中只需要进行声明即可。

接下来也是一个native方法

**public final native Class<?> getClass();**

![](/img/2019-5-19-JDK源码阅读—Object/getClass.png)

返回值是Class类型，通过Class类对象可以获取到改类的方法和属性，以及构造方法等信息。

接下来是hashCode方法和equals方法

**public native int hashCode();**

![](/img/2019-5-19-JDK源码阅读—Object/hashcode.png)

hashcode方法是一个本地方法，Object类的hashcode方法返回当前对象的内存地址经过处理后的结构，每个对象的内存地址不一样，所以哈希码也不一样

**public boolean equals(Object obj){return (this == obj);}**

![](/img/2019-5-19-JDK源码阅读—Object/equals.png)

equals用于比较两个对象是否相等，默认的实现是使用\=\=进行判断，只有两个对象的内存地址一样时，\=\=的结果才会为true，因此一般要比较对象时，需要覆盖equals方法

在使用HashSet，HashMap等跟对象哈希值相关的数据结构时，需要同时覆盖hashCode和equals方法。

这两个方法的联系是：如果两个对象相等(即equals的值为true)，则两个对象的hashCode必须相等，反过来则不一定成立，即如果两个对象的hashCode相等，equals方法的返回值不一定为true。

在HashSet，HashMap中，底层使用的数据结构都是数组+链表+红黑树(HashSet的底层实现基本都是依赖于HashMap)，当HashMap插入数据时，先根据对象的hashCode判断出要插入的对象应该在数组中的位置，如果得到的hash值在数组中的位置已经被其他对象占有了，则这个对象和即将要插入的这两个可能是相同的(这里的相同指的是自定义的equals的结果可能是true，而不是指这两个对象的内存地址一样，因为在一般的使用场景中，假设有student这个类，这个类有studentID和name这两个属性，如果有两个对象的studentID和name的属性值完全一样，则代表了这两个对象是同一个，如果没有覆盖equals方法的话，即根据对象的内存地址进行判断两个对象是否相同，则会出现相同studentID和name的对象但是不是属于同一样对象的判断结果，相信这个结果不是我们想要的)，那么这里就需要使用equals方法来进行判断了，如果equals判断为true，则是同一个对象，不允许重复插入，否则跟在该位置链表的最后面。

接下来是clone方法

**protected native Object clone() throws CloneNotSupportedException;**

![](/img/2019-5-19-JDK源码阅读—Object/clone.png)

关于clone方法，源码中的注释写了，只有实现了Cloneable接口的对象才能够调用该方法，否则会抛出CloneNotSupportedException异常。对于所有的数组类型，默认都实现了Cloneable接口。

对于普通数值类型(如int，float等)和其对应的包装类类型(如Integer，Float等)，clone出的都是全新的数组，即和原来的对象没有关系。

如果要实现对象的深拷贝，则需要在让对象实现Cloneable接口，并在clone方法中把对象的引用属性全都进行复制，但是如果要拷贝的对象中，属性中所指向的对象，又有其他对象的引用的话，这样的“深拷贝”是不完全的。下面举个例子

```java
	static class Body implements Cloneable{
		public Head head;
		public Body() {}
		public Body(Head head) {this.head = head;}
 
		@Override
		protected Object clone() throws CloneNotSupportedException {
			Body newBody =  (Body) super.clone();
			newBody.head = (Head) head.clone();
			return newBody;
		}
		
	}
	
	static class Head implements Cloneable{
		public  Face face;
		
		public Head() {}
		public Head(Face face){this.face = face;}
		@Override
		protected Object clone() throws CloneNotSupportedException {
			return super.clone();
		}
	} 
	
	static class Face{}
	
	public static void main(String[] args) throws CloneNotSupportedException {
		
		Body body = new Body(new Head(new Face()));
		
		Body body1 = (Body) body.clone();
		
		System.out.println("body == body1 : " + (body == body1) );
		
		System.out.println("body.head == body1.head : " +  (body.head == body1.head));
		
		System.out.println("body.head.face == body1.head.face : " +  (body.head.face == body1.head.face));
		
		
	}
```

对于body对象，如果仅仅只是实现了上面的clone方法，虽然拷贝出来的head确实是新的对象，但是拷贝出来的head中，两个head内部的face对象仍然是指向同一个对象，因此如果想要实现真正的深拷贝，还需要让head对象实现深拷贝，拷贝出新的face对象。(如果face对象也有其他的对象引用，也需要让face实现clone方法)



**public String toString(){return getClass().getName() + "@" + Integer.toHexString(hashCode());}**

toString方法打印对象名+@+哈希值

![](/img/2019-5-19-JDK源码阅读—Object/toString.png)



**public final native void notify();**和**public final native void notifyAll();**

![](/img/2019-5-19-JDK源码阅读—Object/notify.png)



![](/img/2019-5-19-JDK源码阅读—Object/notifyAll.png)



**public final native void wait(long timeout) throws InterruptedException;**

![](/img/2019-5-19-JDK源码阅读—Object/wait.png)

notify，notifyAll和wait方法用于线程之间的通信。当调用对象的wait方法时，如果该线程没有持有该对象的锁的话，则会抛出java.lang.IllegalMonitorStateException异常，如果在持有锁的情况下，则该线程会释放该锁，并进行等待状态，只有其他获得该对象锁的线程调用该对象的notify或者notifyAll方法时，这些陷入等待状态的线程才会被唤醒(notify只随机唤醒某一个陷入该对象等待状态的线程，notify则是唤醒所有的陷入该对象等待状态的线程)

wait方法的timeout参数指定了至少要等待的时间才能被唤醒

**public final void wait(long timeout, int nanos) throws InterruptedException**

```java
public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos > 0) {
            timeout++;
        }

        wait(timeout);
    }
```

上面可以看到，wait方法可以额外加两个参数，设置最短等待时间，第二个参数不太清楚有什么用。第一个参数以ms为单位，第二个参数以ns为单位，源码中的注释说第二个参数能更好地控制时间(???不理解)

**public final void wait() throws InterruptedException **

```java
public final void wait() throws InterruptedException {
        wait(0);
    }
```

默认使用的wait，最短等待时间为0ms

**protected void finalize() throws Throwable { }**

如果类中重写了finalize方法，当该类对象被回收时，finalize方法**有可能**会被触发

该方法的调用时机：当垃圾回收器要宣告一个对象死亡时，至少要经过两次标记过程：如果对象在进行可达性分析后发现没有和GC Roots相连接的引用链，就会被第一次标记，并且判断是否执行finalizer( )方法，如果对象覆盖finalizer( )方法且未被虚拟机调用过，那么这个对象会被放置在F-Queue队列中，并在稍后由一个虚拟机自动建立的低优先级的Finalizer线程区执行触发finalizer( )方法，但不承诺等待其运行结束。 

finalization的目的：对象逃脱死亡的最后一次机会。（只要重新与引用链上的任何一个对象建立关联即可。）但是不建议使用，运行代价高昂，不确定性大，且无法保证各个对象的调用顺序。可用try-finally或其他替代。

参考：

[JDK源码阅读开始--Object类](https://mp.weixin.qq.com/s?src=11&timestamp=1558269565&ver=1616&signature=3r4ArvvkM*H59lM5CbI6XWNuErR0wE1zIspefirozfop8vAHAXVro2R9MpL249MKxRcbNYzUhNp988dA0eU9ssDtuRb9rnPV6drvOOaPZUkn7zbTNM*Dj6PD9aED3t4m&new=1)

[详解Java中的clone方法 -- 原型模式](https://blog.csdn.net/zhangjg_blog/article/details/18369201)

[finalize()方法什么时候被调用？析构函数(finalization)的目的是什么？](https://www.nowcoder.com/questionTerminal/d8eab06913084e42b515633604eef7cd)