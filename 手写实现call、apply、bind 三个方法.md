##### 一、上下文（this）

首先我们要搞清一个上下文的问题：

```kotlin
var obj = {
    "name" : "Condor"
};
var name = "Hero";
function fun(){
    console.log(this.name)
};
fun();
```

函数执行，this 指向 window，this.name 就是 window.name，控制台打印出 Hero。

现在变一下：

```kotlin
var name = "Hero";
var obj = {
    "name" : "Condor",
    fun(){
        console.log(this.name)
    }
};
obj.fun();
```

obj 对象属性有个函数，通过 `obj.fun();` 来执行 this 指向 obj，此时的 this.name 就是 obj.name 打印结果为：Condor，记住这个用法下面立马用到。

##### 二、自定义 call

先来看看 call 是怎么用的：

```kotlin
var name = "Hero";
var obj = {
    "name" : "Condor"
};
function fun(){
    console.log(this.name);//Condor
    console.log(...arguments);//1 2 3
}
fun.call(obj,1,2,3);
```

要实现 call，分为两步：

1. 改变 this 指向
2. 处理传入的参数

先看看怎么改变 this 的指向问题。call 这个方法是属于函数的，所以我们要实现的类似功能方法的话，需要给在构造函数上的原型上绑定自定义方法，这个方法现在我们就叫它`_call`：

```kotlin
var name = "Hero";
var obj = {
    "name" : "Condor"
};
//如果你看不懂Function构造函数里面的this请去复习ES5的原型链
Function.prototype._call = function(){
    // ctx需要绑定的上下文，如果没有就去绑定window
    ctx = arguments[0] || window;
    //此处的this就是fun这个函数
    //现在把这个函数，作为ctx(obj)这个对象的属性
    //执行ctx.fn()，fun函数里面的this就变成了obj
    ctx.fn = this;
    return ctx.fn();
}
function fun(){
    console.log(this.name);
    console.log(...arguments);
}
fun._call(obj);
```

程序运行结果是：`Condor`。成功改变 this 的指向。再来看看怎么传参的问题：

```jsx
var name = "Hero";
var obj = {
    "name" : "Condor"
};
//如果你看不懂Function构造函数里面的this请去复习ES5的原型链
Function.prototype._call = function(){
    // ctx需要绑定的上下文，如果没有就去绑定window
    ctx = arguments[0] || window;
    //此处的this就是fun这个函数
    //现在把这个函数，作为ctx(obj)这个对象的属性
    //执行ctx.fn()，fun函数里面的this就变成了obj
    ctx.fn = this;
    let argv = [];
    //第一个参数是绑定上下文的所以从第二个开始拿参数
    for(let i = 1;i < arguments.length;i++){
        argv.push(arguments[i]);
    }
    //把参数argv传给fun
    return ctx.fn(...argv);
}
function fun(){
    console.log(this.name);
    console.log(arguments);
}
fun._call(obj,1,2,3);
```

顺利解决传参的问题，call 的功能模仿完成。

小小的优化一下：改变 this 指向的时候，我们给 ctx ，指定属性是 fn，万一 obj 里面有这个东西不是把原来的给覆盖了吗。所以使用 ES6 的 Symbol，重名也不怕，使用完使用 delete 删掉。

```jsx
let fn = Symbol();//新增
ctx[fn] = this;//改动
let result = ctx[fn](...argv);//改动
delete ctx[fn];//改动
return result;//改动
```

在仔细想想 ctx(obj) 是一个对象，它构造函数 Object 的实例，而方法一般都是绑定在 Object 的原型上的。所以方法绑定在原型上：

```kotlin
ctx.__proto__[fn] = this;
```

##### 三、自定义 apply

call 和 apply 的在使用的时候，有个区别就是 apply 的参数必须是通过数组的形式，参数进去。类似这样 `fun.apply(obj,[1,2,3]);` 这个实现起来就比较简单了， 改变 this 指向和 call 同理，参数直接是个数组都省的我们去处理了。

```jsx
var name = "Hero";
var obj = {
    "name" : "Condor",
    "fn" : 12
};
Function.prototype._apply = function(){
    ctx = arguments[0] || window;
    let fn = Symbol();
    ctx.__proto__[fn] = this;
    let argv = [];
    if(arguments[1]){
        argv = arguments[1];
    }else{
        argv = [];
    }
    let result = ctx[fn](...argv);
    delete ctx[fn];
    return result;
}
function fun(){
    console.log(this.name);
    console.log(...arguments);
}
fun._apply(obj,[1,2,3]);
```

##### 四、自定义 bind

bind 和 call 差不多，有点不同的就是 bind 终生绑定不立即执行返回一个新函数，传参和 call 是一样的。

```jsx
var name = "Hero";
var obj = {
    "name" : "Condor",
    "fn" : 12
};
Function.prototype._bind = function(){
    ctx = arguments[0] || window;
    let fn = Symbol();
    ctx.__proto__[fn] = this;
    let argv = [];
    //第一个参数是绑定上下文的所以从第二个开始拿参数
    for(let i = 1;i < arguments.length;i++){
        argv.push(arguments[i]);
    }
    let result = function(){
        return ctx[fn](...argv);
    };
    return result;
}
function fun(){
    console.log(this.name);
    console.log(...arguments);
}
let newfun = fun._bind(obj,1,2,3);
newfun();
newfun();
newfun();
```

思路和 call 是一样唯一的难点是函数不能立即执行，那我们就顺着思路返回一个匿名函数，当它执行的时候在去改变 this 指向和处理参数。