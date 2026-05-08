# Array.prototype.reduce()方法应用实践
Aug 27, 2018
⭐️⭐️

> 统计一个字符串中每个字母出现的次数？

```javascript
function showCount(str) {
	return str.split('').reduce((sum, item) => {
		sum[item] ? ++sum[item] : sum[item] = 1
	return sum;
}, {});
}
let str = 'sdfasdfeewe';
showCount(str)
//{"s": 2,"d": 2,"f": 2,"a": 1,"e": 3,"w": 1}
```

⭐️⭐️

> 给定一对象和一数组，从该对象中依据该数组给定的key值筛选出一个新对象？

```Javascript
function extractObj(obj) {
	if(Object.prototype.toString.call(obj) !== '[object Object]') {
		throw new Error("The param is JSON's Object type");
	}
	return function(extractKeys) {
		if(Object.prototype.toString.call(extractKeys) !== '[object Array]') {
			throw new Error('The param is Array type');
		} 
		return extractKeys.reduce((extractObj, key) => {
			if(!obj[key]) return extractObj;
			extractObj[key] = obj[key];
			return extractObj;
		}, {});
	}
}

let a = {name: 'sgg', age: 24, phone: 1234567789};
let extractA = extractObj(a);

extractA(['name', 'hello', 'age'])
```

⭐️⭐️⭐️

> 使用reduce方法，实现对一个对象中的多个属性分别执行多次叠加（修改）操作？ —— 处理Obj

```Javascript
// state加工的工厂(工人+工料 = 产品)
function combineReducer(reducers) {
	return function(state, actions) {
		actions.forEach((action) => {
			Object.keys(reducers).reduce((state, keys) => {
				reducers[keys](state, action);
				return state;
			}, state)
		});
		return state;
	}
}

// 创建一个产品， state
let state = {
	iphone: 10000,
	iwatch: 500
}

// state加工的工人
let reducers = {
	dealIPhonePrice(state, action) {
		state.iphone += action.price;
		return state;
	},

	dealIWatchPrice(state, action) {
		state.iwatch *= action.range;
		return state;
	}
};

// state加工的工料
let actions = [
    {price: 100, range: 1.1},
	{price: 180, range: 1.08},
	{price: 280, range: 1.2}
]


//雇佣工人进厂，准备干活
let rootReducer = combineReducer(reducers);
//工人(reducers)把工料(actions) 加工成产品(state)
rootReducer(state, actions)
```

⭐️⭐️⭐️

> 利用reduce实现一个组合函数？ —— 处理Function

```Javascript
//组合多个函数，reduce版
function composeFn(...funcs) {
	return funcs.reduce((a,b) => args => a(b(args)));
}

//for循环版(功能，同上)
function composeFn(...funcs) {
	return function(args) {
		let len = funcs.length - 1;
		for(let i=len; i>=0; i-=2) {
			args = i > 0 ? funcs[i-1](funcs[i](args))
					     : funcs[i](args);
		}
		return args;
	}
}

// ****** demo ******
function fn1(arg) {
	return arg * 2;
}

function fn2(arg) {
	return arg + 10;
}

let composeAfterFn = composeFn(fn1, fn2);
composeAfterFn(1);

```

⭐️⭐️⭐️⭐️

> 描述如下代码的执行过程及结果

```javascript
let arr = [];
for(let i=0; i<10; i++) {
	arr.push((foo) => {
		console.log(i);
		return function() {
			foo();
		}
	});
}

let boo = arr.reduce((pre, cur) => {
	return cur(pre);
});
// => 1 2 3 4 5 6 7 8 9
boo()
// => 0
```



## 参考文档
1. [js数组reduce方法详解及进阶](https://segmentfault.com/a/1190000010731933)

