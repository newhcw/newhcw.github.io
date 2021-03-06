# JVM堆内存优化技术:指针压缩

```markdown
想要将知识转化为能力,第一步进行知识拆解,第二步,进行验证应用.
```

## 前置知识

1. oop是什么

   oop, 原意object origin pointer, 原始对象指针.对应虚拟机中的kclass地址,可以理解为对象在堆内存中的地址指针.

2. 寻址空间

   32位cpu架构,最大寻址内存为4g(2的32次方),64位cpu架构,寻址内存为TB级别(2的64次方)

3. 对象的内存布局

   对象在堆内存中 = 对象头+实例数据+对齐补充.
   对象头= markword + kclass地址(即oop),
   对象在内存中所占内存是8个倍数，所以有对齐补充。

4. markword 占多少内存

   markword就是 锁的标记,锁分类,gc年龄等,32位默认是4字节,64位默认是8字节.

5. kclass地址是占用多少内存

   kclass是堆内存地址,在没有开启指针压缩的情况下,32位jvm中,是4字节,64位jvm中,8字节.那么可以推出这样的结论,jvm由32位升级为64位,堆内存可以设置更大了,同时对象的引用地址(oop)消耗的内存也增大了.进而可以推出对象的引用地址增大了,其他地方的可用内存就变小了.

   

   那么有方法可以解决将jvm的堆内存增大的同时,还保留了32位jvm的引用地址占用内存小的特点吗?

   由此,引用**指针压缩**技术诞生了.



## 为什么使用指针压缩技术



1. 结论一：jdk7以后,默认开启指针压缩。
   可以通过打开jvm启动参数`-XX:+PrintCompressedOopsMode `验证
2. 结论二：指针压缩技术开启，可以减少对象在堆内存的指针地址消耗。

## 验证结论



测试环境： 

- MacBook pro 64位 cpu
-  jdk13 64位



这里我们计算integer的大小。

1. jvm默认启动方式，即开启指针压缩。

   int 基础类型=4字节，integer对象会在堆中分配内存，integer大小= 8字节（markword）+4字节（压缩后kclass地址）+4字节（int数据）=16字节

   即：Integer期望内存大小为16字节，下面我们运行程序看看。

```xml
  <dependency>
            <groupId>org.apache.lucene</groupId>
            <artifactId>lucene-core</artifactId>
            <version>4.0.0</version>
  </dependency>
```



```java

public class ObjectSize {

    public static void main(String[] args) {
        Integer a = 12;//占据16字节 = 4 + 8 对象头+4个字节int
      System.out.println(RamUsageEstimator.shallowSizeOf(a)+"字节");
    }
}
```

```shell
[0.010s][warning][arguments] -XX:+PrintCompressedOopsMode is deprecated. Will use -Xlog:gc+heap+coops=info instead.
[0.037s][info   ][gc,heap,coops] Heap address: 0x0000000081a00000, size: 2022 MB, Compressed Oops mode: 32-bit
16字节 
```



2. 关闭指针压缩，Integer的内存大小。
   integer大小= 8字节（markword）+8字节（未压缩后kclass地址）+4字节（int数据）+ 4字节（padding补齐）=24字节

`关闭指针压缩：-XX:-UseCompressedOops`

打印台如下

```shell
[0.009s][warning][arguments] -XX:+PrintCompressedOopsMode is deprecated. Will use -Xlog:gc+heap+coops=info instead.
24字节
```



由此，结论也被验证过了，记住它吧。