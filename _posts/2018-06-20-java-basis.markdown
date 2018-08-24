---
layout:    post
title:      "Java基础"
subtitle:   "Basic Knowleage of Java"
date:       2018-06-20 12:00:00
author:     "scyhssm"
header-img: "img/Java.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Java
---

> 这是根据网上的Java基础内容结合自己的缺点记录下来的，用于回顾使用

1.基本类型boolean/1,byte/8,char/16,short/16,int/32,float/32,long/64,double/64.基本类型都有对应的包装类型，基本类型和对应包装类型之间的赋值用自动装箱与拆箱完成。

2.new Integer(123)每次都新建对象，Integer.valueOf(123)可能使用缓存对象，多次使用Integer.valueOf(123)会取得同一个对象的引用。

valueOf(int i)源码实现：
```
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

Java8中Integer缓存池默认大小为-128～127

其他放在缓存池中的类型：
```
boolean true and false
all bytes values
short values between -128 and 127
int values between -128 and 127
char in the range \u0000 to \u007f
```

使用这些基本类型对应的包装类型可以直接使用缓冲池对象，因此Integer.valueOf(123)在范围内可以直接使用。

3.String被声明为final，不可继承。内部使用char数组存储数据，数组也被声明为final，value数组初始化后不能再引用其他数组，如果是通过某个数组创建的String，会有构造函数来拷贝数组构造：
```
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);
}
```

4.String不可变的好处：
* 缓存Hash值用于计算，因为String的Hash值经常被使用，String用作HashMap的Key。要求Hash值不变。
* 如果String对象已经被创建过了，再次创建相同对象就会从String Poll中取得引用。要求String不可变，才可以使用Pool。
* 安全，String经常作为参数，String不可变可以保证参数的不可变。比如在网络连接参数情况下String可变那么在网络连接过程中String被改变，改变String对象的一方以为现在连接的是其他主机，实际情况却不一定。
* String不可变使得其具备线程安全能在多个线程中安全使用。

5.String不可变，StringBuffer和StringBuilder可变。String不可变因此线程安全，StringBuilder不是线程安全的，StringBuffer是线程安全的，内部用synchronized同步。

6.String创建和Integer创建类似，如果是String s = new String(“123”)方式，就会创建一个新的String，拥有一个独一无二的final char[]数组，而String s = “123”方式会把String自动放到String Pool中，下回再用String ss = “123”时会自动引用String Pool中的对象。如果我们想指定从String Pool中获取对象，使用s.intern方法返回对象。

7.Java 7前，字符串常量池被放在永久代中，在7及后，字符串常量池被放在堆中。

8.Java传递值是通过传递对象的地址给形参，形参不复制对象但是复制了指向对象的引用。基本类型不是对象，直接复制值。

9.float f = 1.1;这句赋值语句中1.1是double类型,这么写是向下转型编译器会报错，因此要写成float f = 1.1f;

10.同理short和int也是，1为int型，不过可以直接short h = 1;赋值，但是我们要知道，在使用h += 1是实际上执行的是h = (short)(h+1);带上了向下转型。

11.从Java 7开始，可以在switch条件判断语句中使用String对象。

12.成员可见表示其他类可以用这个类的实例对象访问这个成员。类可见表示其他类可以用这个类创建实例。

13.protected只能修饰成员，表示只有子类和当前类能够访问到成员。子类的方法覆盖父类方法，那么这个方法的访问级别必须高于父类。假如说父类有一个方法修饰级别为protected，子类访问级别低于父类为private，那么如果向上转型用父类指向子类时调用父类方法，子类方法无法被访问。

14.4个级别public，default，protected，private。

15.声明类有final表示不可被继承修改，对于基本类型，final使其数值不变，对于引用类型，final使引用不变，对于方法，final使方法不能被子类重写，private方法隐式指定为final，子类如果定义一个和private方法签名相同的方法，那么该方法不是重写是定义的新方法，final对于类使其不允许被继承；static修饰类只有一种情况即静态内部类，一般是在外部类需要使用内部类而内部类无需外部类资源，内部类能够单独创建的时候会考虑，非静态内部类依赖于需要外部类的实例，而静态内部类不需要，静态内部类不能访问外部类的非静态变量和方法。default：在同一包内，不实用任何修饰符。使用对象：类、接口、变量、方法；private：在同一类内，使用对象：变量、方法。注意：不能修饰类（外部类）；public：对所有类可见，使用对象：类、接口、变量、方法；protected：对同一包内的类及所有子类（包括孙类）可见，使用对象：变量、方法。不能修饰类（外部类）。

静态内部类范例：
```
public class OuterClass {
    class InnerClass {
    }

    static class StaticInnerClass {
    }

    public static void main(String[] args) {
        // InnerClass innerClass = new InnerClass(); // 'OuterClass.this' cannot be referenced from a static context
        OuterClass outerClass = new OuterClass();
        InnerClass innerClass = outerClass.new InnerClass();
        StaticInnerClass staticInnerClass = new StaticInnerClass();
    }
}
```

16.字段不能是共有的，这么做失去对该字段修改行为的控制，客户端可以随意修改，可以使用getter和setter方法替换公有字段。也可以是包级私有类中公有的成员，如：
```
public class AccessWithInnerClassExample {
    private class InnerClass {
        int x;
    }

    private InnerClass innerClass;

    public AccessWithInnerClassExample() {
        innerClass = new InnerClass();
    }

    public int getValue() {
        return innerClass.x; // 直接访问
    }
}
```

17.类中如果有抽象方法，那么这个类必须为抽象类，反之不成立。抽象类可以被抽象子类继承。

18.从Java8开始，接口可以有默认的方法实现。接口所有的字段和方法都是public的，不能有private或者protected的，仔细想如果不是的话方法和字段在多次继承中会失去作用无法被使用。接口字段默认是static和final的。

19.抽象类要求子类必须能够替换掉父类对象，接口像是一种方法契约。如果使用抽象类需要几个相关类中共享代码，需要控制继承来的成员访问权限，需要继承非静态和非常量字段。使用接口需要让不相关的类实现一个方法，比如compareTo，需要多重继承的情况。

20.创建子类对象的时候首先调用父类构造器，但是没有创建父类对象，只是调用父类构造方法初始化属性。反编译字节码可以发现只有一个new指令，也就是说开辟了一块空间存放了一个对象。字节码中会有一个u2类型的父类索引，属于CONSTANT_Class_info类型，通过CONSTANT_Class_info的描述可以找到CONSTANT_Class_info的描述，然后找到CONSTANT_utf8_info，然后能够找到指定的父类。方法属性名都是从这里解析出来的，实际变量内容存储在new出来的空间里。super只是访问了空间特定部分的数据（专门存储父类数据的内存部分）。Java虚拟机有个静态类型（外观类型）和实际类型的概念，Object t = new Point(2,3)，Object是静态类型（外观类型），Point是实际类型。

21.重写在继承体系中指子类实现了一个与父类在方法声明上完全相同的一个方法；重载存在于同一个类中，指一个方法与已存在的方法名称相同但是参数类型、个数、顺序至少有一个不同，返回值不同不能算。其他的不同都是在调用时可以确定下来的，而返回值不同会混淆，即传入参数却不知道调用哪个方法。

22.Object通用方法，getClass,hashCode,equals,clone,toString,notify,notifyAll,wait,finalize。

23.基本类型通过==判断值相等，引用类型通过==判断是否引用同一个对象，equals判断引用的对象是否等价，看源码可以知道equals也是首先判断两个对象是否来自一个引用。

24.hasCode返回散列值，equals判断实例是否等价。等价的实例散列值一定要相同，但是散列值相同的两个实例不一定等价。因此在覆盖equals方法时还要覆盖hasCode方法，保证等价的值散列值也相等，这是散列函数的特性要求的。

25.toString默认返回  类名@散列码的形式，如ToStringExample@4554617c，散列码为无符号十六进制。

26.可以显式重写clone方法用以给其他类调用，注意需要加上Cloneable接口才可以否则会报CloneNotSupportedException错误,可以进入Cloneable实现中去看，并没有指定clone方法，Cloneable接口只是一个规定。clone方法拷贝一个对象既复杂又有风险，容易抛出异常还通常需要类型转换。通常在需要复制的时候使用拷贝构造函数，即构造函数中传入需要复制的对象进行复制。

27.静态导包可以在使用静态变量和方法时不再指明ClassName。

28.初始化顺序
* 父类（静态变量、静态语句块）
* 子类（静态变量、静态语句块）
* 父类（实例变量、普通语句块）
* 父类（构造函数）
* 子类（实例变量、普通语句块）
* 子类（构造函数）

29.一个类对应一个Class对象，编译新类会产生一个同名的.class文件，该文件内容保存了Class对象。类加载相当于加载Class对象。Class和java.lang.reflect一起对反射提供了支持，java.lang.reflect包含了以下三个类，Field，Method和Constructor分别是字段、方法和创建新对象。

30.Java和C++区别
* Java纯面向对象语言，所有对象继承自Object，C++为了兼容C既支持面向对象也支持面向过程
* Java通过虚拟机实现跨平台，C++依赖特定平台
* Java没有指针，但是可以将引用理解为安全指针
* Java支持自动垃圾回收，C++需要手动回收，所以C++内存溢出是个大问题
* Java不支持多重继承，但可以实现多个接口达到目的
* Java不支持操作符重载，+=都是语言内置的语法糖，不属于操作符重载
* Java内置对线程的支持，C++需要依赖第三方库
* Java的goto是保留字，不可用，C++可以用goto
* Java不支持条件编译，C++通过#ifdef #ifndef等预处理命令实现条件编译

31.位运算>>如果计算的是负数，则>>相当于将二进制数值移动几位，高位用1代替，如果计算的是0，则>>相当于将二进制数值移动几位，高位用0代替，负数用补码表示，计算更方便直接可以得出结果。>>>这个操作比较特殊，由于忽略符号位，将二进制数值向右移动几位，高位用0补。那么就有一个疑问，既然如此，那么直接用>>代替>>>怎么样？答案是可以但是如果是无符号的数据，少一位符号位，可以多出一倍的数据量，那么>>>不计算符号位的位运算优势更大。如果要延伸开去的话，就是为什么负数用补码了，计算机原理中的东西，如果是补码的话，补码直接用于加减运算，加快了运算速度。

32.子类覆盖父类方法抛出异常不能比父类多。父类抛出的是ParentException，子类抛出的是ChildException。
