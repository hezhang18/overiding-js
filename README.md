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

浅拷贝

无法拷贝对象、数组这样的引用值属性，目标对象修改引用值属性会导致原对象的也被修改。

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

深拷贝

可以拷贝对象、数组这样的引用值属性。

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