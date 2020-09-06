**ES5的prototype与__proto__以及原型链继承**
众所周知，ES5中没有类的概念，对象的创建都是用function模拟出来的，ES5用原型链实现继承。ES6或者TypeScript中用Classs实现继承的方式最后也会被编译或者转义成下面的代码段实现：
```javascript
var __extends  = this.__extends || function(d,b){
	for(var p in b) if(b.hasOwnProperty(p))  d[p] = b[p] ;
	function __() {
		this.constructor = d;
	}
	__prototype = b.prototype;
	d.prototype = new __();
}
```
这里，d表示派生类（derived），或者叫子类，b表示基类。这个函数做了两件事情：
1. 拷贝了基类的静态成员到子类。
```javascript
for(var p in b) if(b.hasOwnProperty(p))  d[p] = b[p] ;
```
1. 设置了
```javascript
d.prototype.__proto__ = b.prototype
```

1很容易理解，2理解要费劲些。下面我们就能解释这些问题。
##### d.prototype.__proto__ = b.prototype 

首先我们来解释，为什么__extends 中的代码与简洁的 `d.prototype.__proto__ = b.prototype` 这代码等价，然后我们再说明这句非常重要。
为理解这些东西，下面这些概念可能需要理解：
1. ` __proto__`
1. prototype
1. 调用的函数中new对this的作用；
1. new对prototype与`__proto__`的作用
