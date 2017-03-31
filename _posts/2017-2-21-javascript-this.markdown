---
layout:     post
title:      "JavaScript中的this"
subtitle:   " \"What's 'this' in JavaScript\""
author:     "Nian Tianlei"
header-img: "img/post-bg-2016.jpg"
tags:
    - JavaScript
    - 前端
---

> 下滑这里查看更多内容

## 前言

我在学习JavaScript的过程中，总是遇到*this*这个关键字。    
在Java中，*this*有三个用法，即访问成员变量，对当前对象的引用，调用其他构造方法。    
而js中的*this*会动态绑定一个对象，这里，详细分析了在JavaScript中*this*在不同情况中的意义。


---

由于动态绑定的特性，*this*含义复杂得多，它可以是全局对象、当前对象或者任意对象，这完全取决于函数的调用方式。JavaScript 中函数的调用有以下几种方式：作为对象方法调用，作为函数调用，作为构造函数调用，和使用 apply 或 call 调用。下面我们将按照调用方式的不同，分别讨论*this*的含义。
## 绑定当前对象
在 JavaScript 中，函数也是对象，因此函数可以作为一个对象的属性，此时该函数被称为该对象的方法，在使用这种调用方式时，this 被自然绑定到该对象。    
例1:    

    var count = {
        a: 0,
	    add10: function(a) {
	        this.a = a + 10;
	    }
	};

	count.add10(2);
	alert(count.a);//输出12, this 绑定当前对象，即count

## 作为函数调用
this绑定全局对象，即window    
例2:    

    var x = 0;

	function move(x) { 
	    this.x = x; 
	} 

	move(2);
	alert(x);//输出2    

对于内部函数，这种绑定到全局对象的方式会产生另外一个问题。以 count 对象为例，这次在move函数里面定义一个函数，结果count.x没有变，反而多了一个全局变量x。    
例3:  

	var count = { 
	    x : 0, 
	    move : function(x) { 
	     // 内部函数
	        var moveX = function(x) { 
	        this.x = x;//this 绑定到了哪里？
	    }; 
	    moveX(x); 
        } 
    }; 
    count.move(1); 
    document.write(count.x + "\n"); //0 
    document.write(x + "\n"); //1 

正常来讲，内部函数的this应该绑定到其外层函数对应的对象。为避免这一缺陷，that就出现了。    
例4:   

	var count = { 
	    x : 0, 
	    move : function(x) { 
		var that = this;
    	// 内部函数
    	var moveX = function(x) { 
    	    that.x = x;
   	};
	    moveX(x); 
	    } 
	}; 
	count.move(1); 
	document.write(count.x + "\n"); //1

## 作为构造函数调用
this绑定到新创建的对象上    
例5：   

	function Count() {
	    this.x = 1;
	}

	var s = new Count();
	alert(s.x);//1

## apply和call调用
例6:    

	function sum(num1, num2){
		return num1 + num2;
	}
	function callSum1(num1, num2){
		return sum.apply(this, arguments); // 传入arguments 对象
	}
	function callSum2(num1, num2){
		return sum.apply(this, [num1, num2]); // 传入数组
	}
		
	alert(callSum1(10,10)); //20
	alert(callSum2(10,10)); //20

这里的this代表全局作用域。  
### 补充一个例子
	var fullname = 'nian';
	var obj = {
		fullname: 'tian', prop: {
			fullname: 'lei', 
			getFullname: function() {
				return this.fullname;
			}
		}
	}

	console.log(obj.prop.getFullname());
	var test = obj.prop.getFullname;
	console.log(test());

输出依次为:lei nian  
原因：this引用函数上下文，取决于函数是如何调用的。第一个console.log中getFullname()是作
为obj.prop对象的函数被调用。当getFullname()被赋值给test变量时，当前的上下文是window。