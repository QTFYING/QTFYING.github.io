---
layout: post
title:  "Generator生成器函数"
date:   2019-03-21 17:22:00
category: study
tags: ES6 Generator
comments: true
---
### 1. 声明

Generator的声明方式类似一般的函数声明，只是多了个*号，并且一般可以在函数内看到yield关键字

```javascript
function* showWords() {
    yield 'one';
    yield 'two';
    return 'three';
}

var show = showWords();

show.next() // {done: false, value: "one"}
show.next() // {done: false, value: "two"}
show.next() // {done: true, value: "three"}
show.next() // {done: true, value: undefined}
```

如上代码，定义了一个showWords的生成器函数，调用之后返回了一个迭代器对象（即show）

调用next方法后，函数内执行第一条yield语句，输出当前的状态done（迭代器是否遍历完成）以及相应值（一般为yield关键字后面的运算结果）

每调用一次next，则执行一次yield语句，并在该处暂停，return完成之后，就退出了生成器函数，后续如果还有yield操作就不再执行了

### 2. yield和yield *

有时候，我们会看到yield之后跟了一个*号，它是什么，有什么用呢？

类似于生成器前面的*号，yield后面的星号也跟生成器有关，举个大栗子：

```javascript
function* showWords() {
    yield 'one';
    yield showNumbers();
    return 'three';
}

function* showNumbers() {
    yield 10 + 1;
    yield 12;
}

var show = showWords();
show.next() // {done: false, value: "one"}
show.next() // {done: false, value: showNumbers}
show.next() // {done: true, value: "three"}
show.next() // {done: true, value: undefined}
```

增添了一个生成器函数，我们想在showWords中调用一次，简单的 yield showNumbers()之后发现并没有执行函数里面的yield 10+1

因为yield只能原封不动地返回右边运算后值，但现在的showNumbers()不是一般的函数调用，返回的是迭代器对象

所以换个yield* 让它自动遍历进该对象

```javascript
function* showWords() {
    yield 'one';
    yield* showNumbers();
    return 'three';
}

function* showNumbers() {
    yield 10 + 1;
    yield 12;
}

var show = showWords();
show.next() // {done: false, value: "one"}
show.next() // {done: false, value: 11}
show.next() // {done: false, value: 12}
show.next() // {done: true, value: "three"}
```

要注意的是，这yield和yield* 只能在generator函数内部使用，一般的函数内使用会报错

```javascript
function showWords() {
    yield 'one'; // Uncaught SyntaxError: Unexpected string
}
```

虽然换成yield*不会直接报错，但使用的时候还是会有问题，因为’one'字符串中没有Iterator接口，没有yield提供遍历

```javascript
function showWords() {
    yield* 'one';
}

var show = showWords();

show.next() // Uncaught ReferenceError: yield is not defined
```

在爬虫开发中，我们常常需要请求多个地址，为了保证顺序，引入Promise对象和Generator生成器函数，看这个简单的栗子：

```javascript
var urls = ['url1', 'url2', 'url3'];

function* request(urls) {
    urls.forEach(function(url) {
        yield req(url);
    });

//     for (var i = 0, j = urls.length; i < j; ++i) {
//         yield req(urls[i]);
//     }
}

var r = request(urls);
r.next();

function req(url) {
    var p = new Promise(function(resolve, reject) {
        $.get(url, function(rs) {
            resolve(rs);
        });
    });

    p.then(function() {
        r.next();
    }).catch(function() {

    });
}
```

上述代码中forEach遍历url数组，匿名函数内部不能使用yield关键字，改换成注释中的for循环就行了

### 3. next()调用中的传参

参数值有注入的功能，可改变上一个yield的返回值，如

```javascript
function* showNumbers() {
    var one = yield 1;
    var two = yield 2 * one;
    yield 3 * two;
}

var show = showNumbers();

show.next().value // 1
show.next().value // NaN
show.next(2).value // 6
```

第一次调用next之后返回值one为1，但在第二次调用next的时候one其实是undefined的，因为generator不会自动保存相应变量值，我们需要手动的指定，这时two值为NaN，在第三次调用next的时候执行到yield 3 * two，通过传参将上次yield返回值two设为2，得到结果

另一个栗子：

由于ajax请求涉及到网络，不好处理，这里用了setTimeout模拟ajax的请求返回，按顺序进行，并传递每次返回的数据

```javascript
var urls = ['url1', 'url2', 'url3'];

function* request(urls) {
    var data;

    for (var i = 0, j = urls.length; i < j; ++i) {
        data = yield req(urls[i], data);
    }
}

var r = request(urls);
r.next();

function log(url, data, cb) {
    setTimeout(function() {
        cb(url);
    }, 1000);

}


function req(url, data) {
    var p = new Promise(function(resolve, reject) {
        log(url, data, function(rs) {
            if (!rs) {
                reject();
            } else {
                resolve(rs);
            }
        });
    });

    p.then(function(data) {
        console.log(data);
        r.next(data);
    }).catch(function() {

    });
}
```

达到了按顺序请求三个地址的效果，初始直接r.next()无参数，后续通过r.next(data)将data数据传入

```javascript
url1
url2
url3
```

注意代码的第16行，这里参数用了url变量，是为了和data数据做对比, 因为初始next()没有参数，若是直接将url换成data的话，就会因为promise对象的数据判断 !rs == undefined 而reject;所以将第16行换成 cb(data \|\| url);

```javascript
url1
url1
url1
```



通过模拟的ajax输出，可了解到next的传参值，第一次在log输出的是 url = 'url1'值，后续将data = 'url1'传入req请求，在log中输出 data = 'url1'值



### 4. for...of循环代替.next()

除了使用.next()方法遍历迭代器对象外，通过ES6提供的新循环方式for...of也可遍历，但与next不同的是，它会忽略return返回的值，如

```javascript
function* showNumbers() {
    yield 1;
    yield 2;
    return 3;
}

var show = showNumbers();

for (var n of show) {
    console.log(n) // 1 2
}
```

此外，处理for...of循环，具有调用迭代器接口的方法方式也可遍历生成器函数，如扩展运算符...的使用

```javascript
function* showNumbers() {
    yield 1;
    yield 2;
    return 3;
}

var show = showNumbers();

[...show] // [1, 2, length: 2]
```