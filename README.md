单向绑定非常简单，就是把Model绑定到View，当我们用JavaScript代码更新Model时，View就会自动更新。
有单向绑定，就有双向绑定。如果用户更新了View，Model的数据也自动被更新了，这种情况就是双向绑定。
这么个能让人从dom操作解放出来浑身通泰的东西，能不研究一下它的原理？
****
Vue源码的英文解释很详细。以下代码，仅仅用于原理的说明。   

**阅读顺序建议粗略过代码，对照着思路再看代码。**
## 两个核心

在研究之前，得先明白了Vue实现数据绑定的两个核心理念，即：
- [Object.defineProperty()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) 
监听数据的变动
- 观察者(发布-订阅者)模式
数据对应的逻辑操作
**它们的关系又是如何？**
一句话描述，一个页面在多处订阅使用了同一个数据，用defineProperty监听其改变，并由发布者通知  订阅者去更新它所持有的数据。

### 关键字get/set 
使用 [Object.defineProperty()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) 的 `get／set` 对传入new Vue({})所有数据对象做一个**数据监听，用于在属性获取(get)和设置(set)时，添加对应的逻辑。**    
```

  // ---------------数据监听----------------------
  observe = function(value){
    // 是否监听
    if(!value || typeof value !== 'object'){
        return
    }
    return new Observer(value)
  }
  // ------
  class Observer{
    constructor(value){
        this.value = value
        this.walk(value)
    }
    walk(value){   //监听数据的所有属性
        Object.keys(value).forEach(key => this.convert(key, value[key]))
    }
    convert(key, val){
        defineReactive(this.value, key, val)
    }
  }
  defineReactive = function(obj, key, val){
    var dep = new Dep()
    // 给当前属性的值添加监听
    var chlidOb = observe(val)
    Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get: ()=> {
            //判断是否有watcher需要添加
            if(Dep.target){
                dep.addSub(Dep.target)
            }
            // -------
            return val
        },
        set: (newVal) => {
            if(val === newVal) return
            val = newVal
            // -------
            // 对新值进行监听
            chlidOb = observe(newVal)
            // 通知所有订阅者更新数据
            dep.notify()
        }
    })
  }
```
将一些的枝枝叶叶去掉，剩下的`Object.defineProperty`就可以对一个数据监听了。就不重复贴代码了。

****

### 观察者(发布-订阅者)模式
**WHY** 为什么要用这个模式？   
观察者模式是开发基于行为的大型应用程序的有力手段。在一次浏览器会话期间，应用程序中可能会断断续续地发生几十次，几百次甚至上千次各种事件。你可以消减为事件注册监听器的次数，让可观察者对象借助一个事件监听器替你处理各种行为并将信息委托给它的所有订阅者，从而降低内存消耗和提高互动性能。这样一来，就不用没完没了地为同样的元素增添新的事件监听器。这样有利于减少系统开销并提高程序的可维护性。（JavaScript设计模式）  
频繁的数据操作与此模式非常的契合   
观察者模式实质就是你可以对程序中某个对象的状态进行观察，并在其发生改变时能得到通知。    
观察者模式存在两个角色:    
- 观察者(发布者)
- 被观察者(订阅者)    
****
**发布者:**
- [] 一个用来管理订阅者的数组
- addSub() 添加订阅者
- notify() 用于发布消息，通知订阅者有新的订阅信息   
```
// ------------------------------------
  class Dep{
    constructor(){
        this.subs = []  //管理订阅者队列
    }
    addSub(sub){
        // 去重复
        var alreadyExists = this.subs.some( (el) => {
            return el === sub
        })
        if (!alreadyExists) {
            this.subs.push(sub)
        }
    }
    notify(){
        // 通知所有的订阅者(Watcher)，触发订阅者的相应逻辑处理
        this.subs.forEach((sub) => sub.update())
    }
  }
```

**订阅者:**
- value 自身的值，在这里是用来保存发布者发布过来的值
- updata() 接收发布者更新通知
```
// ----------------------------------
  class Watcher{
    constructor(vm, expOrFn, cb){
        this.vm = vm // 整个实例
        this.cb = cb // 当数据更新时想要做的事情
        this.expOrFn = expOrFn // 被订阅的数据
        this.val = this.get() // 获取订阅的值作为自身的值
    }
    // 更新数据
    update(){
        this.run()
    }
    run(){
        const val = this.get()
        
        if(val !== this.val){
            this.val = val;
            this.cb.call(this.vm)
        }
    }
    get(){
        // 当前订阅者(Watcher)读取发布者的值
        Dep.target = this 
        const exp = this.expOrFn //获取键名，用来定位到是哪一个发布者
        var val = this.vm._data[exp] //获取发布者的值
        Dep.target = null
        return val;
    }
    
  }
```
****

### 如何整合
在整理完这些核心点之后该，我们拥有了零件。接下来该如何组装起来呢？
既然用的是*观察者模式*
谁是发布者？怎么添加订阅？   
首先，我们监听的每一个数据都应该是一个发布者，这样就可以在数据发生改变的时候通知到各个订阅者。那么，就可以在初始化该数据监听的时候(defineReactive)，在函数里面`var dep = new Dep()`。
```
defineReactive = function(obj, key, val){
    var dep = new Dep()
    ...
}
```       
那订阅者呢，前面说过，在`get/set`的时候添加对应的逻辑，这就派上用场了。    
**get:**    
我们可以在调用get时，往当前的发布者dep中添加订阅者。**注意！是添加。**  
/*  */  
在这一处有一个比较绕的一个点，就是订阅者的创建。   
订阅者的创建应该伴随的是在页面的某个地方需要用到这个数据，可以说是出现一个 `{{}}`，这时就有一个watcher。    
这时候回过头来看watcher类，在`new Watcher(this, expOrFn, cb)`时，为了初始化订阅者的值，调用了get（）`this.val = this.get() // 获取订阅的值作为自身的值`，而且在watcher的参数里面，可以知道需要获取的是哪一个发布者的值
```
    const exp = this.expOrFn//获取键名，定位到是哪一个发布者
    var val = this.vm._data[exp] //获取发布者的值
```
自然而然的，这一步触发了**发布者的get()**，然后我们再看`Dep.target = this`，我们就可以将整个watcher添加进订阅者的队列了。       
**set:**    
set需要做的就简单得多了，当发布者的数据发生改变时，会调用set，在这将新值更新完之后，就该通知该发布者的所有订阅者更新信息。
****
保留了添加订阅者和更新发布者数据两个功能。
```
class Vue{
    constructor(options = {}){
        // 简化了$options的处理
        this.$options = options
        // 简化了对data的处理
        let data = this._data = this.$options.data
        // 将所有data最外层属性代理到Vue实例上
        Object.keys(data).forEach(key => this._proxy(key))
        // 监听数据
        console.log('listen data :')
        observe(data)
    }
    // 对外暴露调用订阅者的接口，内部主要在指令中使用订阅者
    // ------------------{{}}---------------------
    $watch(expOrFn, cb){
        //
        new Watcher(this, expOrFn, cb)
    }
    // -------------------------------------------
    _proxy(key){
        Object.defineProperty(this, key, {
            configurable: true,
            enumerable: true,
            get: () => this._data[key],
            set: (val) => {
                //更新数据
                this._data[key] = val
            } 
        })
    }
    
  }


  // test-----------
  var t = new Vue({
      data: {
          name: 'NAME'
      }
  })
  t.$watch('name', () => console.log('cb:the one'))
  t.$watch('name', () => console.log('cb:the second'))
  t.name = 'ghjk'
```

到这Vue双向数据绑定的简单逻辑基本也就完成了，还剩下的是同界面的解析交互了。

相信在现在的前端环境，双向数据绑定几乎是选择框架的一种标准了。    
粗略算起来，研究了小半个星期的Vue双向数据绑定，略有所得，做个记录，不足或有错之处，望指出。
