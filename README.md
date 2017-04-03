利用 koa 搭建一個简易的论坛系统，参考 [koa-web](https://nswbmw.github.io/N-club/index.html)


----


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


## yield 与 yield*

两者的区别在于：```yield``` 只是返回右值，而 ```yield*``` 则将函数委托（delegate）到另一个生成器（ Generator）或可迭代的对象（如字符串、数组和类数组 ```arguments```，以及 ```ES6``` 中的 ```Map```、```Set``` 等）

#### Array 与 String

```js
function* bar () {
    yield [1, 2];
    yield* [3, 4];
    yield "20";
    yield* "20";
}

var foo = bar();

foo.next();  // {value: [1, 2], done: false}

foo.next();  // {value: 3, done: false}

foo.next();  // {value: 4, done: false}

foo.next();  // {value: "20", done: false}

foo.next();  // {value: "2", done: false}

foo.next();  // {value: "0", done: false}
```


#### arguments

```js
function* bar () {
    yield arguments;
    yield* arguments;
}

var foo = bar(1, 2);

console.log(gen.next().value); // { "0": 1, "1": 2 }

console.log(gen.next().value); // 1
console.log(gen.next().value); // 2
```

#### Generator

```js
function* bar1 () {
    yield 2;
    yield 3;
}

function* bar2() {
    yield 1;
    yield* bar1();
    yield 4;
}

var foo = bar2();

console.log(foo.next().value); // 1
console.log(foo.next().value); // 2
console.log(foo.next().value); // 3
console.log(foo.next().value); // 4
```


#### Object

```js
function* bar () {
    yield { a: "1", b: "2" };
    yield* { a: "1", b: "2" };
}

var foo = bar();

foo.next();  // {value: { a: "1", b: "2" }, done: false}

foo.next();  // undefined is not a function
```




## co 和 koa

#### co

```co``` 库的主要是用来通过生成器函数以及 ```yield``` 解决回调嵌套的

例如我们要按顺序读取 a，b，c 三个文件，通常是下面这样的：

```js
var fs = require("fs");

function readFile(path, callback) {
    fs.readFile(path, { encoding: "utf-8" }, callback)
}

readFile("a.txt", function (err, dataA) {
    console.log(dataA);
    readFile("b.txt", function (err, dataB) {
        console.log(dataB);
        readFile("c.txt", function (err, dataC) {
            console.log(dataC);
            // ...
        })
    })
})
```

在使用了 ```co``` 模块之后：

```js
var fs = require("fs");
var co = require("co");

function readFile (path) {
    fs.readFile(path, {encoding: "utf-8"}, callback)
}

co(function* () {
    var dataA = yield readFile("a.txt");
    console.log(dataA);

    var dataB = yield readFile("b.txt");
    console.log(dataB);

    var dataC = yield readFile("c.txt");
    console.log(dataC);
    
}).catch(function(err){
    console.log(err);
})
```

我们把 ```readFile``` 进行改造，使它返回了 ```thunk``` 函数（即有且只有一个参数是 ```callback``` 的函数，且 ```callback``` 的第一个参数为 ```error```）

co@4 核心代码如下：

```js
function co(gen) {

    var ctx = this;

    return new Promise(function (resolve, reject) {
        if (typeof gen === 'function') gen = gen.call(ctx);
        if (!gen || typeof gen.next !== 'function') return resolve(gen);

        onFulfilled();

        function onFulfilled(res) {
            var ret;
            try {
                ret = gen.next(res);
            } catch (e) {
                return reject(e);
            }
            next(ret);
        }

        function onRejected(err) {
            var ret;
            try {
                ret = gen.throw(err);
            } catch (e) {
                return reject(e);
            }
            next(ret);
        }

        function next(ret) {
            if (ret.done) return resolve(ret.value);
            var value = toPromise.call(ctx, ret.value);
            if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
            return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '
                + 'but the following object was passed: "' + String(ret.value) + '"'));
        }
    });
    
}
```

可以看出，```co``` 将所有 ```yield``` 后面的表达式都封装成了 ```Peomise``` 对象（本身也返回一个 ```Promise``` 对象），只有当前表达式执行结束后（即调用 ```.then```），然后会在 ```onFulfilled``` 函数内执行 ```gen.next(res)``` 将 ```res``` 赋值给 ```yield``` 左侧的变量并执行到下一个 ```yield```，下一个表达式执行结束后油调用 ```gen.next()```，如此循环，直至 ```done``` 为 ```true```




