---
layout: default
title: Java和Javascript的一些比较
comments: true
---

本文就java和javascript的区别进行说明

###一、java和javascript编译的区别

Java程序从源文件创建到程序运行要经过两大步骤：
1、源文件由编译器编译成字节码（ByteCode）

2、字节码由java虚拟机解释运行。因为java程序既要编译同时也要经过JVM的解释运行，所以说Java被称为半解释语言（ "semi-interpreted" language）。［1］

javascript为解释型语言 通过词法分析和语法分析得到语法分析树以后，就可以解释执行了

java生成的字节码，通过不同平台的jvm解析运行实现了跨平台的特性，这点和javascript经过不同浏览器的解析引擎解析实现在不同浏览器的运行十分类似。

java字节码的解析过程主要是生成方法区 堆 栈等内容［2］，而javascript的解析过程首先创建一个全局对象，同时 Math，String，Date等也会被初始化，在预解析时，将全局声明的变量和函数（非函数表达式）作为全局变量的属性，并为函数创建作用域[[scope]]，该作用域包含该函数的所有的上层变量对象。在函数调用的时候复制给执行环境的作用域链属性。该作用域链属性包含当前函数的活动对象和其在创建时生成的作用域。可表示为Scope=VO+[[scope]]。并且创建全局的执行环境。［3］［4］

###二、java是静态类型而javascript是动态类型

静态和动态的概念是表示变量的类型是什么时候被绑定的。

静态类型的变量类型在编译时候就必须绑定，比如 int num;而动态类型表示变量的类型在运行时候绑定，所以我们可以写var 同时绑定可以在运行是随时进行，比如

```
var x // x 为 undefined
var x = 6; // x 为数字
var x = "Bill”;//x 为字符串
```

###三、java是强类型语言javascript是弱类型语言

强弱类型指的是类型检查的严格程度。弱类型检查更不严格，比如说允许变量类型的隐式转换，允许强制类型转换等等

###四、java是通过类来继承 而javascript通过prototype来进行继承

javascript在new 一个function的时候会把该对象的_proto_指向该function的prototype

###五、java和javascript对于分号的处理不同

java需要所有语句块以分号结尾

javascript则不需要，不过我们还是为了可读性以及代码安全性以分号结尾

###六、java和javascript的作用域

java是块级作用域 而javascript则是函数级作用域

同时javascript的作用域是词法作用域 即在函数创建时就定义好的

###七、java和javascript关于闭包的理解

闭包就是为了解决局部变量所在的函数销毁以后，仍然能够访问到该局部变量的问题，ECMA的闭包实现主要通过函数在创建时就创建的作用域来完成，这样当我们在函数中定义一个函数，并在这个内部函数中访问这个外部函数中的对象（就是通过函数的作用域来实现），即使外部函数被销毁了，我们仍然能够访问到这个变量
java的闭包通过匿名类来实现，比如［5］

```
public class DemoClass2 {
	private int length =0;
	public ILog logger() {
	    //在方法体的作用域中定义此局部内部类
	    class InnerClass implements ILog
	    {
	        @Override
	        public void Write(String message) {
	            length = message.length();
	            System.out.println("DemoClass2.InnerClass:" + length);
	        }
	    }
	    return new InnerClass();
	}
}
```

或者

```
public class DemoClass3 {
    private int length =0;
   final int logLevel = level+1;
    public ILog logger() {
    return new ILog() {
        @Override
        public void Write(String message) {
              length = message.length();
              System.out.println("DemoClass3.AnonymousClass:" + length);
              System.out.println(logLevel);   
        }
    };
    }
}
```

或者将new InterfaceClass()作为参数传入
闭包里面所使用外部变量必须使用final修饰符表示其创建以后不能被更改。为什么要使用final标示为不可更改，主要时为了防止闭包共享中变量取值错误的问题。

```
interface Action
{
    void Run();
}

public class ShareClosure {

    List<Action> list = new ArrayList<Action>();
   
    public void Input()
    {
        for(int i=0;i<10;i++)
        {
            final int copy = i;
            list.add(new Action() {   
                @Override
                public void Run() {
                    System.out.println(copy);
                }
            });
        }
    }
   
    public void Output()
    {
        for(Action a : list){a.Run();}
    }
   
    public static void main(String[] args) {
        ShareClosure sc = new ShareClosure();
        sc.Input();
        sc.Output();

    }
}
```

因为 i 变量在各个匿名内部类中使用，这里产生了闭包共享，java编译器会强制要求传入匿名内部类中的变量添加final

###八、javascript的函数是可变的 而java则需要进行一些显示的标记才可变

首先 可变意味者其对参数数量没有要求［6］

在javascript 不需要我们声明，我们可以不使用形参，而在函数内部使用arguments

在java中 需要我们明确声明参数为可变 比如public void function(Object... args){}

［1］http://althing.cs.dartmouth.edu/local/www.acm.uiuc.edu/sigmil/RevEng/ch02.html

［2］http://hxraid.iteye.com/blog/676235

［3］http://dmitrysoshnikov.com/ecmascript/chapter-2-variable-object/

［4］http://dmitrysoshnikov.com/ecmascript/chapter-4-scope-chain/

［5］http://www.cnblogs.com/chenjunbiao/archive/2011/01/26/1944417.html

［6］http://en.wikipedia.org/wiki/Variadic_function