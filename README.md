# JS 源码实现

阅读源码，学习前人的经验以提升自己的能力。源码往往是前人留下的最佳实践，我们跟着前人的脚步去学习更会让我们事半功倍。

### 1. new 运算符

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

### 2. 原型继承

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

### 4. bind

原理：

bind与call、apply一样可以改变this指向；bind方法即可在bind中传参，也可在执行时传参；实例化时this指向会失效（即，new一个对象时指向实例化对象，否则指向被绑定的context）。

实现：

1. 一般方法

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

1. 圣杯模式

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

### 5. 