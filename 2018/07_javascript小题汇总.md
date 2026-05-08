# javascript小题汇总
Aug 27, 2018
## js变量及函数提升 ——预编译

> 考点
>
> 首先，预编译是发生在函数执行的前一刻
>
> 第一步：创建AO（Activation Object)对象也就是执行期上下文
>
> 第二步：找形参和变量声明，将变量和形参作为AO属性名，此时值是undefined
>
> 第三步：将实参和形参统一，也就是如果有实参传过来的话，改变对应AO里的属性值
>
> 第四步：在函数体里面找函数声明，值赋予函数体。

**b函数提升， b后被赋值，被覆盖。**

```javascript
function test(a, b) {
	console.log(a);
	c = 0;
	var c;
	a = 3;
	b = 2;
	console.log(b);
	function b() {}
	function d() {}
	console.log(b);
}
test(1)

// => 1 2 2

```

**VO/AO对象，进入到函数的执行上下文后，变量声明不会干扰到VO中已经存在的同名函数声明或形式参数声明？**

eg:1st_place_medal:

```javascript
function test(a,b) {
	console.log(a);
	console.log(b);
	var b = 234;
	console.log(b);
	a = 123;
	console.log(a);
	function a() {} //函数定义[FD]
	var a;   // 不会影响VO中的 a的形式参数声明下的值， 即a还是那个a；
	b = 234;
	var b = function() {}  //函数表达式[FunctionExpression, 缩写为 FE]
	console.log(a);
	console.log(b);
}
test(1)

// => function a() {}  undefined  234  123  123  function() {}
```

eg:1st_place_medal:

```javascript
//if里a函数无提升(预解析)
function foo() {
	console.log(a);
	if(false) {
		function a() {}
	}
	var a;
	console.log(a)
}
// undefined
// undefined

// a函数和a变量都参与 预解析，且函数优先级高于变量
function foo() {
	console.log(a);
	function a() {}
	var a;
	console.log(a)
}
// function a() {}
// function a() {}
```

eg:1st_place_medal::1st_place_medal:

```javascript
console.log(test);
function test(test) {
	console.log(test);
	var test = 345;
	console.log(test);
	function test() {};
}
test(1);
var test = 123;

// => function test(test) {…}    function test() {}   345
```

eg:1st_place_medal::1st_place_medal::1st_place_medal:

```javascript
function test() {
	console.log(b);
	if(a){
		var b = 100;
	}
	c = 234;
	console.log(c);
}
var a;
test();
a = 10;
console.log(c);
// => undefined   234  234
```

eg:1st_place_medal::1st_place_medal::1st_place_medal::1st_place_medal:

```javascript
a = 100;
function fn() {
	console.log(a);
	a = 200;
	console.log(a);
	var a = 300;
}
fn();
var a;
// => undefined 200
```

eg:1st_place_medal:🥇🥇🥇🥇

```javascript
function bar() {
	return foo;
	foo = 10;
	function foo() {

	}
	var foo = 11;
}
console.log(bar());

// => function foo() {}
```

## 操作符运算

> 考点：
>
> 1. 变量未定义（未声明），直接访问会报错，但typeof a  不会报错，而是返回 undefined。
> 2. 一元操作符 `-` 或 `+` 会隐式转换操作数为数字，无法转出数字的返回NaN。-true => -1 ,  +undefined =>NaN。
> 3. 只有当加法运算时，其中一方是String，就会隐式转换另一方为String；其他运算只要有一方是Number，那另一方就转成Number。
> 4.  ! > 算术运算符 > 关系运算符 > 逻辑运算符 > 条件（三元）运算符> 赋值运算符 > 逗号运算符。

```Javascript
var str = false + 1;
 console.log( str );
var demo = false == 1;
 console.log( demo);
 if(typeof(a)&&-true + (+undefined) + "") {
 console.log('扎实');
}
 if(11 + '11' * 2 == 33) {
 console.log('扎实');
    }
!!" " + !!"" - !!false||console.log('🍎');

```

## this指向问题

1. **arguments  &&  this指向**

```javascript

// 示例1：
var length = 10;
function fn() { 
    console.log(this.length);
}
var obj = {
    length: 5,
    callAPI: function(fn) {
        fn();
        arguments[0]();
    }
}

obj.callAPI(fn, 100, 200); //分别输出 10 和 3;
```

```Javascript
// 示例2：(简化版)
function helloThis() {
    console.log(this.length);
}
var arr = [helloThis, 1, 'smile', 'sgg', 100]
arr[0]() // ->5
```

2. **自执行函数、对象的方法调用  && this指向 **

```Javascript
// 下面的代码将输出到控制台的是什么？，为什么？
var myObject = {
    foo: "bar",
    func: function() {
        var self = this;
        console.log("outer func:  this.foo = " + this.foo);
        console.log("outer func:  self.foo = " + self.foo);
        (function() {
            console.log("inner func:  this.foo = " + this.foo);
            console.log("inner func:  self.foo = " + self.foo);
        }());
    }
};
myObject.func();    // -> bar ,  bar ,  undefined,  bar
```

3. **严格模式 && this指向**

```Javascript
function foo() {
	console.log(this.a);  // 全局为非严格模式，this指向window；-> this.a 为全局的 2；
}
var a = 2;
function foo2(){
	"use strict";
	foo(); // foo2为严格模式，但调用的全局foo函数在全局非严格模式下定义，this依然指向window;
    //console.log(this.a);  -> 严格模式下 this === undefined
 }
foo2();  // -> foo() -> console.log(2);
```

4. **箭头函数 && this指向**

```javascript
var obj = {
    birth: 1990,
    getAge: function (year) {
        var b = this.birth; 
        var fn = (y) => y - this.birth;  // 箭头函数this已经按照词法作用域绑定为obj。
 		return fn.call({birth:2000}, year);  // call、apply不能改变箭头函数的this指向。
    }
};
obj.getAge(2015);  -> return 25  注释： [2015 - 1990]
```



## 你不知道的js易错小题

**1. parseFloat知道多少**

```javascript

parseFloat('10hello'); // -> 10

parseFloat('hello 10'); // -> NaN

parseFloat(' '); // -> NaN

parseFloat(' 10'); // -> 10

parseFloat(' 10 20 30'); // -> 10

/**
* 【说明】
* 如果参数值只要空格时返回NaN，但以空格开头+数字，可以返回数字。
* 字符串中只返回第一个数字。
* 注意： 开头和结尾的空格是允许的。
* 注意： 如果字符串的第一个字符不能被转换为数字，那么 parseFloat() 会返回 NaN。
*/
```

**2. es5继承——apply、call、bind**

```Javascript

function myBind(func, target) {
    return function () {
          return func.apply(target, arguments);
    };
}

function a(arg) {
	console.log('a:' + arg);
}

function b(arg) {
    console.log('b:' + arg);
}

// myBind(a, b)('sgg'); //-> 'a:sgg'

b('sgg') //-> 'b:sgg'

```

**3. a如何晋升为全局变量?**

```Javascript
function fn() {
    a = 'hello';
}
// fn();   // 如果先不调用fn， 直接输入a，报错[1]；先调用fn，后输出a。正确打印 a -> 'hello'
console.log(a);
// [1] VM6392:5 Uncaught ReferenceError: a is not defined
//    at <anonymous>:5:13
```

**4.var还是没var？**

```Javascript
// 未var b；
(function() {
    var a = b = 3;    // == var a = 3; b = 3; [b未用var声明，不参与变量提升->预解析]；
})();

//输出验证
console.log(a);  // -> VM6419:1 Uncaught ReferenceError: a is not defined
    			// at <anonymous>:1:1
console.log(typeof a === 'undefined')  // -> true;
console.log(typeof b === 3);  // -> true;

#################################################

// 已var b；
(function() {
    var a, b = 3; 	  // == var a; var b = 3;  分别用var声明了a和b，都会参与预解析；
})

//输出验证
console.log(typeof a === 'undefined');  // -> true;
console.log(typeof b === 'undefined');  // -> true;

```

**5. return了什么？**

```Javascript
function fn1() {
	return {
        say: 'hello'
	}
}

function fn2() {
    return 
    {
        say: 'hello'
    }
}

console.log(fn1()); // -> {say: 'hello'};
console.log(fn2()); // -> undefined;

/**
* 解释：
* 1. javascript中分号在技术上是可选。尽管忽略它通常是非常糟糕的形式；
* 2. 遇到return中的语句的行，没有其他内容时，会立即在return 语句后边自动插入分号。
*/
```

