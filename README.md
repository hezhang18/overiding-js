# JS 源码实现

阅读源码，学习前人的经验以提升自己的能力。源码往往是前人留下的最佳实践，我们跟着前人的脚步去学习更会让我们事半功倍。

### 1. new

原理：

1. 获取构造函数与参数；
2. 继承foo.prototype的属性；
3. 继承构造函数中的属性;
4. 返回新对象。

实现：

```
let myNew = function(){
    let foo = arguments[0],
        params = Array.prototype.slice.call(arguments, 1);
    let obj = Object.create(foo.prototype);
    let nObj = foo.apply(obj, params);
    return typeof nObj === 'object' ? nObj : obj;
}
```

测试：

```
function P(name, words){
    this.name = name;
    this.say = function(){
        console.log(words);
    }
}

let p = myNew(P, 'Jeff', 'Hi!');
```

### 2. call、apply、bind

原理：
call、apply、bind本质都是改变this的指向。不同点在于call以逗号分隔方式传参、apply以数组形式传参，此外call、apply都是绑定函数并执行返回结果，而bind是绑定函数但不执行。bind方法即可在bind中传参也可在执行时传参，实例化时this指向会失效（即，new一个对象时指向实例化对象，否则指向被绑定的context）。

bind实现：

1. 预热方法

```
Function.prototype.myBind = function(context){
    let _self = this,
        args = Array.prototype.slice.call(arguments, 1);

    let fn = function(){
        let newArgs = Array.prototype.slice.call(arguments),
            newThis = this instanceof _self ? this : context;
            //1，修正this指向错误问题。

        _self.apply(newThis, args.concat(newArgs);)
    }
    fn.prototype = _self.prototype; 
    //2，通过原型链使实例化对象可以继承绑定函数中的属性。

    return fn;
}
```
> var person = Person.myBind(p, 'Jeff'); var obj = new person('male'); 此时在fn = function(){...}中console.log(this, _self)可以发现，this指向实例化对象obj（有2语句时，obj中会有name与gender属性；没有该语句时，obj会被实例化成一个空对象），_self指向绑定函数Person(){...}。
  
> var person = Person.myBind(p, 'Jeff')(); 此时在fn = function(){...}中console.log(this, _self)可以发现，this指向window(出现了this指向错误的问题)，_self指向绑定函数Person(){...}，因此在程序中需要通过newThis与apply对this的指向进行修正。

2. 圣杯模式

```
Function.prototype.myBind = function(context){
    let _self = this,
        args = Array.prototype.slice.call(arguments, 1),
        Buffer = function(){};
    
    let fn = function(){
        let newArgs = Array.prototype.slice.call(arguments),
            newThis = this instanceof _self ? this : context;
        _self.apply(newThis, args.concat(newArgs));
    }

    Buffer.prototype = _self.prototype;
    fn.prototype = new Buffer();

    return fn;
}
```

call实现：

```
Function.prototype.myCall = function(context){
    if(typeof this !== 'function){
        throw new TypeError(`${this} is not a function`); // 调用call的若不是函数则报错
    }

    let args = [...arguments].slice(1);

    context = context || window;
    context[fn] = this; // 将调用call函数的对象添加到context的属性中

    let res = context[fn](...args); // 执行该属性

    delete context[fn]; // 删除该属性

    return res;
}
```

apply实现：

```
Function.prototype.myApply = function(context){
    if(typeof this !== 'function){
        throw new TypeError(`${this} is not a function`);
    }

    let args = arguments[1];

    context = context || window;
    context[fn] = this;

    let res = context[fn](...args);

    delete context[fn];

    return res;
}
```

### 3. 对象克隆（浅拷贝、深拷贝）

原理：

浅拷贝无法拷贝对象、数组这样的引用值属性，目标对象修改引用值属性会导致原对象的也被修改。深拷贝可以拷贝对象、数组这样的引用值属性。目标对象修改引用值属性不影响原对象。

实现：

1. 浅拷贝

```
function clone（target, origin）{
    let target = target || {};
    for(let key in origin){
        // 原型上的属性会被拷贝到target中，因此需要剔除。
        if(origin.hasOwnProperty(key)){
            target[key] = origin[key];
        }
    }
    return target;
}
```

2. 深拷贝

```
function deepClone(target, origin){
    let target = target || {},
        toStr = Object.prototype.toString(),
        arrType = '[object Array]';
    for(let key in origin){
        if(typeof origin[key] === 'object' && origin[key] !== null){
            toStr.call(origin[key]) === arrType ? target[key] = [] : target[key] = {};
            deepClone(target[key], origin[key]);
        }else{
            target[key] = origin[key];
        }
    }
}
```

测试：

```
// 浅拷贝

var person1 = clone(person); // 或
var person1 = {}; clone(person, person1);

person1.name = 'Tom'; 
// 此时不会影响person的该属性。

person1.children.third={name: 'Ben', age: 8}; 
// 此时person的children属性也被修改。

// 深拷贝

var person2 = deepClone(person); // 或
var person2 = {}; deepClone(person, person2);

person2.name = 'Tencent';
//此时不会影响person的该属性。

person2.children.fourth = {name: 'Mary', age: 1};
//此时person的children属性不会被修改。
```

### 4. Object.create

原理：

```
Object.create =  function (o) {
    var F = function () {};
    F.prototype = o;
    return new F();
};
```

重写：

```
if (typeof Object.create !== "function") {
    Object.create = function (prototype, properties) {
        if (typeof prototype !== "object") { throw TypeError(); }
        
        function Ctor() {}
        Ctor.prototype = prototype;
        let o = new Ctor();
        
        if (prototype) { o.constructor = Ctor; }
    
        if (properties !== undefined) {
            if (properties !== Object(properties)) { throw TypeError(); }
            Object.defineProperties(o, properties);
        }
        
        return o;
    };
}
```

### 5. 原型继承

原理：

1. 创建一个中间类，使子类原型指向该中间类的实例化空对象，目的是子类在修改属性时不会影响父类的原型；
2. 修改构造函数指向；
3. 添加超类，指明或标记继承于谁（即Target生父）。

实现：

1. 圣杯模式

```
function inherit(Target, Origin){
    function Buffer(){};
    Buffer.prototype = Origin.prototype;
    Target.prototype = new Buffer();
    Target.prototype.constructor = Target;
    Target.prototype.uber = Origin;
}
```

2. 圣杯模式（雅虎版）

利用闭包将Buffer变成私有化构造函数。其在函数执行完后会被销毁，但是他传递原型的功能已经实现。

```
let inherit = (function(){
    let Buffer = function(){};
    return function(Target, Origin){
        Buffer.prototype = Origin.prototype;
        Target.prototype = new Buffer();
        Target.prototype.constructor = Target;
        Target.prototype.uber = Origin;
    }
})();
```

3. 我的实现思路

Object.create()创建了一个空对象，空对象的原型指向Origin.prototype, Target的原型指向该空对象。与圣杯模式中创建buffer有异曲同工之妙。

```
let inherit = function(Target, Origin){
    Target.prototype = Object.create(Origin.prototype);
    Target.prototype.constructor = Target;
    Target.prototype.uber = Origin;
}
```

4. ES6中继承的实现

```
class Father {
    constructor(name,age){
        this.name = name;
        this.age = age;
    }
    show(){
        console.log(`我的名字是${this.name}，今年${this.age}岁。`);
    }
}

class Son extends Father{};

let s = new Son('Jeff',24);
```

### 6. instanceof 

原理：

L 的 __proto__ 是不是等于 R.prototype，不等于再找 L.__proto__.__proto__ 直到 __proto__ 为 null （L 表示左表达式，R 表示右表达式）

实现：

```
function myInstanceof(L, R) {
    let O = R.prototype;
    L = L.__proto__;
    while (true) {
        if (L === null) return false;
        if (O === L) return true; // 重点：当 O 严格等于 L 时，返回 true
        L = L.__proto__;
  }
}
```

### 7. Array.isArray

```
Array.myIsArray = function(o) {
    return Object.prototype.toString.call(Object(o)) === '[object Array]';
};
```

### 8. Promise

原理：

1. 构造函数接收一个 executor 函数，并会在 new Promise() 时立即执行该函数
2. then 时收集依赖，将回调函数收集到 成功/失败队列
3. executor 函数中调用 resolve/reject 函数
4. resolve/reject 函数被调用时会通知触发队列中的回调

```
const isFunction = variable => typeof variable === 'function';

// 定义Promise的三种状态常量
const PENDING = 'pending';
const FULFILLED = 'fulfilled';
const REJECTED = 'rejected';

class MyPromise {
    // 构造函数，new时触发
    constructor(handle: Function) {
        try {
            handle(this._resolve, this._reject);
        } catch (err) {
            this._reject(err);
        }
    }
  
    // 状态 pending fulfilled rejected
    private _status: string = PENDING;
    
    // 储存 value，用于 then 返回
    private _value: string | undefined = undefined;
  
    // 失败队列，在 then 时注入，resolve 时触发
    private _rejectedQueues: any = [];
  
    // 成功队列，在 then 时注入，resolve 时触发
    private _fulfilledQueues: any = [];
  
    // resovle 时执行的函数
    private _resolve = val => {
        const run = () => {
            if (this._status !== PENDING) return;
            this._status = FULFILLED;
      
            // 依次执行成功队列中的函数，并清空队列
            const runFulfilled = value => {
                let cb;
                while ((cb =   this._fulfilledQueues.shift())) {
                    cb(value);
                }
            };
      
            // 依次执行失败队列中的函数，并清空队列
            const runRejected = error => {
                let cb;
                while ((cb = this._rejectedQueues.shift())) {
                    cb(error);
                }
            };
        
            /*
            * 如果resolve的参数为Promise对象，
            * 则必须等待该Promise对象状态改变后当前Promsie的状态才会改变
            * 且状态取决于参数Promsie对象的状态
            */
            if (val instanceof MyPromise) {
                val.then(
                    value => {
                        this._value = value;
                        runFulfilled(value);
                    },
                    err => {
                        this._value = err;
                        runRejected(err);
                    }
                );
            } else {
                this._value = val;
                runFulfilled(val);
            }
        };
    
        // 异步调用
        setTimeout(run);
    };

    // reject 时执行的函数
    private _reject = err => {
        if (this._status !== PENDING) return;
    
        // 依次执行失败队列中的函数，并清空队列
        const run = () => {
            this._status = REJECTED;
            this._value = err;
            let cb;
            while ((cb = this._rejectedQueues.shift())) {
                cb(err);
            }
        };
    
        // 为了支持同步的Promise，这里采用异步调用
        setTimeout(run);
    };
  
    // then 方法
    then(onFulfilled?, onRejected?) {
        const { _value, _status } = this;
    
        // 返回一个新的Promise对象
        return new MyPromise((onFulfilledNext, onRejectedNext) => {
        
            // 封装一个成功时执行的函数
            const fulfilled = value => {
                try {
                    if (!isFunction(onFulfilled)) {
                        onFulfilledNext(value);
                    } else {
                        const res = onFulfilled(value);
                        if (res instanceof MyPromise) {
                
                            // 如果当前回调函数返回MyPromise对象，必须等待其状态改变后在执行下一个回调      
                            res.then(onFulfilledNext, onRejectedNext);
                        } else {
                            //否则会将返回结果直接作为参数，传入下一个then的回调函数，并立即执行下一个then的回调函数
                            onFulfilledNext(res);
                        }
                    }
                } catch (err) {
                    // 如果函数执行出错，新的Promise对象的状态为失败
                    onRejectedNext(err);
                }
            };

            // 封装一个失败时执行的函数
            const rejected = error => {
                try {
                    if (!isFunction(onRejected)) {
                        onRejectedNext(error);
                    } else {
                        const res = onRejected(error);
                        if (res instanceof        MyPromise) {
                            // 如果当前回调函数返回MyPromise对象，必须等待其状态改变后在执行下一个回调
                            res.then(onFulfilledNext, onRejectedNext);
                        } else {
                            //否则会将返回结果直接作为参数，传入下一个then的回调函数，并立即执行下一个then的回调函数
                            onFulfilledNext(res);
                        }
                    }
                } catch (err) {
                    // 如果函数执行出错，新的Promise对象的状态为失败
                    onRejectedNext(err);
                }
            };

            switch (_status) {
                // 当状态为pending时，将then方法回调函数加入执行队列等待执行
                case PENDING:
                    this._fulfilledQueues.push(fulfilled);
                    this._rejectedQueues.push(rejected);
                    break;
            
                // 当状态已经改变时，立即执行对应的回调函数
                case FULFILLED:
                    fulfilled(_value);
                    break;
            
                case REJECTED:
                    rejected(_value);
                    break;
            }
        });
    }
  
    // catch 方法
    catch(onRejected) {
        return this.then(undefined, onRejected);
    }
  
    // finally 方法
    finally(cb) {
        return this.then(
            value => MyPromise.resolve(cb()).then(() => value),
            reason =>MyPromise.resolve(cb()).then(() => {
                throw reason;
            })
        );
    }
  
    // 静态 resolve 方法
    static resolve(value) {
        // 如果参数是MyPromise实例，直接返回这个实例
        if (value instanceof MyPromise) return value;
        return new MyPromise(resolve => resolve(value));
    }
  
    // 静态 reject 方法
    static reject(value) {
        return new MyPromise((resolve, reject) => reject(value));
    }
  
    // 静态 all 方法
    static all(list) {
        return new MyPromise((resolve, reject) => {
            // 返回值的集合
            let values = [];
            let count = 0;
            for (let [i, p] of list.entries()) {
            
                // 数组参数如果不是MyPromise实例，先调用MyPromise.resolve
                this.resolve(p).then(
                    res => {
                        values[i] = res;
                        count++;
                    
                        // 所有状态都变成fulfilled时返回的MyPromise状态就变成fulfilled
                        if (count === list.length) resolve(values);
                    },
                    err => {
                        // 有一个被rejected时返回的MyPromise状态就变成rejected
                        reject(err);
                    }
                );
            }
        });
    }
  
    // 添加静态race方法
    static race(list) {
        return new MyPromise((resolve, reject) => {
            for (let p of list) {
                // 只要有一个实例率先改变状态，新的MyPromise的状态就跟着改变
                this.resolve(p).then(
                    res => {
                        resolve(res);
                    },
                    err => {
                        reject(err);
                    }
                );
            }
        });
    }
}
```

### 9. 防抖、节流

防抖函数(debounce)：onscroll 结束时触发一次，延迟执行。

```
// fn是我们需要包装的事件回调, delay是每次推迟执行的等待时间
function debounce(fn, delay) {
    // 定时器
    let timer = null;
  
    // 将debounce处理结果当作函数返回
    return function () {
        // 保留调用时的this上下文
        let context = this;
        // 保留调用时传入的参数
        let args = arguments;

        // 每次事件被触发时，都去清除之前的旧定时器
        if(timer) {
            clearTimeout(timer);
        }
        // 设立新定时器
        timer = setTimeout(function () {
            fn.apply(context, args);
        }, delay);
    }
}

// 用debounce来包装scroll的回调
const better_scroll = debounce(() => console.log('触发了滚动事件'), 1000);
document.addEventListener('scroll', better_scroll);
```

节流函数(throttle)：onscroll 时，每隔一段时间触发一次，像水滴一样。

```
// fn是我们需要包装的事件回调, interval是时间间隔的阈值
function throttle(fn, interval) {
    // last为上一次触发回调的时间
    let last = 0;
  
    // 将throttle处理结果当作函数返回
    return function () {
        // 保留调用时的this上下文
        let context = this;
        // 保留调用时传入的参数
        let args = arguments;
        // 记录本次触发回调的时间
        let now = +new Date();
      
        // 判断上次触发的时间和本次触发的时间差是否小于时间间隔的阈值
        if (now - last >= interval) {
            // 如果时间间隔大于我们设定的时间间隔阈值，则执行回调
            last = now;
            fn.apply(context, args);
        }
    }
}

// 用throttle来包装scroll的回调
const better_scroll = throttle(() => console.log('触发了滚动事件'), 1000);
document.addEventListener('scroll', better_scroll)
```

用 Throttle 来优化 Debounce

```
// fn是我们需要包装的事件回调, delay是时间间隔的阈值
function throttle(fn, delay){
    // last为上一次触发回调的时间, timer是定时器
    let last = 0, timer = null;
    // 将throttle处理结果当作函数返回
    return function () { 
        // 保留调用时的this上下文
        let context = this;
        // 保留调用时传入的参数
        let args = arguments;
        // 记录本次触发回调的时间
        let now = +new Date();
    
        // 判断上次触发的时间和本次触发的时间差是否小于时间间隔的阈值
        if(now - last < delay){
            // 如果时间间隔小于我们设定的时间间隔阈值，则为本次触发操作设立一个新的定时器
            clearTimeout(timer);
            timer = setTimeout(function () {
                last = now;
                fn.apply(context, args);
            }, delay);
        }else{
            // 如果时间间隔超出了我们设定的时间间隔阈值，那就不等了，无论如何要反馈给用户一次响应
            last = now;
            fn.apply(context, args);
        }
    }
}

// 用新的throttle包装scroll的回调
const better_scroll = throttle(() => console.log('触发了滚动事件'), 1000)

document.addEventListener('scroll', better_scroll)
```

### 10. 双向绑定

defineProperty版本：

```
<section>
    输入文本：<input type="text" id="input">
    展示文本：<span id="span"></span>
</section>
<script>
    let input = document.getElementById('input');
    let span = document.getElementById('span');

    let data = {};

    // 数据劫持
    Object.defineProperty(data,"text",{
        //设置text属性时会自动触发set方法，数据变化 --> 修改视图
        set: (newVal)=>{
	        input.value = newVal;
	        span.innerHTML = newVal;
        }
    });

    // 视图更改 --> 数据变化
    input.addEventListener('keyup',(event)=>{
        data.text = event.target.value;
    });
</script>
```

proxy版本：

```
let input = document.getElementById('input');
let span = document.getElementById('span');

let data = {};

// 数据劫持
let handler = {
    set(target, key, value) {
        target[key] = value;
        // 数据变化 --> 修改视图
        input.value = value;
        span.innerHTML = value;
    }
};

let proxy = new Proxy(data, handler);

// 视图更改 --> 数据变化
input.addEventListener('keyup', function(e) {
    proxy.text = e.target.value;
});
```

### 11. typeof 封装

```
function myTypeof(val){
    let type = typeof(val),
        toStr = Object.prototype.toString;

    var res = {
        '[object Object]': 'Object',
        '[object Array]': 'Array',
        '[object Number]': 'object Number',
        '[object String]': 'object String',
        '[object Boolean]': 'object Boolean',
        '[object Date]': 'Date',
        '[object RegExp]': 'RegExp'
    }

    if(val === null){
        return null;
    }else if(type === 'object'){
        let ret = toStr.call(val);
        return res[ret];
    }else{
        return type;
    }
}
```

### 12. 事件绑定封装

```
function addEvent(elem, type, handle){
    if(elem.addEventListener){
        elem.addEventListener(type, handle, false);
    }else if(elem.attachEvent){ // 兼容IE
        elem.attachEvent('on'+type, function(){
            handle.call(elem);
        });
   }else{
        elem['on' + type] = handle;
   }
}
```

### 13. 事件代理（事件委托）

```
<!--输出li中内容-->

<div>
    <ul id="ull">
        <li>
            <a href="javascript:;">anchor label</a>
        </li>
        <li>
            text
        </li>
        <li>
            <span>span1</span>
            <span>span2</span>
        </li>
    </ul>
</div>
<script>
    ;(function(){
        var oUl = document.getElementById("ull");
        oUl.addEventListener('click', function(eve){
            handler(eve)
        }, false);

        function handler(eve){
            var eve = eve || window.event;
            var target = eve.target || eve.srcElement;
          	while(target != oUl){
            	if(target.nodeName.toLowerCase() === 'li'){
                	console.log(target.innerText);
              	}
              	target = target.parentNode;
                //与while循环一起实现，当点击li子标签依然能触发事件
            }
        }
    })();
</script>
```

### 14. Ajax 封装

```
let opt = {
    url: 'test.txt',
    type: 'get',
    data: {
        name:'Jeff',
        age:24
    }
};

function ajax(opt){
    if(opt.url && opt.type){
        let xhr = XMLHttpRequest ? new XMLHttpRequest() : new ActiveXObject('Microsoft.XMLHTTP');

        let url = opt.url,
            type = opt.type.toUpperCase(),
            data = opt.data,
            dataArr = [];

        for(let key in data){
            dataArr.push(key + '=' +data[key]);
        }

        if(type == 'GET'){
            url = url + '?' + dataArr.join('&');
            xhr.open(type, url.replace(/\?$/g,''), true);//true表示异步
            xhr.send();
        }

        if(type == 'POST'){
            xhr.open(type, url, true);
            xhr.setRequestHeader('Content-type', 'application/x-www-form-urlencoded');
            xhr.send();
        }

        //方法1
        xhr.onreadystatechange = function(){
            if(xhr.readyState == 4 && xhr.status == 200){
                var res = xhr.responseText;
                console.log(res);
            }else{
                console.log("Request was unsuccessful: " + xhr.readyState);
            }
        }

        //方法2
        xhr.onload = function(){
            if((xhr.status>=200 && xhr.status<300) || xhr.status==304){
                let res = xhr.responseText;
                console.log(res);
            }else{
                console.log("Request was unsuccessful: " + xhr.status);
            }
        }
    }else{
        console.log("URL or Send Type Error !");
    }
}
```

### 15. Promise封装Ajax

```
let opt = {
    url: 'test.txt',
    type: 'get',
    data: {
        name:'Jeff',
        age:24
    }
};

function ajax(opt){
    let url = opt.url,
        type = opt.type.toUpperCase(),
        data = opt.data,
        dataArr = [];

    for(let key in data){
        dataArr.push(key + '=' +data[key]);
    }

    return new Promise(){
        let xhr = XMLHttpRequest ? new XMLHttpRequest() : new ActiveXObject('Microsoft.XMLHTTP');

        if(type == 'GET'){
            url = url + '?' + dataArr.join('&');
            xhr.open(type, url.replace(/\?$/g,''), true);//true表示异步
            xhr.send();
        }

        if(type == 'POST'){
            xhr.open(type, url, true);
            xhr.setRequestHeader('Content-type', 'application/x-www-form-urlencoded');
            xhr.send();
        }

        xhr.onreadystatechange = function(){
            if(xhr.readyState == 4 && xhr.status == 200){
                let data = xhr.responseText;
                resolve(data);
            }else{
                let errMsg = {
                    'code': xhr.status,
                    'msg': xhr.response
                }
                reject(errMsg);
            }
        }
    }
}

ajax.then(function(data){
    console.log(data);
}).catch(function(err) {
    console.error(err);
})
```