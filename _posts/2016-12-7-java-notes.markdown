---
layout: post
title:  "Thinking in Java第4版学习笔记"
subtitle:  " \"the notes of Thinking in Java(4th)\""
date:   2016-12-7 10:40:18 +0800
tags:
    - java
author: Nian Tianlei
header-img: "img/post-bg-2016.jpg"
---

> This document is not completed and will be updated in the future.

前段时间看了*Thinking in Java*和*疯狂java讲义精粹*,学到了不少东西，虽然有一点java基础但也都忘差不多了。这次学完，基本掌握了java的基础语法。方便以后进一步的学习。  
在*Thinking in java*这本书的后面几章，我个人觉得有点难，所以我参考了*疯狂java讲义精粹*。  
总之，两本书我都看完了，基础部分我重点看的前者，后面的知识重点看后者。如果要比较的话，前者例子多，但知识讲的比较抽象，重点是在传授一种编程的思想，我觉得适合有一定基础的人看，尤其最后几章；后者例子少，但都是帮助理解的例子，而且知识比较系统，容易找，方便记忆，适合新手吧。  
本笔记只记录一些难记忆或难理解的内容。  


* 基础
  * [类和对象](#class-and-object)
  * [数组](#array)
  * [访问权限](#access-authority)
* 面向对象
	* [equals和==](#equals)
	* [string、stringbuffer、stringbuilder](#string-stringbuffer-stringbuilder)
	* [变量](#variable)
	* [this](#this)
	* [super](#super)
	* [重载和重写](#overload-and-override)
	* [继承](#inheritance)
	* [多态](#polymorphism)
	* [static](#static)
	* [初始化](#initialization)
	* [final、finalize、finally](#final-finalize-finally)
	* [抽象类及接口](#abstract-class-and-interface)
* 容器
	* [collection](#collection)
  * [map](#map)
* 异常
  * [异常](#exception)
* 注解
  * [基本注解](#annotation)
* 并发
  * [并发](#concurrency)
  * [线程同步](#synchronize)
  * [线程通信](#communication)
  * [线程死锁](#deadlock)
* 难点
  * [sleep和yield](#sleep-and-yield)
  * [sychronized和lock](#synchronized-and-lock)


#### class and object

<b>类：</b>是模版，描述一类对象的行为和状态。  
<b>对象：</b>是类的一个实例。  
类不占用内存空间，仅定义变量，必须使用new完成内存的分配。  
java将具有相似功能的方法定义一个类。一个源文件只能包含一个公共类，多个源文件可位于一个包。  
<b>API</b>（Application Programming Interface）：应用程序编程接口。  
面向对象三大特性：封装、多态、继承  

#### array

数组的静态定义:  
`int Array[]={1,2,3}`  
数组的动态定义：  
`int Array[]=new Array[3];`  

#### access authority

public:公有的，对所有类可见   
protected:受保护的，对同一包内的类及其所有子类可见  
default:默认不加修饰符，在同一包内可见  
private:私有的，仅在同一类中可见  
protected不能修饰类和接口。  

<p id = "equals"></p>
#### equals and ==

==:若两个变量是基本类型变量，则比较数值（不要求数据类型相同）；若是引用类型变量，则检查是否指向同一对象。  
equals:仅用于引用变量。若类中没有重写该方法则和==一样，判断是否指向同一对象，例如Object。String重写了equals方法，基本类似于“值相等”。

#### String StringBuffer StringBuilder

String:字符串常量，不可改变。对String的操作会生成新的String对象，效率低，耗费内存。  
StringBuffer:不生成新对象，默认分配16字节的缓冲区，长度可改变。线程安全。  
StringBuilder:同StringBuffer，方法和构造器基本相同，但线程不安全。

#### variable

<b>成员变量：</b>类变量、实例变量  
<b>局部变量：</b>方法变量、块变量  
<b>类变量：</b>需要用satic关键字修饰，在类定义后存在，占用内存空间，可通过类名访问，不需要实例化。  
<b>实例变量：</b>实例化后才会分配内存空间，才可以访问。  
<b>方法变量：</b>定义在方法内的变量  
<b>块变量：</b>定义在一个块内部的变量，变量的生存周期就是这个块  
<b>说明：</b>  
a.方法内部除了能访问方法级的变量，还可以访问类级和实例级的变量  
b.块内部能够访问类级、实例级变量，如果块被包含在方法内部，它还可以访问方法级的变量  
c.方法级和块级的变量必须被显示地初始化，否则不能访问。其中成员变量无需显示初始化，系统会赋予其初值（除了final修饰的变量，后文解释）。

#### this

1.使用this区分同名变量  
  成员变量与方法内部的变量重名时，希望在方法内部调用成员变量，这时候只能使用this访问成员变量  
2.作为方法名来初始化对象  
  相当于调用本类的其他构造方法。  
<b>说明：</b>  
  a.在构造方法中调用另一个构造方法，调用动作必须置于最起始的位置  
  b.不能在构造方法以外的任何方法内调用构造方法  
  c.在一个构造方法内只能调用一个构造方法  
3.作为参数传递  
  将当前对象的一个引用作为参数传递或者返回  


#### super

super关键字与this类似，this用来表示当前类的实例，super用来表示父类  
<b>功能：</b>  
1.调用父类中声明为private的变量  
2.点取已经覆盖了的方法  
3.作为方法名表示父类构造方法    

注：a.this和super不能出现在static修饰方法中。    
　　b.子类构造器无super和this时，系统会在执行子类构造器之前，隐式调用父类无参构造器。

#### overload and override

###### 重载
重载就是在一个类中，有相同的方法名称，但参数列表不同的方法。  
<b>说明：</b>  
  a.参数列表不同包括：个数不同、类型不同和顺序不同  
  b.仅仅参数变量名称不同是不可以的  
  c.跟成员方法一样，构造方法也可以重载  
  d.声明为final的方法不能被重载  
  e.声明为static的方法不能被重载，但是能够被再次声明  
<b>方法的重载的规则：</b>  
  a.方法名称必须相同  
  b.参数列表必须不同（个数不同、或类型不同、参数排列顺序不同等）  
  c.方法的返回类型可以相同也可以不相同  
  d.仅仅返回类型不同不足以成为方法的重载

###### 重写
  a.重写方法的返回类型、方法名称、参数列表必须与原方法的相同  
  b.重写方法不能比原方法访问性差（即访问权限不允许缩小）  
  c.重写方法不能比原方法抛出更多的异常  
  d.被重写的方法不能是final类型，因为final修饰的方法是无法重写的  
  e.被重写的方法不能为private，否则在其子类中只是新定义了一个方法，并没有对其进行重写  
  f.被重写的方法不能为static。如果父类中的方法为静态的，而子类中的方法不是静态的，但是两个方法
    除了这一点外其他都满足重写条件，那么会发生编译错误；反之亦然。即使父类和子类中的方法都是静态的，并且满足重写条件，但是仍然不会发生重写，因为静态方法是在编译的时候把静态方法和类的引用类型进行匹配。

###### 重载与重写的区别
  a.方法重写要求参数列表必须一致，而方法重载要求参数列表必须不一致。  
  b.方法重写要求返回类型必须一致，方法重载对此没有要求。  
  c.方法重写只能用于子类重写父类的方法，方法重载用于同一个类中的所有方法（包括从父类中继承而来的方法）。  
  d.方法重写对方法的访问权限和抛出的异常有特殊的要求，而方法重载在这方面没有任何限制。  
  e.父类的一个方法只能被子类重写一次，而一个方法可以在所有的类中可以被重载多次。

#### inheritance

子类可以覆盖父类的方法  
子类可以继承父类除private以为的所有的成员  
构造方法不能被继承  
一个类仅能继承一个其他类  
父类中声明为public的方法在子类中也必须为public  
父类中声明为protected的方法在子类中要么声明为protected，要么声明为public。不能声明为private  
父类中默认修饰符声明的方法，不能够在子类中声明为private  

#### polymorphism

多态的三个条件:继承、重写、父类变量引用子类对象  
当使用多态方式调用方法时：  
首先检查父类中是否有该方法，如果没有，则编译错误；如果有，则检查子类是否覆盖了该方法  
如果子类覆盖了该方法，就调用子类的方法，否则调用父类方法。举个例子:    
```
class BaseClass {
	public String book = "父类变量";
	public void base(){
		System.out.println("父类普通方法");
	}
	public void test(){
		System.out.println("父类被覆盖的方法");
	}
}
public class demo extends BaseClass{
	public String book = "子类变量";
	public void test(){
		System.out.println("子类覆盖的方法");
	}
	public void sub() {
		System.out.println("子类普通方法");
	}
	public static void main(String[] args) {
		BaseClass a = new demo();
		System.out.println(a.book);//output:父类变量
		a.base();//output:父类普通方法
		a.test();//output:子类覆盖的方法
		//sub()为子类普通方法，此时BaseClass无此方法，发生错误。
		//a.sub();
	}
}
```
编写时类型是BaseClass，运行时类型demo。引用变量只能调用编译时类型，而不能调用运行时类型方法（不考虑重写）

#### static

static修饰符修饰方法、变量，表示是静态的。能够通过类名来访问，不需创建对象实例。  
静态变量属于类，不属于任何独立的对象，也就是只分配一个内存空间。即使有多个实例，这些实例爷会共享该内存，不会重新创建。  
静态变量在类加载的时候就会被初始化，任何一个对象对其修改都会影响其他对象。  
因为静态方法不能操作对象，所以不能在静态方法中访问实例变量，只能访问自身类的静态变量  
<b>总结：</b>  
a.一个类的静态方法只能访问静态变量  
b.一个类的静态方法不能够直接调用非静态方法  
c.如访问控制权限允许，静态变量和静态方法也可以通过对象来访问，但是不被推荐  
d.静态方法中不存在当前对象，因而不能使用this，当然也不能使用super  
e.静态方法不能被非静态方法覆盖  
f.构造方法不允许声明为static的  
g.局部变量不能使用static修饰

#### initialization

父类静态变量->父类静态块初始化->子类静态变量->子类静态块初始化->  
父类变量->父类初始化块->父类构造方法->   
子类变量->子类初始化块->子类构造方法    
(先静后动，先父后子)              静态对象不会再次被初始化

#### final finalize finally

###### final
final可修饰类、变量和方法；  
final修饰的类不能被继承；  
final修饰的方法不能被子类重写,但可以在一个类中重载;    
final修饰的变量（成员变量或局部变量）即成为常量，只能赋值一次；  
final修饰的成员变量必须显式初始化，系统不会为其隐式初始化；  
final修饰类变量必须在静态初始化块或声明时指定初值；  
final修饰实例变量必须在非静态初始化块或构造器中或声明时设定初值；  
final修饰的局部变量可以只声明不赋值，然后再进行一次性的赋值；  
final修饰引用类型变量，只保证引用的地址不变；    
final修饰的方法为静态绑定，不会产生多态。  

###### finalize
方法名，在垃圾收集器将对象从内存中清除出去之前做必要的清理工作，删除对象之前对这个对象调用  

###### finally
finally创建一个代码块，该代码块在一个try，catch块完成之后。finally块无论有没有异常抛出都会执行。
每一个try语句至少需要一个catch语句或finally语句。

#### abstract class and interface

<b>抽象方法：</b>只给出方法定义没有具体实现的方法，即没有方法体。  
<b>抽象类：</b>用abstract修饰的类，可以不含抽象方法，防止被实例化。  
包含一个或者多个抽象方法的类必须定位为抽象类（包括直接定义的抽象方法、继承一个抽象父类却没有完全实现其抽象方法、实现一个接口却没有完全实现其抽象方法）。  
抽象类不能被实例化，抽象方法必须在子类中被实现。  
不能有抽象变量、抽象构造方法或抽象静态方法。  
abstract和final不能同时使用。  
<b>接口：</b>接口中所有方法必须是抽象的。  
接口中声明的成员变量必须是public static final的（这些修饰符可省略），必须显示的初始化。  
接口中只能定义抽象方法，这些方法默认为public abstract的（修饰符可省略），试图在接口中定义实例变量、非抽象的实例方法及静态方法都是非法的。  
接口之间可以继承，并且可以多继承。  
接口中没有构造方法不能被实例化。  
abstract不能与static共同修饰一个类。  
<b>区别：</b>  
1.抽象类可以有具体实现的方法，接口中只能包含抽象方法。  
2.一个类只能继承一个直接的父类，但一个类可以实现多个接口。  
3.抽象类中的成员变量可以是各种类型的，而接口中的成员变量只能是public static final类型的。  
4.接口中不能含有静态代码块以及静态方法，而抽象类可以有静态代码块和静态方法。

#### collection

两个基类接口Map和Collection  
<b>1.Set接口，无序集合，不能有重复元素</b>    
a.EnumSet  
b.StoredSet接口  
c.TreeSet是StoredSet接口的实现类，可以确保元素处于排序状态  
d.HashSet类，按hash算法存储集合中的元素  
e.LinkedHashSet类，HashSet的子类，也是根据元素的HashCode值决定元素的位置，但同时使用链表维护元素的次序。会按元素的添加顺序来访问集合里的元素。  
<b>2.List接口，有序，可以有重复元素，元素对应指引</b>   
a.ArrayList类  
b.Vector类  
c.Stack：Vector的子类  
<b>3.Queue接口，队列，先进先出</b>   
a.Deque接口，双端队列  
b.ArrayDeque是Deque接口的实现类，可作为栈使用，也可以作为队列使用  
c.LinkedList集合是List接口和Deque接口的实现类不仅提供了List功能，还提供了双端队列、栈的功能  

<b>迭代器：</b>用于遍历集合。  
`Iterator it = a.iterator();`遍历a集合  
<b>工具类：</b>Arrays工具类，提供asList(Object... a)方法，可以把一个数组或一些对象转换成List集合     
#### map

Map用于保存具有映射关系的数据，存有两组值，一组保存Key，一组保存Value  
1.HashMap是Map的实现类，Key和Value允许为null  
2.Hashtable不允许  
3.LinkedHashMap是HashMap的子类，使用双向链表维护k-v对的次序，迭代顺序与插入顺序保持一致。  
4.StoredMap接口和TreeMap实现类

与Collection对应也有Collections工具类  

**各个容器的具体方法参考笔记或者API**

#### exception

异常处理结构中只有try块是必需的，catch块和finally块至少出现一个。  
多个catch块时，捕获父类异常的放在后面，否则子类异常捕获不到。  
常见异常：  
IndexOutOfBoundsException  
NullPointerException  
NumberFormatException
ArithmeticException  
ClassNotFoundException

#### annotation

@Override:限定重写父类方法。  
@Deprecatted:标记已过时元素，使用标记的元素会被编译器警告。  
@SupressWarnings:抑制编译器警告。  
<p>元注解：</p>
@Target:注定注解用在什么地方  
@Retention:注解保留多长时间  
@Documented:将注解包含在javadoc中  
@Inherited:允许子类继承父类的注解  

#### concurrency

线程是进程的组成部分，一个进程可以拥有多个线程，一个线程必须有一个父进程。线程可以拥有自己的栈堆、自己的程序计数器和自己的局部变量，但不能拥有系统资源。与其父进程的其他线程共享该近程所拥有的全部资源。  
就像进程在操作系统中的地位一样，线程在程序中独立、并发的执行流。  
<b>创建线程的三种方法：</b>  
1.继承Thread  
2.实现Runnable  
3.使用Callable和Future创建  
其中2、3两种方法类似，只是Callable接口定义的方法有返回值，能声明抛出异常  
<b>1和2、3方式的区别</b>     
a.2、3实现接口的方式，还可以继承其他类  
b.2、3多线程可共享同一个target对象，允许多线程处理同一资源  
c.2、3访问当前线程必须使用Thread.currentThread()  
d.1访问当前线程只需使用this  

#### synchronize

synchronized可修饰方法和代码块，不能修饰成员变量、构造器等。  

#### communication

wait()当前线程等待  
notify()唤醒在等待的单个线程  
notify()唤醒所有线程  

#### deadlock

死锁条件：
互斥使用：有一个资源是不能共享的，只能一个使用使用  
不可强占：资源只能由占有者自愿释放  
一个线程申请新资源的同时保持对原有资源的占有  
循环等待：P1线程等待P2线程占有的资源，P2线程等待P3相爱难成占有的资源，最后的线程又等待P1线程占有的资源  

#### sleep and yield

sleep()方法暂停当前线程后，会给其他线程执行机会，不会理会其他线程的优先级;yield()方法只会给优先级相同或更高的线程执行机会。  
sleep()方法会将线程转入阻塞状态，直到经过阻塞时间才会转入就绪状态；yield会使当前线程进入就绪状态，有可能暂停后继续执行该线程。  
sleep()方法抛出InterruptedException异常，要么显示声明抛出该异常，要么捕捉该异常；yield不抛出异常  

#### synchronized and lock