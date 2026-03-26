# String

String对象一旦被创建 在内存里的值就不可辨。

```java
String s = "Hello";
s = s + " World"; 
System.out.println(s); // 输出 Hello World
```

实际上这里的s只是一个引用 并没有改变string在内存里的值。 当执行s+"xxx"  是在堆内存里重新开辟空间 存入 整个字符串 然后修改s的指向。



## string为什么是不可变得

1. 天生线程安全 如果无法被修改 则多线程并发读安全
2. 安全性 网络连接url 数据库账号密码是string 如果string可以被修改 不安全
3. 是实现字符串常量池复用的前提

## 字符串常量池

1. 字面量赋值

```java
String s1 = "Java";
String s2 = "Java";
```

当jvm看到"java" 会去字符串常量池寻找 找到了就指向 找到不到就创建

所以s1==s2 指向内存同一个对象 节约内存

2. new关键字

```java
String s3 = new String("Java");
```

只要看到new关键字 就回去堆内存里开辟一个新空间 创建一个全新的String对象 同时也会去常量池看有没有 没有顺手创建

## StringBuilder

既然字符串不可变 如果在for循环里拼接字符串 就会产生很多的中间对象 撑爆内存 不断触发gc 导致内存卡顿

- StringBuilder 可变得 调用append()方法 是直接在原有内存空间上修改内容 不产生中间对象
- StringBuffer 所有方法都加了synchronized锁 线程安全



## 编译器优化

```java
String c = "HelloWorld"; String d = "Hello" + "World";
```

+号不会产生新的对象 javac在编译器看到两个纯粹的字符串字面量+在一起 则值永远不会变 则直接合并了 所以c==d

## 运行期拼接

```java
String a = "Hello";
String b = "World";
String c = "HelloWorld";
String e = a + b;
```

e的值要到运行的时候才知道 a和b是什么 是变量 对于javac来说 变量的值在运行前是不可知的。

在jvm运行期 java底层会new一个StringBuilder 然后调用append(a) append(b) 然后使用toString-->String

只要调用了new 就在堆内存里开辟一个全新的 所以c!=e



## Srting.intern()

强行把字符串塞进常量池 返回常量池里的内容。

对一个字符串对象s调用s.intern jvm会去查看StringPool里有没有 底层调用equals看内容是否相同

- 如果有了 返回StringPool里的地址
- 如果没有 就在StringPool里创建【JDK7后会把堆里原本那个s对象的引用放进常量池里 然后返回这个常量池里的地址

调用完不会改变s本来指向的地址 是返回一个StringPool里的地址 s原本如果指向heap 调用后还是。

### 为什么要有 `intern()`？（架构师视角）

假设你的系统要从数据库里读取 100 万个用户的“所在省份”。因为是从数据库读出来的，Java 底层会在**堆内存**里 `new` 出 100 万个 String 对象。 但实际上全中国只有 30 多个省份，这 100 万个对象里有大量内容是重复的（比如有 10 万个 `"浙江省"`）。这就造成了极其恐怖的内存浪费！

**破局方法：** 在读取的时候调用一下 `intern()`：`String province = rs.getString("province").intern();` 这样，虽然数据库读出了 100 万次，但最终这 100 万个变量都会指向常量池里仅有的 30 多个地址。**堆内存里的那 100 万个废弃对象就会被 GC（垃圾回收器）迅速清理掉**。内存占用瞬间降低 99%！