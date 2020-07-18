# mvvm
从创建一个mvvm实例，学习vue双向数据绑定原理。下图是双向数据绑定原理图，下文将围绕这个图进行原理分析整理。
[!img](https://github.com/DMQ/mvvm/raw/master/img/2.png)

一、创建一个mvvm实例，经过两个步骤
（1）数据劫持，绑定响应式数据
（2）模板解析编译
```js
function Vue(options) {
    this.data = options.data;
    var data = this.data;
    var id = options.el

    observe(data,this);//监听数据

    var dom = nodeToFragment(document.getElementById(id),this);//节点拦截
    document.getElementById(id).appendChild(dom);
}

var vm = new Vue({
    el: 'app',
    data: {
        text:'hello Vue!'
    }
})
```

二、响应式的数据绑定
通过defineReactive方法实现数据劫持，代码核心原理就是通过Object.defineProperty将数据定义为访问器属性，通过set和get方法对数据进行劫持监听。
以下是简化后的源码：

```js
function defineReactive(obj,key,val){
    const dep = new Dep()

    Object.defineProperty(obj,key,{
        get:function() {
            //添加订阅者到主题对象Dep,即绑定Watcher到当前数据
            if(Dep.target) {
                dep.addSub(Dep.target)
            }
            return val
        },
        set:function(newVal) {
            if(newVal === val || (newVal !== newVal && val !== val)) {
                return
            }
            val = newVal

            //当数据更新时，通知所有订阅者watcher
            dep.notify()
        }
    })
}

//对vue实例中的data数据设置为访问器属性
function observer(obj:Object,vm) {
    Object.keys(obj).forEach(function(key) {
        defineReactive(vm,key,obj[key])
    })
}

```

（1）get方法主要做两件事，
* 首先，添加订阅者到主题对象Dep,即绑定watcher到当前数据
* 然后，返回value值，

（2）set方法主要做两件事：
* 首先，给数据赋予新值
* 然后，通知所有订阅者watcher更新数据

响应式数据实现还差一步，监听view视图的数据变化，将变化更新到model模型，将model中的数据给view视图中的数据赋值。这一步在complie模板解析中进行。
```js
//绑定数据，数据初始化,每个watcher将自己添加到相应属性的dep中
function compile(node,vm) {
    var reg = /\{\{(.*)\}\}/;
    //节点类型为元素
    if(node.nodeType === 1) {
        var attr = node.attributes;
        //解析属性
        for(var i = 0;i < attr.length;i++) {
            console.log('attr[i].nodeName=',attr[i].nodeName);
            if(attr[i].nodeName == 'v-model') {
                var name = attr[i].nodeValue;
                node.addEventListener('input', function(e) {
                    //给对应的数据赋值，从而出发set方法。
                    vm[name] = e.target.value;
                })
                //绑定数据，触发get方法
                node.value = vm[name];
                node.removeAttribute('v-model');
            }
        }
    }
    //节点类型为text
    if(node.nodeType === 3) {
        if(reg.test(node.nodeValue)) {
            var name = RegExp.$1;
            name = name.trim();//去除前后空格
            // node.nodeValue = vm.data[name];

            //在编译 HTML 过程中，为每个与 data 关联的节点生成一个 Watcher，将关联属性添加到dep中
            new Watcher(vm,node,name);
        }
    }
}

```

compile做了两件事：
(1)解析html,初始化视图
(2)订阅数据变化，绑定更新函数，即new Watcher()

三、订阅/发布模式
以上两步只实现了输入框数据变化更新到model模型，即一对一的响应式数据，要实现一对多的数据响应，还需要引入订阅/发布模式。
订阅发布模式定义了一种一对多的模式，让多个观察者同事监听同一个主题对象，当主题对象的状态发生改变时就会通知所有的观察者对象进行更新。

订阅者模式，可以添加多个订阅者，更新信息
```js
// 三个订阅者
// var sub1 = { update:function() { console.log('sub1'); } };
// var sub2 = { update:function() { console.log('sub2'); } };
// var sub3 = { update:function() { console.log('sub3'); } };

//一个主题对象
function Dep() {
    // this.subs = [sub1,sub2,sub3];
    this.subs = [];
}
Dep.prototype = {
    addSub: function(sub) {
        this.subs.push(sub);
    },
    notify: function() {
        this.subs.forEach(sub => {
            sub.update();
        })
    }
}

Dep.target = null;

//一个发布者publisher
var pub = {
    publish: function() {
        dep.notify();
    }
}

//发布者发布消息，主题对象执行notify()方法，从而触发订阅者执行update方法，更新视图
var dep = new Dep();
pub.publish();
```

在编译 HTML 过程中，为每个与 data 关联的节点生成一个 Watcher。Watcher 函数中发生了什么呢？

```js
function Watcher(vm,exp,cb) {
    this.cb = cb;
    this.vm = vm;
    this.exp = exp;
    this.value = this.get();
}

Watcher.prototype = {
    update: function() {
        this.run();
    },
    run: function() {
        var value = this.vm.data[this.exp];
        var oldVal = this.value;
        if(value !== oldVal) {
            this.value = value;
            this.cb.call(this.vm,value,oldVal);
        }
    },
    get: function () { 
        Dep.target = this;
        var value = this.vm.value[this.exp];
        Dep.target = null;
        return value;
    }
}
```

Watcher做了这几件事：
首先，执行了get方法，将自己赋值给全局主题对象Dep.target，get 的方法读取了 vm 的访问器属性，从而触发了访问器属性的 get 方法，get 方法中将该 watcher 添加到了对应访问器属性的 dep 中；
再次，获取属性的值，然后更新视图。
最后，将 Dep.target 设为空。因为它是全局变量，也是 watcher 与 dep 关联的唯一桥梁，任何时刻都必须保证 Dep.target 只有一个值。




