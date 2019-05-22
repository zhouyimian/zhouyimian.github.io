---
layout:     post
title:      JDK源码阅读—String
subtitle:   
date:       2019-5-20
author:     BY KiloMeter
header-img: img/2019-5-20-JDK源码阅读—String/54.png
catalog: true
tags:
    - 源码阅读
---

String字符串在底层是以字符数组的形式进行存放的，默认实现了序列化接口

![](/img/2019-5-20-JDK源码阅读—String/value_hashcode.png)

### 构造方法

**public String(){this.value = "".value;}**

**public String(String original)**

```java
public String(String original) {
        this.value = original.value;
        this.hash = original.hash;
    }
```

**public String(char value[]){this.value = Arrays.copyOf(value, value.length);}**

**public String(char value[],int offset,int count)**

```java
public String(char value[], int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count <= 0) {
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            if (offset <= value.length) {
                this.value = "".value;
                return;
            }
        }
        // Note: offset or count might be near -1>>>1.
        if (offset > value.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }
        this.value = Arrays.copyOfRange(value, offset, offset+count);
    }
```

前面几个构造方法都比较简单，注意，从上面的方法中可以看出，String类型在创建的时候，字符串的Hash值也是随之同时赋值的。

下面这个构造方法是把int数组根据Unicode编码转换成字符串

**public String(int[] codePoints, int offset, int count)**

```java
public String(int[] codePoints, int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count <= 0) {
            if (count < 0) {
                throw new StringIndexOutOfBoundsException(count);
            }
            if (offset <= codePoints.length) {
                this.value = "".value;
                return;
            }
        }
        // Note: offset or count might be near -1>>>1.
        if (offset > codePoints.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }
        //上面是偏移量和长度有问题的条件
        //下面进行int数组和char数组之间的转换
        final int end = offset + count;

        int n = count;
        //下面先计算出结果的char数组的长度
        for (int i = offset; i < end; i++) {
            int c = codePoints[i];
            if (Character.isBmpCodePoint(c))
                continue;
            else if (Character.isValidCodePoint(c))
                n++;
            else throw new IllegalArgumentException(Integer.toString(c));
        }

        //填充char数组
        final char[] v = new char[n];

        for (int i = offset, j = 0; i < end; i++, j++) {
            int c = codePoints[i];
            if (Character.isBmpCodePoint(c))
                v[j] = (char)c;
            else
                Character.toSurrogates(c, v, j++);
        }

        this.value = v;
    }
```

上面出现了Character.isBmpCodePoint(c)、Character.isValidCodePoint(c)和Character.toSurrogates(c, v, j++);方法，对应的代码如下

```java
public static boolean isBmpCodePoint(int codePoint) {
        return codePoint >>> 16 == 0;
    }
public static boolean isValidCodePoint(int codePoint) {
        int plane = codePoint >>> 16;
        return plane < ((MAX_CODE_POINT + 1) >>> 16);
    }
static void toSurrogates(int codePoint, char[] dst, int index) {
        dst[index+1] = lowSurrogate(codePoint);
        dst[index] = highSurrogate(codePoint);
    }
```

isBmpCodePoint(int codePoint)，将数值右移16位，判断是否为0

这里右移16位的原因是，Unicode编码中每个字符对应一个长度为16的二进制字符，这个范围内的字符称为BMP，超出这个范围的叫做增补字符。一般我们使用的字符都是在BMP范围内的，一些特殊的字符，比如日本的字符等，就不属于BMP范围的。在Java中，char是两个字节，正对应上BMP字符的长度(16个比特)，为了支持超出BMP范围的字符，这些字符在Java中是以四个字节的形式进行存放的，isValidCodePoint方法就是用来判断是否是超出了BMP范围的字符，如果是的话，需要额外增加一个两个字节(即增加最终字符串数组的一个长度，体现在n++上，n为最终数组的长度)。

理解了上面这两个之后，下面的赋值操作也容易理解了，如果是BMP范围的，直接进行赋值，对于超出了BMP范围，需要占据两个char的位置来表示该字符。

接下来的构造方法也和上面类似

**public String(byte ascii[], int hibyte, int offset, int count)**

**public String(byte ascii[],int hibyte){this(ascii,hibyte,0,ascii.length);}**

checkBounds方法用于验证指定的偏移量和长度是否越界

**private static void checkBounds(byte[] bytes, int offset, int length)**

```java
private static void checkBounds(byte[] bytes, int offset, int length) {
        if (length < 0)
            throw new StringIndexOutOfBoundsException(length);
        if (offset < 0)
            throw new StringIndexOutOfBoundsException(offset);
        if (offset > bytes.length - length)
            throw new StringIndexOutOfBoundsException(offset + length);
    }
```

**public String(byte bytes[], int offset, int length,String charsetName)**

该构造方法指定了编码方式

**public String(byte bytes[], int offset, int length, Charset charset)**

这个构造方法和上面的相反，这个是指定解码方式进行字符串的解码

**public String(byte bytes[], String charsetName)**

```java
public String(byte bytes[], String charsetName)
            throws UnsupportedEncodingException {
        this(bytes, 0, bytes.length, charsetName);
    }
```

**public String(byte bytes[], Charset charset){this(bytes, 0, bytes.length, charset);}**

**public String(byte bytes[], int offset, int length)**

**public String(byte bytes[])**

上面的构造方法基本都是大同小异，都是调用前面的构造方法

**public String(StringBuffer buffer)**

```java
public String(StringBuffer buffer) {
        synchronized(buffer) {
            this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
        }
    }
```

StringBuffer是线程安全的，利用StringBuffer进行创建String时进行加锁

**public String(StringBuilder builder){this.value = Arrays.copyOf(builder.getValue(), builder.length());}**

用StringBuilder进行创建则没有进行加锁

**String(char[] value,boolean share)**

```java
String(char[] value, boolean share) {
        // assert share : "unshared not supported";
        this.value = value;
    }
```

从注释上可以看到，该方法不支持false参数

该方法的作用为了提高性能，在前面的String(char[] value)中，调用了Arrays.copyOf(value, value.length)方法，使用该方法只需要赋值下应用即可。该方法设置成了protect，可以防止外部调用该方法，然后读字符串进行修改

### 获取属性方法

**public int length(){return value.length;}**

返回字符串长度

**public boolean isEmpty(){return value.length==0;}**

判读是否为空

**public char charAt(int index)**

返回指定位置的字符

```java
public char charAt(int index) {
        if ((index < 0) || (index >= value.length)) {
            throw new StringIndexOutOfBoundsException(index);
        }
        return value[index];
    }
```

**public int codePointAt(int index)**

返回指定位置的字符的Unicode编码值

```java
public int codePointAt(int index) {
        if ((index < 0) || (index >= value.length)) {
            throw new StringIndexOutOfBoundsException(index);
        }
        return Character.codePointAtImpl(value, index, value.length);
    }
```

**public int codePointBefore(int index)**

返回指定位置前一个字符的Unicode编码值

```JAVA
public int codePointBefore(int index) {
        int i = index - 1;
        if ((i < 0) || (i >= value.length)) {
            throw new StringIndexOutOfBoundsException(index);
        }
        return Character.codePointBeforeImpl(value, index, 0);
    }
```

**public int codePointCount(int beginIndex, int endIndex)**

该方法返回的是指定范围内多少个*代码点*，代码点在前面讲到，就是跟字符的长度有关，如果字符串的所有字符都在BMP范围内，则该方法的返回值和length()方法返回值一样，但如果有一些字符超出了BMP范围，则该方法的返回值比length()小，因为一个代码点指的是一个完整的字符，超出BMP范围的需要用2个数组位置才能表示，这两个位置算一个代码点，但是如果在length中，其返回值是2。这里有一个测试例子

```java
public class AppTest {
    public static void main(String[] args) throws InterruptedException {
        int[]nums = {70000};
        String s = new String(nums, 0, nums.length);
        System.out.println(s);
        String temp = "𑅰";
        System.out.println(temp.length());
        System.out.println(temp.codePointCount(0,temp.length()));
    }
}
//控制台打印内容
//𑅰
//2
//1
```

从上面的结果可以看到，长度为9的int数组，在转换成string时，由于70000是超出BMP范围的，因此在底层的时候，char数组是用两个位置进行分配的，因此length的长度返回2，但是codePointCount的返回是1

**public int offsetByCodePoints(int index, int codePointOffset)**

返回从index处偏移codePointOffset个代码点的索引

```java
public int offsetByCodePoints(int index, int codePointOffset) {
        if (index < 0 || index > value.length) {
            throw new IndexOutOfBoundsException();
        }
        return Character.offsetByCodePointsImpl(value, 0, value.length,
                index, codePointOffset);
    }
```

### 数组拷贝方法

**void getChars(char dst[], int dstBegin){System.arraycopy(value, 0, dst, dstBegin, value.length);}**

这个方法是protect的，外界无法调用

**public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin)**

该方法是把该字符串的srcBegin到srcEnd位置的字符复制到dst的dstBegin后面。

```java
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
        if (srcBegin < 0) {
            throw new StringIndexOutOfBoundsException(srcBegin);
        }
        if (srcEnd > value.length) {
            throw new StringIndexOutOfBoundsException(srcEnd);
        }
        if (srcBegin > srcEnd) {
            throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
        }
        System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
    }
```

**public void getBytes(int srcBegin, int srcEnd, byte dst[], int dstBegin)**

**public byte[] getBytes(String charsetName)**

**public byte[] getBytes(Charset charset)**

**public byte[] getBytes()**

这几个getbyte方法和getChars方法类似，不过这里传的是byte数组，前面传的是char数组

### 比较方法

**public boolean equals(Object anObject)**

```java
public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
```

equals方法比较简单，先判断是否是String类型，然后对字符串数组的每个值依次进行比较即可

**public boolean contentEquals(StringBuffer sb){return contentEquals((CharSequence)sb);}**

也是进行内容的比较，后面的contentEquals方法代码如下

根据参数的类型进行内容的比较，如果是AbstractStringBuilder的子类，则判断是不是，StringBuffer类型，如果是的话，则使用synchronized进行加锁后调用nonSyncContentEquals方法进行比较，该方法的实现就是逐个逐个字符进行比较。如果是String类型，则直接调用String的equals方法，不然就逐个逐个进行比较。

```java
public boolean contentEquals(CharSequence cs) {
        // Argument is a StringBuffer, StringBuilder
        if (cs instanceof AbstractStringBuilder) {
            if (cs instanceof StringBuffer) {
                synchronized(cs) {
                   return nonSyncContentEquals((AbstractStringBuilder)cs);
                }
            } else {
                return nonSyncContentEquals((AbstractStringBuilder)cs);
            }
        }
        // Argument is a String
        if (cs instanceof String) {
            return equals(cs);
        }
        // Argument is a generic CharSequence
        char v1[] = value;
        int n = v1.length;
        if (n != cs.length()) {
            return false;
        }
        for (int i = 0; i < n; i++) {
            if (v1[i] != cs.charAt(i)) {
                return false;
            }
        }
        return true;
    }

```

**private boolean nonSyncContentEquals(AbstractStringBuilder sb)**

非同步的字符串内容比较

```java
private boolean nonSyncContentEquals(AbstractStringBuilder sb) {
        char v1[] = value;
        char v2[] = sb.getValue();
        int n = v1.length;
        if (n != sb.length()) {
            return false;
        }
        for (int i = 0; i < n; i++) {
            if (v1[i] != v2[i]) {
                return false;
            }
        }
        return true;
    }
```

**public boolean equalsIgnoreCase(String anotherString)**

该方法也是用于字符串内容的比较，但是这个比较忽略大小写。从下面regionMatches的比较实现中，可以看到既把字符转换成了大写，又转换成了小写进行比较，从注释可以知道有些字符转换成大写会出问题，因此这里做了两次比较。

```java
public boolean equalsIgnoreCase(String anotherString) {
        return (this == anotherString) ? true
                : (anotherString != null)
                && (anotherString.value.length == value.length)
                && regionMatches(true, 0, anotherString, 0, value.length);
    }


public boolean regionMatches(boolean ignoreCase, int toffset,
            String other, int ooffset, int len) {
        char ta[] = value;
        int to = toffset;
        char pa[] = other.value;
        int po = ooffset;
        // Note: toffset, ooffset, or len might be near -1>>>1.
        if ((ooffset < 0) || (toffset < 0)
                || (toffset > (long)value.length - len)
                || (ooffset > (long)other.value.length - len)) {
            return false;
        }
        while (len-- > 0) {
            char c1 = ta[to++];
            char c2 = pa[po++];
            if (c1 == c2) {
                continue;
            }
            if (ignoreCase) {
                // If characters don't match but case may be ignored,
                // try converting both characters to uppercase.
                // If the results match, then the comparison scan should
                // continue.
                char u1 = Character.toUpperCase(c1);
                char u2 = Character.toUpperCase(c2);
                if (u1 == u2) {
                    continue;
                }
                // Unfortunately, conversion to uppercase does not work properly
                // for the Georgian alphabet, which has strange rules about case
                // conversion.  So we need to make one last check before
                // exiting.
                if (Character.toLowerCase(u1) == Character.toLowerCase(u2)) {
                    continue;
                }
            }
            return false;
        }
        return true;
    }
```

**public int compareTo(String anotherString)**

该方法用于两个字符串的比较，返回值是最开始不一样位置的字符的差值，否则就返回两字符串的长度之差

```java
public int compareTo(String anotherString) {
        int len1 = value.length;
        int len2 = anotherString.value.length;
        int lim = Math.min(len1, len2);
        char v1[] = value;
        char v2[] = anotherString.value;

        int k = 0;
        while (k < lim) {
            char c1 = v1[k];
            char c2 = v2[k];
            if (c1 != c2) {
                return c1 - c2;
            }
            k++;
        }
        return len1 - len2;
    }
```



String的内部还实现了一个CaseInsensitiveComparator静态内部类，实现了序列化接口和Comparator接口，用于字符串的比较

```java
public static final Comparator<String> CASE_INSENSITIVE_ORDER
                                         = new CaseInsensitiveComparator();
    private static class CaseInsensitiveComparator
            implements Comparator<String>, java.io.Serializable {
        // use serialVersionUID from JDK 1.2.2 for interoperability
        private static final long serialVersionUID = 8575799808933029326L;

        public int compare(String s1, String s2) {
            int n1 = s1.length();
            int n2 = s2.length();
            int min = Math.min(n1, n2);
            for (int i = 0; i < min; i++) {
                char c1 = s1.charAt(i);
                char c2 = s2.charAt(i);
                if (c1 != c2) {
                    c1 = Character.toUpperCase(c1);
                    c2 = Character.toUpperCase(c2);
                    if (c1 != c2) {
                        c1 = Character.toLowerCase(c1);
                        c2 = Character.toLowerCase(c2);
                        if (c1 != c2) {
                            // No overflow because of numeric promotion
                            return c1 - c2;
                        }
                    }
                }
            }
            return n1 - n2;
        }

        /** Replaces the de-serialized object. */
        private Object readResolve() { return CASE_INSENSITIVE_ORDER; }
    }
```

### 前后缀判断方法

**public boolean startsWith(String prefix, int toffset)**

判断给定字符串prefix是不是本字符串toffset位置后的前缀

```java
public boolean startsWith(String prefix, int toffset) {
        char ta[] = value;
        int to = toffset;
        char pa[] = prefix.value;
        int po = 0;
        int pc = prefix.value.length;
        // Note: toffset might be near -1>>>1.
        if ((toffset < 0) || (toffset > value.length - pc)) {
            return false;
        }
        while (--pc >= 0) {
            if (ta[to++] != pa[po++]) {
                return false;
            }
        }
        return true;
    }
```

**public boolean startsWith(String prefix){return startsWith(prefix, 0);}**

**public boolean endsWith(String suffix){return startsWith(suffix, value.length - suffix.value.length);}**

这两个方法都是调用上面的startsWith方法实现的

**public int hashCode()**

计算字符串的哈希值

```java
public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
```

### 位置查找方法

**public int indexOf(int ch) { return indexOf(ch, 0); }**

获取给定字符的位置

**public int indexOf(int ch, int fromIndex)**

如果给定字符是BMP范围的字符(即一个char位置)，那么只需要一个个进行比较即可，但如果超出了BMP范围，即下面的ch >= Character.MIN_SUPPLEMENTARY_CODE_POINT的情况，那么要调用indexOfSupplementary(ch, fromIndex);该方法进行查找。

```java
public int indexOf(int ch, int fromIndex) {
        final int max = value.length;
        if (fromIndex < 0) {
            fromIndex = 0;
        } else if (fromIndex >= max) {
            // Note: fromIndex might be near -1>>>1.
            return -1;
        }

        if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {
            // handle most cases here (ch is a BMP code point or a
            // negative value (invalid code point))
            final char[] value = this.value;
            for (int i = fromIndex; i < max; i++) {
                if (value[i] == ch) {
                    return i;
                }
            }
            return -1;
        } else {
            return indexOfSupplementary(ch, fromIndex);
        }
    }
```

indexOfSupplementary方法查找方式也是一个for循环，但是因为ch超出了范围，需要使用两个char进行表示，因此需要比较两个位置。

```java
private int indexOfSupplementary(int ch, int fromIndex) {
        if (Character.isValidCodePoint(ch)) {
            final char[] value = this.value;
            final char hi = Character.highSurrogate(ch);
            final char lo = Character.lowSurrogate(ch);
            final int max = value.length - 1;
            for (int i = fromIndex; i < max; i++) {
                if (value[i] == hi && value[i + 1] == lo) {
                    return i;
                }
            }
        }
        return -1;
    }
```

**public int lastIndexOf(int ch) { return lastIndexOf(ch, value.length - 1);}**

**public int lastIndexOf(int ch, int fromIndex)**

**private int lastIndexOfSupplementary(int ch, int fromIndex)**

这几个方法和上面的基本一样



**public int indexOf(String str) { return indexOf(str, 0);}**

**public int indexOf(String str, int fromIndex)**

```java
public int indexOf(String str, int fromIndex) {
        return indexOf(value, 0, value.length,
                str.value, 0, str.value.length, fromIndex);
    }
```

**static int indexOf(char[] source, int sourceOffset, int sourceCount,String target, int fromIndex)**

```java
static int indexOf(char[] source, int sourceOffset, int sourceCount,
            String target, int fromIndex) {
        return indexOf(source, sourceOffset, sourceCount,
                       target.value, 0, target.value.length,
                       fromIndex);
    }
```

具体indexOf实现如下

```java
static int indexOf(char[] source, int sourceOffset, int sourceCount,
            char[] target, int targetOffset, int targetCount,
            int fromIndex) {
        if (fromIndex >= sourceCount) {
            return (targetCount == 0 ? sourceCount : -1);
        }
        if (fromIndex < 0) {
            fromIndex = 0;
        }
        if (targetCount == 0) {
            return fromIndex;
        }

        char first = target[targetOffset];
        int max = sourceOffset + (sourceCount - targetCount);

        for (int i = sourceOffset + fromIndex; i <= max; i++) {
            /* Look for first character. */
            if (source[i] != first) {
                while (++i <= max && source[i] != first);
            }

            /* Found first character, now look at the rest of v2 */
            if (i <= max) {
                int j = i + 1;
                int end = j + targetCount - 1;
                for (int k = targetOffset + 1; j < end && source[j]
                        == target[k]; j++, k++);

                if (j == end) {
                    /* Found whole string. */
                    return i - sourceOffset;
                }
            }
        }
        return -1;
    }
```

**public int lastIndexOf(String str) { return lastIndexOf(str, value.length);}**

**public int lastIndexOf(String str, int fromIndex)**

**static int lastIndexOf(char[] source, int sourceOffset, int sourceCount, String target, int fromIndex)**

**static int lastIndexOf(char[] source, int sourceOffset, int sourceCount, char[] target, int targetOffset, int targetCount, int fromIndex)**

这几个方法也和上面的类似

### 字符串切割方法

**public String substring(int beginIndex)**

该方法在底层是又生成了新的String对象

```java
public String substring(int beginIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        int subLen = value.length - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
    }
```

**public String substring(int beginIndex, int endIndex)**

**public CharSequence subSequence(int beginIndex, int endIndex) { return this.substring(beginIndex, endIndex);}**

### 字符串连接方法

**public String concat(String str)**

```java
public String concat(String str) {
        int otherLen = str.length();
        if (otherLen == 0) {
            return this;
        }
        int len = value.length;
        char buf[] = Arrays.copyOf(value, len + otherLen);
        str.getChars(buf, len);
        return new String(buf, true);
    }
```

在这个方法中通过Arrays.copyOf拷贝出了一个新数组，在new一个String时，调用构造方法是有两个参数的，这个构造方法前面有说到，该构造方法只是把new的String的value数组指向这个新数组，而不是像普通的构造方法一样进行数组的拷贝，可以节省时间。

### 字符替换方法

**public String replace(char oldChar, char newChar)**

实现如下，不过感觉它的代码实现比较奇怪，感觉实现得比较复杂

```java
public String replace(char oldChar, char newChar) {
        if (oldChar != newChar) {
            int len = value.length;
            int i = -1;
            char[] val = value; /* avoid getfield opcode */

            while (++i < len) {
                if (val[i] == oldChar) {
                    break;
                }
            }
            if (i < len) {
                char buf[] = new char[len];
                for (int j = 0; j < i; j++) {
                    buf[j] = val[j];
                }
                while (i < len) {
                    char c = val[i];
                    buf[i] = (c == oldChar) ? newChar : c;
                    i++;
                }
                return new String(buf, true);
            }
        }
        return this;
    }
```

### 字符串匹配方法

**public boolean matches(String regex) { return Pattern.matches(regex, this); }**

**public String replaceFirst(String regex, String replacement) { return Pattern.compile(regex).matcher(this).replaceFirst(replacement); }**

**public String replaceAll(String regex, String replacement) { return Pattern.compile(regex).matcher(this).replaceAll(replacement); }**

**public String replace(CharSequence target, CharSequence replacement) { return Pattern.compile(target.toString(), Pattern.LITERAL).matcher(this).replaceAll(Matcher.quoteReplacement(replacement.toString()));}**

底层都是调用了Pattern类的方法，由于Pattern的源码比较复杂，而且这一篇主要是讲String，再加上目标只是了解下常见类的源码，Pattern的源码放之后，把常见的类的源码看完后再去写

### 字符串切割方法

**public String[] split(String regex, int limit)**

**public String[] split(String regex) { return split(regex, 0);}**



### 连接字符串数组方法

**public static String join(CharSequence delimiter, CharSequence... elements)**

用字符串delimiter把字符数组elements中的所有元素都连接起来

**public static String join(CharSequence delimiter,Iterable<? extends CharSequence> elements)**



### 字符串大小写转换方法

**public String toLowerCase(Locale locale)**

**public String toLowerCase() { return toLowerCase(Locale.getDefault()); }**

**public String toUpperCase(Locale locale)**

**public String toUpperCase() { return toUpperCase(Locale.getDefault()); }**



**public String trim()**

去除字符串前后的空格

```java
public String trim() {
        int len = value.length;
        int st = 0;
        char[] val = value;    /* avoid getfield opcode */

        while ((st < len) && (val[st] <= ' ')) {
            st++;
        }
        while ((st < len) && (val[len - 1] <= ' ')) {
            len--;
        }
        return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
    }
```

**public String toString() { return this; }**

**public char[] toCharArray()**

```java
public char[] toCharArray() {
        // Cannot use Arrays.copyOf because of class initialization order issues
        char result[] = new char[value.length];
        System.arraycopy(value, 0, result, 0, value.length);
        return result;
    }
```

**public static String format(String format, Object... args) **

```java
public static String format(String format, Object... args) {
        return new Formatter().format(format, args).toString();
    }
```

该方法和C语言中的printf "%s" "%d"这些差不多，把format字符串中的对应的%d等符号和args中的值对应进行替换。

**public static String valueOf(Object obj) { return (obj == null) ? "null" : obj.toString(); }**

**public static String valueOf(char data[]) { return new String(data); }**

**public static String valueOf(char data[], int offset, int count) { return new String(data, offset, count); }**

**public static String copyValueOf(char data[], int offset, int count) { return new String(data, offset, count); }**

**public static String copyValueOf(char data[]) { return new String(data); }**

**public static String valueOf(boolean b) { return b ? "true" : "false"; }**

**public static String valueOf(char c) { char data[] = {c}; return new String(data, true); }**

**public static String valueOf(int i) { return Integer.toString(i); }**

**public static String valueOf(long l) { return Long.toString(l); }**
**public static String valueOf(float f) { return Float.toString(f); }**
**public static String valueOf(double d) { return Double.toString(d); }**

上面这几个方法都是把对应的类型转换成成字符串类型。

**public native String intern();**

intern方法返回该字符在字符串常量池的引用