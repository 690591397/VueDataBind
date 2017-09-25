# VueDataBind

# 快速看懂Vue双向数据绑定  
Vue源码封装很完善，看起来可能会有些绕。各位看官可以自行对照源码食用，也可以对照以下代码。仅仅用于原理的说明。   
## 两个核心
在研究之前，得先明白了Vue实现数据绑定的两个核心理念，即：
- ES5的Object.defineProperty()
- 观察者(发布-订阅者)模式
### 关键字get/set
相信我们最先想知道的多半是怎么做数据的改变，而不是对如何实现的思路，掌握核心科技。    
使用 [Object.defineProperty()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) 的 `get／set` 对传入new Vue({})所有数据对象做一个**数据监听，用于在属性获取(get)和设置(set)时，添加对应的逻辑。**    
```

  // ---------------数据监听----------------------
  observe = function(value){
    // 当值不存在，或者不是复杂数据类型时，不再需要继续深入监听
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
    walk(value){
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
            // 通知所有订阅者，数值被改变了
            dep.notify()
        }
    })
  }
```


****
### 观察者(发布-订阅者)模式
**WHY** 为什么要用这个模式？   
观察者模式是开发基于行为的大型应用程序的有力手段。在一次浏览器会话期间，应用程序中可能会断断续续地发生几十次，几百次甚至上千次各种事件。你可以消减为事件注册监听器的次数，让可观察者对象借助一个事件监听器替你处理各种行为并将信息委托给它的所有订阅者，从而降低内存消耗和提高互动性能。这样一来，就不用没完没了地为同样的元素增添新的事件监听器。这样有利于减少系统开销并提高程序的可维护性。（JavaScript设计模式）    
实质就是你可以对程序中某个对象的状态进行观察，并在其发生改变时能得到通知。    
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
        console.log('updata =>')
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

### 思路
在整理完这些核心点之后该，我们来整理一下实现如何整合的思路。    
谁是订阅者？怎么往订阅器添加订阅者？   
首先，我们需要对数据进行监听的是*发布者*,那么就可以在初始化该数据监听的时候(defineReactive)，在函数里面`var dep = new Dep()`。所以监听的每一个数据都是一个发布者。    
那订阅者呢，前面说过，在`get/set`的时候添加对应的逻辑，这就派上用场了。    
**get:**    
我们可以在调用get时，往当前的发布者dep中添加订阅者。**注意！是添加。**  
/* 可以跳过先看整体思路 */  
在这一处有一个比较绕的一个点，就是订阅者的创建。   
订阅者的创建应该伴随的是在页面的某个地方需要用到这个数据，可以说是出现一个 `{{}}`，这时就有一个watcher。    
这时候回过头来看watcher类，在`new Watcher(this, expOrFn, cb)`时，为了初始化订阅者的值，调用了get（)`this.val = this.get() // 获取订阅的值作为自身的值`，而且在watcher的参数里面，可以知道需要获取的是哪一个发布者的值
```
    const exp = this.expOrFn//获取键名，定位到是哪一个发布者
    var val = this.vm._data[exp] //获取发布者的值
```
自然而然的，这一步触发了**发布者的get()**，然后我们再看`Dep.target = this`，我们就可以将整个watcher添加进订阅者的队列了。    
PS：{{}} 这里可以是一个数组，就得对数据进行解析了。还有在数据添加的时需要做一个去重的判断，可以用一个唯一的id或者some()来进行。    
**set**
set需要做的就简单得多了，当发布者的数据发生改变时，会调用set，在这将新值更新完之后，就该通知该发布者的所有订阅者更新信息。
****
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
        new Watcher(this, expOrFn, cb)
    }
    // -------------------------------------------
    _proxy(key){
        Object.defineProperty(this, key, {
            configurable: true,
            enumerable: true,
            get: () => this._data[key],
            set: (val) => {
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