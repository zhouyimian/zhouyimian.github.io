---
layout:     post
title:      JDK源码阅读—AbstractCollection
subtitle:   
date:       2019-5-29
author:     BY KiloMeter
header-img: img/2019-5-29-JDK源码阅读—AbstractCollection/56.png
catalog: true
tags:
    - 源码阅读
---

该类内部有iterator()和size()两个抽象方法

```java
public abstract Iterator<E> iterator();

public abstract int size();
```

判断是否为空

```java
public boolean isEmpty() {
    return size() == 0;
}
```

判断是否含有某对象，通过迭代器进行遍历

```java
public boolean contains(Object o) {
        Iterator<E> it = iterator();
        if (o==null) {
            while (it.hasNext())
                if (it.next()==null)
                    return true;
        } else {
            while (it.hasNext())
                if (o.equals(it.next()))
                    return true;
        }
        return false;
    }
```

转换成数组形式，部分代码解释以注释方式写在代码中

```java
//下面有两个toArray()方法，对于第一个toArray方法，无法使用像下面的代码将list转换成数组，会抛出类型转换异常，因为该方法的返回值是Object[]类型，该类型无法转换成String[]类型，所以第一个方法有点鸡肋
//List<String> list = new ArrayList<>();
//list.add("1");
//list.add("2");
//String[] ans = (String[]) list.toArray();
public Object[] toArray() {
        // Estimate size of array; be prepared to see more or fewer elements
        Object[] r = new Object[size()];
        Iterator<E> it = iterator();
        //循环中进行判断是因为在迭代过程中，可能会出现其他线程也对该集合进行remove操作，导致集合长度小于刚开始循环时的长度
        for (int i = 0; i < r.length; i++) {
            if (! it.hasNext())
                return Arrays.copyOf(r, i);
            r[i] = it.next();
        }
        //也有可能出现其他线程添加数据的操作
        return it.hasNext() ? finishToArray(r, it) : r;
    }
//下面第二种toArray()方法比较好用，用法如下
//List<String> list = new ArrayList<>();
//list.add("1");
//list.add("2");
//String[] ans = list.toArray(new String[list.size()]);
public <T> T[] toArray(T[] a) {
        
        int size = size();
        T[] r = a.length >= size ? a :
                  (T[])java.lang.reflect.Array
                  .newInstance(a.getClass().getComponentType(), size);
        Iterator<E> it = iterator();
        //最终返回数组的长度是Math.max(a.length,size())
        for (int i = 0; i < r.length; i++) {
            if (! it.hasNext()) { // fewer elements than expected
                //如果一开始a.length就是>=size的话，剩下的位置用null填充
                if (a == r) {
                    r[i] = null; // null-terminate
                //这种情况是一开始a.length<size，但是在迭代的过程中，其他线程对该集合进行了remove操作但是结果仍然比a.lenth长，这种情况下直接返回已经得到的结果数组
                } else if (a.length < i) {
                    return Arrays.copyOf(r, i);
                } else {
                    //最后这种情况是，第二种情况中其他线程进行remove操作后，结果比a.length短
                    System.arraycopy(r, 0, a, 0, i);
                    if (a.length > i) {
                        a[i] = null;
                    }
                }
                return a;
            }
            r[i] = (T)it.next();
        }
        //这种情况就是在迭代过程中，其他线程往该集合中添加了更多的对象，需要额外处理。
        // more elements than expected
        return it.hasNext() ? finishToArray(r, it) : r;
    }
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
private static <T> T[] finishToArray(T[] r, Iterator<?> it) {
        int i = r.length;
        while (it.hasNext()) {
            int cap = r.length;
            //这里的i是数组中的数据量，cap是数组的长度，当二者相等时，即数组满了，需要对数组进行扩容，默认扩容是原来长度的1.5倍+1，
            //如果超出MAX_ARRAY_SIZE(Integer.MAX_VALUE-8)，就判断原来长度+1是否大于MAX_ARRAY_SIZE，如果大于，则扩容后的大小为Integer.MAX_VALUE,否则扩容后大小为MAX_ARRAY_SIZE
            if (i == cap) {
                int newCap = cap + (cap >> 1) + 1;
                // overflow-conscious code
                if (newCap - MAX_ARRAY_SIZE > 0)
                    newCap = hugeCapacity(cap + 1);
                r = Arrays.copyOf(r, newCap);
            }
            r[i++] = (T)it.next();
        }
        // trim if overallocated
        return (i == r.length) ? r : Arrays.copyOf(r, i);
    }
private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError
                ("Required array size too large");
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

删除指定对象

```java
public boolean remove(Object o) {
        Iterator<E> it = iterator();
        if (o==null) {
            while (it.hasNext()) {
                if (it.next()==null) {
                    it.remove();
                    return true;
                }
            }
        } else {
            while (it.hasNext()) {
                if (o.equals(it.next())) {
                    it.remove();
                    return true;
                }
            }
        }
        return false;
    }
```

判断是否包含集合中的所有元素

```java
public boolean containsAll(Collection<?> c) {
        for (Object e : c)
            if (!contains(e))
                return false;
        return true;
    }
```

添加集合中的所有元素

```java
public boolean addAll(Collection<? extends E> c) {
        boolean modified = false;
        for (E e : c)
            if (add(e))
                modified = true;
        return modified;
    }
```

移除给定集合中的所有元素

```java
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);
    boolean modified = false;
    Iterator<?> it = iterator();
    while (it.hasNext()) {
        if (c.contains(it.next())) {
            it.remove();
            modified = true;
        }
    }
    return modified;
}
```

求与给定集合的交集

```java
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    boolean modified = false;
    Iterator<E> it = iterator();
    while (it.hasNext()) {
        if (!c.contains(it.next())) {
            it.remove();
            modified = true;
        }
    }
    return modified;
}
```

清空集合的所有元素

```java
public void clear() {
    Iterator<E> it = iterator();
    while (it.hasNext()) {
        it.next();
        it.remove();
    }
}
```

集合打印

```java
public String toString() {
    Iterator<E> it = iterator();
    if (! it.hasNext())
        return "[]";

    StringBuilder sb = new StringBuilder();
    sb.append('[');
    for (;;) {
        E e = it.next();
        sb.append(e == this ? "(this Collection)" : e);
        if (! it.hasNext())
            return sb.append(']').toString();
        sb.append(',').append(' ');
    }
}
```