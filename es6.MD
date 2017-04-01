## function 和 function*

先看下面这个实例

```js
function* hello () {
    yield "hello";
    yield "world";
    return "!";
}

var hello = hello();

hello.next();  // {value: "hello", done: false}

hello.next();  // {value: "world", done: false}

hello.next();  // {value: "!", done: true}

hello.next();  // {value: undefined, done: true}
```

* 生成器函数执行后会返回一个生成器（Generator）

* 函数执行到每个 ```yield``` 的时候都会暂停执行并返回 ```yield``` 的值（函数上下文，如变量绑定等信息会保留），通过调用生成器的 next 方法返回一个包含 ```value```（当前返回的值）和 ```done```（表明函数是否执行结束） 的值，每次调用 ```next```，函数会从 ```yield``` 的下一个语句继续执行。等到整个函数执行完，next 方法返回 ```value``` 变为 ```undefined```，```done``` 变成 ```true```

* 生成器函数也是函数，所以拥有所有函数的特性。比如作用域，闭包，遇到第一个 ```return``` 会执行结束等等

* 生成器函数又有些普通函数没有的特性，如可以使用 ```yield``` 并且 ```yield``` 只能在生成器函数内使用，如果在普通函数内使用 ```yield``` 将会报错
