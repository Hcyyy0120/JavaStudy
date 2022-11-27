---
`title`: JVM系列-第7章-对象的实例化内存布局与访问定位
tags:
  - JVM
  - 虚拟机
categories:
  - JVM
  - 1.内存与垃圾回收篇
keywords: JVM，虚拟机。
description: JVM系列-第7章-对象的实例化内存布局与访问定位。
cover: 'https://npm.elemecdn.com/lql_static@latest/logo/jvm.png'
abbrlink: debff71a
date: 2020-11-14 19:38:42
---



对象的实例化内存布局与访问定位
======================

对象的实例化
--------

**大厂面试题**

美团：

1.  对象在`JVM`中是怎么存储的？
2.  对象头信息里面有哪些东西？



蚂蚁金服：

二面：`java`对象头里有什么



<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_007/0001.png">



### 对象创建的方式

1.  new：最常见的方式、单例类中调用getInstance的静态类方法，XXXFactory/XXXBuilder的静态方法
2.  Class的newInstance方法：在JDK9里面被标记为过时的方法，因为只能调用空参构造器，并且权限必须为 public
3.  Constructor的newInstance(Xxxx)：反射的方式，可以调用空参的，或者带参的构造器
4.  使用clone()：不调用任何的构造器，要求当前的类需要实现Cloneable接口中的clone方法
5.  使用序列化：从文件中，从网络中获取一个对象的二进制流，序列化一般用于Socket的网络传输
6.  第三方库 Objenesis

```java
public class User implements Cloneable {
	private String name;
    private Integer age;
    
    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}


public class Test {
	public static void main(String[] args) throws Exception {
        User user1 = (User) Class.forName("com.hcy.User").newInstance();
        User user2 = (User) User.class.getConstructors()[0].newInstance();
        User user3 = new User();
        User user4 = (User) user3.clone();
    }
}

```



### 对象创建的步骤

> **从字节码看待对象的创建过程**

```java
public class ObjectTest {
    public static void main(String[] args) {
        Object obj = new Object();
    }
}
```



```
 public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #2                  // class java/lang/Object
         3: dup           
         4: invokespecial #1                  // Method java/lang/Object."<init>":()V
         7: astore_1
         8: return
      LineNumberTable:
        line 9: 0
        line 10: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
            8       1     1   obj   Ljava/lang/Object;
}
```



#### **1、判断对象对应的类是否加载、链接、初始化**

1.  虚拟机遇到一条new指令，首先去检查这个指令的参数能否在Metaspace的运行时常量池中定位到一个类的符号引用（`class文件中的常量池在类加载后存放到方法区的运行时常量池中`），并且检查这个符号引用代表的类是否已经被加载，解析和初始化。（即判断类元信息是否存在）。
2.  如果该类没有加载，那么在双亲委派模式下，使用当前类加载器以ClassLoader + 包名 + 类名为key进行查找对应的.class文件，如果没有找到文件，则抛出ClassNotFoundException异常，**如果找到，则进行类加载**，并生成对应的Class对象。



#### **2、为对象分配内存**

1.  首先计算对象占用空间的大小，接着在堆中划分一块内存给新对象（`对象所需的内存大小在类加载完成后便可确定`）。如果实例成员变量是引用变量，仅分配引用变量空间即可，即4个字节大小
2.  如果内存规整：采用指针碰撞分配内存
    *   如果内存是规整的，那么虚拟机将采用的是指针碰撞法（Bump The Point）来为对象分配内存。
    *   **指针碰撞法**：意思是所有用过的内存在一边，空闲的内存放另外一边，中间放着一个指针作为分界点的指示器，分配内存就仅仅是把指针往空闲内存那边挪动一段与对象大小相等的距离罢了。
    *   如果垃圾收集器选择的是Serial ，ParNew这种基于压缩算法的，虚拟机采用这种分配方式。一般使用带Compact（整理）过程的收集器时，使用指针碰撞。
    *   标记压缩（整理）算法会整理内存碎片，堆内存一存对象，另一边为空闲区域
3.  如果内存不规整
    *   如果内存不是规整的，已使用的内存和未使用的内存相互交错，那么虚拟机将采用的是空闲列表来为对象分配内存。
    *   **空闲列表**：意思是虚拟机维护了一个列表，记录上哪些内存块是可用的，再分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的内容。这种分配方式成为了 “空闲列表（Free List）”

选择以上两种方式中的哪一种，取决于 Java 堆内存是否规整。而 Java 堆内存是否规整，取决于 GC 收集器的算法是"标记-清除"（标记清除算法清理过后的堆内存，会存在很多内存碎片），还是"标记-整理"（也称作"标记-压缩"），值得注意的是，复制算法内存也是规整的



#### **3、处理并发问题**（2、3可合到一块讲，分配内存时发生的并发问题）

在创建对象的时候有一个很重要的问题，就是线程安全，因为在实际开发过程中，创建对象是很频繁的事情，**例如**多个并发执行的线程创建对象，分配内存的时候，有可能在Java堆中的同一个位置申请（也就是并发的时候同时有两个不同的对象分配的内存是同一块内存），这就需要对这部分内存空间进行加锁或者采用CAS等操作保证线程安全，即保证该区域只分配给一个线程。作为虚拟机来说，必须要保证线程是安全的，通常来讲，虚拟机采用两种方式来保证线程安全：

+ **CAS+失败重试：** CAS 是乐观锁的一种实现方式。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。**虚拟机采用 CAS 配上失败重试的方式保证更新操作的原子性。**
+ **TLAB：** 为每一个线程预先在 Eden 区分配一块内存，JVM 在给线程中的对象分配内存时，首先在 TLAB 分配，当对象大于 TLAB 中的剩余内存或 TLAB 的内存已用尽时，再采用上述的 CAS +失败重试进行内存分配



#### **4、初始化分配到的空间/初始化零值（默认初始化）**

- 所有属性设置默认值，保证对象实例字段在不赋值可以直接使用

- 给对象属性赋值的顺序：

1.  属性的默认值初始化
2.  显示初始化/代码块初始化（并列关系，谁先谁后看代码编写的顺序）
3.  构造器初始化



#### **5、设置对象的对象头**

将对象的所属类（即类的元数据信息）、对象的HashCode和对象的GC信息、锁信息等数据存储在对象的对象头中。这个过程的具体设置方式取决于JVM实现。

对象头由类的元数据信息和类型指针（指向方法区的类元信息）组成（如果是数组还有数组长度）



#### **6、执行init方法进行初始化（自定义初始化）**

1.  在上面工作都完成之后，从虚拟机的视角来看，一个新的对象已经产生了，在Java程序的视角看来，初始化才正式开始。初始化成员变量，执行实例化代码块，调用类的构造方法，并把堆内对象的首地址赋值给引用变量
  
2.  因此一般来说（由字节码中跟随invokespecial指令所决定），new指令之后会接着就是执行init方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完成创建出来。




> **从字节码角度看 init 方法**

```java
/**
 * 测试对象实例化的过程
 *  ① 加载类元信息 - ② 为对象分配内存 - ③ 处理并发问题  - ④ 属性的默认初始化（零值初始化）
 *  - ⑤ 设置对象头的信息 - ⑥ 属性的显式初始化、代码块中初始化、构造器中初始化
 *
 *
 *  给对象的属性赋值的操作：
 *  ① 属性的默认初始化 - ② 显式初始化 / ③ 代码块中初始化 - ④ 构造器中初始化
 */

public class Customer{
    int id = 1001;
    String name;
    Account acct;

    {
        name = "匿名客户";
    }
    public Customer(){
        acct = new Account();
    }

}
class Account{

}
```



**Customer类的字节码**

```java
 0 aload_0
 1 invokespecial #1 <java/lang/Object.<init>>
 4 aload_0
 5 sipush 1001
 8 putfield #2 <com/atguigu/java/Customer.id>
11 aload_0
12 ldc #3 <匿名客户>
14 putfield #4 <com/atguigu/java/Customer.name>
17 aload_0
18 new #5 <com/atguigu/java/Account>
21 dup
22 invokespecial #6 <com/atguigu/java/Account.<init>>
25 putfield #7 <com/atguigu/java/Customer.acct>
28 return
```



*   init() 方法的字节码指令：
    *   属性的默认值初始化：`id = 1001;`
    *   显示初始化/代码块初始化：`name = "匿名客户";`
    *   构造器初始化：`acct = new Account();`

    
    

对象的内存布局
---------

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_007/0002.png">

解释：

类型指针，即对象指向它的类元数据的指针，**虚拟机通过这个指针来确定这个对象是哪个类的实例（obj.getClass(）)**（并不是所有的虚拟机实现都必须在对象数据上保留类型指针，也就是说，查找对象的元数据信息并不一定要经过对象本身）

**实例数据部分是对象真正存储的有效信息**，也是在程序中所定义的各种类型的字段内容

**对齐填充部分不是必然存在的，也没有什么特别的含义，仅仅起占位作用。** 因为 Hotspot 虚拟机的自动内存管理系统要求对象起始地址必须是 8 字节的整数倍，换句话说就是对象的大小必须是 8 字节的整数倍。而对象头部分正好是 8 字节的倍数（1 倍或 2 倍），因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。

#### **内存布局总结**

```java
public class Customer{
    int id = 1001;
    String name;
    Account acct;

    {
        name = "匿名客户";
    }
    public Customer(){
        acct = new Account();
    }
	public static void main(String[] args) {
        Customer cust = new Customer();
    }
}
class Account{

}
```

#### 图解内存布局

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_007/0003.png">



对象的访问定位
---------

**JVM是如何通过栈帧中（局部变量表）的对象引用访问到其内部的对象实例呢？**

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_007/0004.png">

定位，通过栈上reference访问

**对象的两种访问方式：句柄访问和直接指针**

**1、句柄访问**

1.  缺点：在堆空间中开辟了一块空间作为句柄池，句柄池本身也会占用空间；通过两次指针访问才能访问到堆中的对象（reference先指向句柄池，句柄池中再指向堆中对象实例），效率低
2.  优点：reference中存储稳定句柄地址，对象被移动（垃圾收集时移动对象很普遍）时只会改变句柄中实例数据指针即可，reference本身不需要被修改

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_007/0005.png">



**2、直接指针（HotSpot采用）**

1.  优点：直接指针是局部变量表中的引用，直接指向堆中的实例，在对象实例中有类型指针，指向的是方法区中的对象类型数据
2.  缺点：对象被移动（垃圾收集时移动对象很普遍）时需要修改 reference 的值

<img src="https://npm.elemecdn.com/youthlql@1.0.8/JVM/chapter_007/0006.png">