### 常见的VUE知识点

#### vue数据双向绑定原理

采用数据劫持结合发布者-订阅者模式的方式，通过 Object.defineProperty() 来劫持各个属性的setter，getter，在数据变动时发布消息给订阅者，触发相应监听回调。当把一个普通 Javascript 对象传给 Vue 实例来作为它的 data 选项时，Vue 将遍历它的属性，用 Object.defineProperty() 将它们转为 getter/setter。用户看不到 getter/setter，但是在内部它们让 Vue 追踪依赖，在属性被访问和修改时通知变化。

vue的数据双向绑定 将MVVM作为数据绑定的入口，整合Observer，Compile和Watcher三者，通过Observer来监听自己的model的数据变化，通过Compile来解析编译模板指令（vue中是用来解析 {{}}），最终利用watcher搭起observer和Compile之间的通信桥梁，达到数据变化 —>视图更新；视图交互变化（input）—>数据model变更双向绑定效果。

defineProperty 它接收三个参数，要操作的对象，要定义或修改的对象属性名，属性描述符。

代码演示

```js
<body>
  <div type='text' id='txt'>
    <p id='show'></p>
</div>
</body>
<script type='text/javascript'>
  var obj={}
  Object.defineProperty(obj,'txt',{
    get: function(){
      return obj
    },
    set: function(newValue){
      document.getElementById('txt').value=newValue
      document.getElementById('show').innerHtml=newValue
    }
  })
  document.addEventlistener('keyup',function(e){
    obj.txt=e.target.value
  })
```



#### vue computed原理

1. `computed`是计算一个新的属性，并将该属性挂载到vm（Vue实例）上，而`watch`是监听已经存在且已挂载到`vm`上的数据，所以用`watch`同样可以监听`computed`计算属性的变化（其它还有`data`、`props`）
2. `computed`本质是一个惰性求值的观察者，具有缓存性，只有当依赖变化后，第一次访问 `computed` 属性，才会计算新的值，而`watch`则是当数据发生变化便会调用执行函数
3. 从使用场景上说，`computed`适用一个数据被多个数据影响，而`watch`适用一个数据影响多个数据；

[计算属性原理](https://segmentfault.com/a/1190000016368913?utm_source=tag-newest)

1. 当组件初始化的时候，`computed`和`data`会分别建立各自的响应系统，`Observer`遍历`data`中每个属性设置`get/set`数据拦截
2. 初始化`computed`会调用`initComputed`函数
   1. 注册一个`watcher`实例，并在内实例化一个`Dep`消息订阅器用作后续收集依赖（比如渲染函数的`watcher`或者其他观察该计算属性变化的`watcher`）
   2. 调用计算属性时会触发其`Object.defineProperty`的`get`访问器函数
   3. 调用`watcher.depend()`方法向自身的消息订阅器`dep`的`subs`中添加其他属性的`watcher`
   4. 调用`watcher`的`evaluate`方法（进而调用`watcher`的`get`方法）让自身成为其他`watcher`的消息订阅器的订阅者，首先将`watcher`赋给`Dep.target`，然后执行`getter`求值函数，当访问求值函数里面的属性（比如来自`data`、`props`或其他`computed`）时，会同样触发它们的`get`访问器函数从而将该计算属性的`watcher`添加到求值函数中属性的`watcher`的消息订阅器`dep`中，当这些操作完成，最后关闭`Dep.target`赋为`null`并返回求值函数结果。
3. 当某个属性发生变化，触发`set`拦截函数，然后调用自身消息订阅器`dep`的`notify`方法，遍历当前`dep`中保存着所有订阅者`wathcer`的`subs`数组，并逐个调用`watcher`的 `update`方法，完成响应更新。

#### vue编译器结构图

```cpp
├── scripts ------------------------------- 构建相关的文件
│   ├── git-hooks ------------------------- git钩子的目录
│   ├── alias.js -------------------------- 别名配置
│   ├── config.js ------------------------- rollup配置文件 打包用的是rollup
│   ├── build.js -------------------------- 对 config.js 中所有的rollup配置进行构建
│   ├── ci.sh ----------------------------- 持续集成运行的脚本
│   ├── release.sh ------------------------ 用于自动发布新版本的脚本
├── dist ---------------------------------- 构建后文件的输出目录
├── examples ------------------------------ 存放一些使用Vue开发的应用案例
├── flow ---------------------------------- 类型声明，使用开源项目 [Flow](https://flowtype.org/)
├── packages ------------------------------ 存放独立发布的包的目录
├── test ---------------------------------- 包含所有测试文件
├── src ----------------------------------- 源码
│   ├── compiler -------------------------- 编译器代码的存放目录将 template 编译为 render 函数
│   ├── core ------------------------------ 存放通用的，与平台无关的代码
│   │   ├── observer ---------------------- 响应系统，包含数据观测的核心代码
│   │   ├── vdom -------------------------- 虚拟DOM创建(creation)和打补丁(patching)的代码
│   │   ├── instance ---------------------- Vue构造函数设计相关的代码
│   │   ├── global-api -------------------- Vue构造函数挂载全局方法(静态方法)或属性的代码
│   │   ├── components -------------------- 抽象出来的通用组件
│   ├── server ---------------------------- 服务端渲染(server-side rendering)的相关代码
│   ├── platforms ------------------------- 平台特相关代码，不同平台的不同构建的入口文件存放地
│   │   ├── web --------------------------- web平台
│   │   │   ├── entry-runtime.js ---------- 运行时构建的入口，不包含模板(template)到render函数的编译器。
│   │   │   ├── entry-runtime-with-compiler.js -- 独立构建版本的入口，在 entry-runtime 的基础上添加了模板(template)到render函数的编译器
│   │   │   ├── entry-compiler.js --------- vue-template-compiler 包的入口文件
│   │   │   ├── entry-server-renderer.js -- vue-server-renderer 包的入口文件
│   │   │   ├── entry-server-basic-renderer.js -- 输出 packages/vue-server-renderer/basic.js 文件
│   │   ├── weex -------------------------- 混合应用
│   ├── sfc ------------------------------- 包含单文件组件(.vue文件)的解析逻辑，用于vue-template-compiler包
│   ├── shared ---------------------------- 包含整个代码库通用的代码
├── package.json --------------------------  你懂得
├── yarn.lock ----------------------------- yarn 锁定文件
├── .editorconfig ------------------------- 针对编辑器的编码风格配置文件
├── .flowconfig --------------------------- flow 的配置文件
├── .babelrc ------------------------------ babel 配置文件
├── .eslintrc ----------------------------- eslint 配置文件
├── .eslintignore ------------------------- eslint 忽略配置
├── .gitignore ---------------------------- git 忽略配置
```

宏观的理论上讲讲compile编译分成 parse(解析)、optimize(优化) 与 generate(生成) 三个阶段，最终需要得到 render function。
 parse
 parse 会用正则等方式解析 template 模板中的指令、class、style等数据，形成AST。

optimize
 optimize 的主要作用是标记 static 静态节点，这是 Vue 在编译过程中的一处优化，后面当 update 更新界面时，会有一个 patch 的过程， diff 算法会直接跳过静态节点，从而减少了比较的过程，优化了 patch 的性能。

generate
 generate 是将 AST 转化成 render function 字符串的过程，得到结果是 render 的字符串以及 staticRenderFns 字符串。

在经历过 parse、optimize 与 generate 这三个阶段以后，组件中就会存在渲染 VNode 所需的 render function 了。



**vuejs源码中的 template 解析**

```js
var template = options.template;
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template);
        }
      } else if (template.nodeType) {
        template = template.innerHTML;
      } else {
        {
          warn('invalid template option:' + template, this);
        }
        return this
      }
    } else if (el) {
      template = getOuterHTML(el);
    }
```

**逻辑主要步骤如下：**

- 先判断是否有`template`属性
- 如果没有，则直接通过`el`中的html代码作为模版
- 如果有，判断是否是字符串(非字符串的形式暂不讨论)
- 是字符串的情况下，是否以`#`字符开头
- 如果是，则获取对应id的`innerHTML`作为模版
- 如果不是以`#`字符开头，则直接作为作为模版

#### 生命周期

- **beforeCreate**
- **created**
- **beforeMount**
- **mounted**
- **beforeUpdate**
- **updated**
- **beforeDestroy**
- **destroyed**

**1. 在beforeCreate和created钩子函数之间的生命周期**

在这个生命周期之间，进行**初始化事件，进行数据的观测**，可以看到在**created**的时候数据已经和**data属性进行绑定**（放在data中的属性当值发生改变的同时，视图也会改变）。
注意看下：此时还是没有el选项

**2. created钩子函数和beforeMount间的生命周期**

在这一阶段发生的事情还是比较多的。

首先会判断对象是否有**el选项**。**如果有的话就继续向下编译，如果没有**el选项**，则停止编译，也就意味着停止了生命周期，直到在该vue实例上调用vm.$mount(el)。**此时注释掉代码中:

```js
el: '#app',
```

然后运行可以看到到created的时候就停止了：

**3. beforeMount和mounted 钩子函数间的生命周期**

可以看到此时是给vue实例对象添加**$el成员**，并且替换掉挂在的DOM元素。因为在之前console中打印的结果可以看到**beforeMount**之前el上还是undefined。

**mounted**

在mounted之前h1中还是通过**{{message}}**进行占位的，因为此时还有挂在到页面上，还是JavaScript中的虚拟DOM形式存在的。在mounted之后可以看到h1中的内容发生了变化。

**5. beforeUpdate钩子函数和updated钩子函数间的生命周期**

当vue发现data中的数据发生了改变，会**触发对应组件的重新渲染**，先后调用**beforeUpdate**和**updated**钩子函数。我们在console中输入

**6.beforeDestroy和destroyed钩子函数间的生命周期**

**beforeDestroy**钩子函数在实例销毁之前调用。在这一步，实例仍然完全可用。
**destroyed**钩子函数在Vue 实例销毁后调用。调用后，Vue 实例指示的所有东西都会解绑定，所有的事件监听器会被移除，所有的子实例也会被销毁。

[生命周期详解](https://segmentfault.com/a/1190000011381906)

#### vue组件通信

##### 方法一、`props`/`$emit`

父组件A通过props的方式向子组件B传递，B to A 通过在 B 组件中 $emit, A 组件中 v-on 的方式实现

**父组件通过props向下传递数据给子组件。注：组件中的数据共有三种形式：data、props、computed**

子组件向父组件传值（通过事件形式）

**子组件通过events给父组件发送消息，实际上就是子组件把自己的数据发送到父组件**

```js
// 子组件
<template>
  <header>
    <h1 @click="changeTitle">{{title}}</h1>//绑定一个点击事件
  </header>
</template>
<script>
export default {
  name: 'app-header',
  data() {
    return {
      title:"Vue.js Demo"
    }
  },
  methods:{
    changeTitle() {
      this.$emit("titleChanged","子向父组件传值");//自定义事件  传递值“子向父组件传值”
    }
  }
}
</script>
```

```js
// 父组件
<template>
  <div id="app">
    <app-header v-on:titleChanged="updateTitle" ></app-header>//与子组件titleChanged自定义事件保持一致
   // updateTitle($event)接受传递过来的文字
    <h2>{{title}}</h2>
  </div>
</template>
<script>
import Header from "./components/Header"
export default {
  name: 'App',
  data(){
    return{
      title:"传递的是一个值"
    }
  },
  methods:{
    updateTitle(e){   //声明这个函数
      this.title = e;
    }
  },
  components:{
   "app-header":Header,
  }
```

##### 方法二、`$emit`/`$on`

**这种方法通过一个空的Vue实例作为中央事件总线（事件中心），用它来触发事件和监听事件,巧妙而轻量地实现了任何组件间的通信，包括父子、兄弟、跨级**。当我们的项目比较大时，可以选择更好的状态管理解决方案vuex。

具体实现方式

```js
 var Event=new Vue();
    Event.$emit(事件名,数据);
    Event.$on(事件名,data => {});
```

假设兄弟组件有三个，分别是A、B、C组件，C组件如何获取A或者B组件的数据

```js
<div id="itany">
    <my-a></my-a>
    <my-b></my-b>
    <my-c></my-c>
</div>
<template id="a">
  <div>
    <h3>A组件：{{name}}</h3>
    <button @click="send">将数据发送给C组件</button>
  </div>
</template>
<template id="b">
  <div>
    <h3>B组件：{{age}}</h3>
    <button @click="send">将数组发送给C组件</button>
  </div>
</template>
<template id="c">
  <div>
    <h3>C组件：{{name}}，{{age}}</h3>
  </div>
</template>
<script>
var Event = new Vue();//定义一个空的Vue实例
var A = {
    template: '#a',
    data() {
      return {
        name: 'tom'
      }
    },
    methods: {
      send() {
        Event.$emit('data-a', this.name);
      }
    }
}
var B = {
    template: '#b',
    data() {
      return {
        age: 20
      }
    },
    methods: {
      send() {
        Event.$emit('data-b', this.age);
      }
    }
}
var C = {
    template: '#c',
    data() {
      return {
        name: '',
        age: ""
      }
    },
    mounted() {//在模板编译完成后执行
     Event.$on('data-a',name => {
         this.name = name;//箭头函数内部不会产生新的this，这边如果不用=>,this指代Event
     })
     Event.$on('data-b',age => {
         this.age = age;
     })
    }
}
var vm = new Vue({
    el: '#itany',
    components: {
      'my-a': A,
      'my-b': B,
      'my-c': C
    }
});    
</script>
```

`$on` 监听了自定义事件 data-a和data-b，因为有时不确定何时会触发事件，一般会在 mounted 或 created 钩子中来监听。

##### 方法三、vuex

Vuex实现了一个单向数据流，在全局拥有一个State存放数据，当组件要更改State中的数据时，必须通过Mutation进行，Mutation同时提供了订阅者模式供外部插件调用获取State数据的更新。而当所有异步操作(常见于调用后端接口异步获取更新数据)或批量的同步操作需要走Action，但Action也是无法直接修改State的，还是需要通过Mutation来修改State的数据。最后，根据State的变化，渲染到视图上。



- Vue Components：Vue组件。HTML页面上，负责接收用户操作等交互行为，执行dispatch方法触发对应action进行回应。
- dispatch：操作行为触发方法，是唯一能执行action的方法。
- actions：**操作行为处理模块,由组件中的`$store.dispatch('action 名称', data1)`来触发。然后由commit()来触发mutation的调用 , 间接更新 state**。负责处理Vue Components接收到的所有交互行为。包含同步/异步操作，支持多个同名方法，按照注册的顺序依次触发。向后台API请求的操作就在这个模块中进行，包括触发其他action以及提交mutation的操作。该模块提供了Promise的封装，以支持action的链式触发。
- commit：状态改变提交操作方法。对mutation进行提交，是唯一能执行mutation的方法。
- mutations：**状态改变操作方法，由actions中的`commit('mutation 名称')`来触发**。是Vuex修改state的唯一推荐方法。该方法只能进行同步操作，且方法名只能全局唯一。操作之中会有一些hook暴露出来，以进行state的监控等。
- state：页面状态管理容器对象。集中存储Vue components中data对象的零散数据，全局唯一，以进行统一的状态管理。页面显示所需的数据从该对象中进行读取，利用Vue的细粒度数据响应机制来进行高效的状态更新。
- getters：state对象读取方法。图中没有单独列出该模块，应该被包含在了render中，Vue Components通过该方法读取全局state对象。

##### 方法四、`$attrs`/`$listeners`

多级组件嵌套需要传递数据时，通常使用的方法是通过vuex。但如果仅仅是传递数据，而不做中间处理，使用 vuex 处理，未免有点大材小用。为此Vue2.4 版本提供了另一种方法----`$attrs`/`$listeners`

- `$attrs`：包含了父作用域中不被 prop 所识别 (且获取) 的特性绑定 (class 和 style 除外)。当一个组件没有声明任何 prop 时，这里会包含所有父作用域的绑定 (class 和 style 除外)，并且可以通过 v-bind="$attrs" 传入内部组件。通常配合 interitAttrs 选项一起使用。
- `$listeners`：包含了父作用域中的 (不含 .native 修饰器的) v-on 事件监听器。它可以通过 v-on="$listeners" 传入内部组件

##### 方法五、`provide/inject`

Vue2.2.0新增API,这对选项需要一起使用，**以允许一个祖先组件向其所有子孙后代注入一个依赖，不论组件层次有多深，并在起上下游关系成立的时间里始终生效**。一言而蔽之：祖先组件中通过provider来提供变量，然后在子孙组件中通过inject来注入变量。
**provide / inject API 主要解决了跨级组件间的通信问题，不过它的使用场景，主要是子组件获取上级组件的状态，跨级组件间建立了一种主动提供与依赖注入的关系**。

##### 方法六、`$parent` / `$children`与 `ref`

- `ref`：如果在普通的 DOM 元素上使用，引用指向的就是 DOM 元素；如果用在子组件上，引用就指向组件实例
- `$parent` / `$children`：访问父 / 子实例

需要注意的是：这两种都是直接得到组件实例，使用后可以直接调用组件的方法或访问数据。我们先来看个用 `ref`来访问组件的例子：

```js
// component-a 子组件
export default {
  data () {
    return {
      title: 'Vue.js'
    }
  },
  methods: {
    sayHello () {
      window.alert('Hello');
    }
  }
}
```

```js
// 父组件
<template>
  <component-a ref="comA"></component-a>
</template>
<script>
  export default {
    mounted () {
      const comA = this.$refs.comA;
      console.log(comA.title);  // Vue.js
      comA.sayHello();  // 弹窗
    }
  }
</script>
```

不过，**这两种方法的弊端是，无法在跨级或兄弟间通信**



常见使用场景可以分为三类：

- 父子通信：

父向子传递数据是通过 props，子向父是通过 events（`$emit`）；通过父链 / 子链也可以通信（`$parent` / `$children`）；ref 也可以访问组件实例；provide / inject API；`$attrs/$listeners`

- 兄弟通信：

Bus；Vuex

- 跨级通信：

Bus；Vuex；provide / inject API、`$attrs/$listeners`

[组件通信](https://segmentfault.com/a/1190000019208626)

#### mmvm模式

**核心是提供对View 和 ViewModel 的双向数据绑定，这使得ViewModel 的状态改变可以自动传递给 View，即所谓的数据双向绑定**。

Vue.js 是一个提供了 MVVM 风格的双向数据绑定的 Javascript 库，专注于View 层。它的核心是 MVVM 中的 VM，也就是 ViewModel。 ViewModel负责连接 View 和 Model，保证视图和数据的一致性，这种轻量级的架构让前端开发更加高效、便捷。

**在MVVM架构下，View 和 Model 之间并没有直接的联系，而是通过ViewModel进行交互，Model 和 ViewModel 之间的交互是双向的， 因此View 数据的变化会同步到Model中，而Model 数据的变化也会立即反应到View 上**。

**MVC**

 MVC 架构模式对于简单的应用来看是OK 的，也符合**软件架构**的分层思想。 但实际上，随着H5 的不断发展，人们更希望使用H5 开发的应用能和Native 媲美，**或者接近于原生App 的体验效果**，于是前端应用的复杂程度已不同往日，今非昔比。这时前端开发就暴露出了三个痛点问题：

　　1、 开发者在代码中大量调用相同的 DOM API，处理繁琐 ，操作冗余，使得代码难以维护。

　　2、大量的DOM 操作使页面渲染性能降低，加载速度变慢，影响用户体验。

　　3、 当 Model 频繁发生变化，开发者需要主动更新到View ；当用户的操作导致 Model 发生变化，开发者同样需要将变化的数据同步到Model 中，这样的工作不仅繁琐，而且很难维护复杂多变的数据状态。

　　其实，早期jquery 的出现就是为了前端能更简洁的操作DOM 而设计的，但它只解决了第一个问题，另外两个问题始终伴随着前端一直存在。

#### mvc模式理解

**为什么前端要工程化，要是使用MVC？** 

　　相对 HTML4，HTML5 最大的亮点是**它为移动设备提供了一些非常有用的功能**，使得 HTML5 具备了开发App的能力， HTML5开发App 最大的好处就是**跨平台、快速迭代和上线，节省人力成本和提高效率**，因此很多企业开始对传统的App进行改造，逐渐用H5代替Native，到2015年的时候，市面上大多数App 或多或少嵌入都了H5 的页面。既然要用H5 来构建 App， 那View 层所做的事，就不仅仅是简单的数据展示了，它不仅要管理复杂的数据状态，还要处理移动设备上各种操作行为等等。因此，前端也需要工程化，也需要一个类似于MVC 的框架来管理这些复杂的逻辑，使开发更加高效。 但这里的 MVC 又稍微发了点变化：

　　View ：UI布局，展示数据

　　Model ：管理数据

　　Controller ：响应用户操作，并将 Model 更新到 View 上

#### vue dom diff

###### 2.1虚拟DOM+diff为什么快?

（1）真实DOM的创建需要完成默认样式，挂载相应的属性，注册相应的Event Listener ...效率是很低的。如果元素比较多的时候，还涉及到嵌套，那么元素的属性和方法等等就会很多，效率更低。  **`diff算法对DOM进行原地复用，减少DOM创建性能耗费`**
 （2）**`虚拟DOM很轻量，对虚拟DOM操作快`**
 （3）页面的排版与重绘也是一个相当耗费性能的过程。

通过对虚拟DOM进行diff，逐步找到更新前后vdom的差异，然后将差异反应到DOM树上（也就是patch）**`减少过多DOM节点排版与重绘损耗`**。特别要提一下Vue的patch是即时的，并不是打包所有修改最后一起操作DOM（React则是将更新放入队列后集中处理），朋友们会问这样做性能很差吧？实际上现代浏览器对这样的DOM操作做了优化，并太大差别。

###### 2.2虚拟DOM+diff的缺点?

引入虚拟DOM实际上有优点也缺点。
 (1)尺寸
 更多的功能意味着更多的代码。
 (2)内存
 虚拟DOM需要在内存中的维护一份DOM的副本。在DOM更新速度和使用内存空间之间取得平衡。
 (3)不是适合所有情况
 如果虚拟DOM大量更改，这是合适的。但是少量的，频繁的更新的话，虚拟DOM将会花费更多的时间处理计算的工作。所以，如果一个DOM节点相对较少页面，用虚拟DOM，它实际上有可能会更慢。

##### 3.Vue2.x的虚拟DOM diff原理

##### 3.1 patch函数

diff的过程就是调用patch函数，就像打补丁一样修改真实dom。

**3.2 patchVnode函数**

节点的比较有5种情况 patchVnode用于比较新旧节点的的区别

**3.3 updateChildren函数**

更新dom树

#### vuex

`VueX`是适用于在`Vue`项目开发时使用的状态管理工具。试想一下，如果在一个项目开发中频繁的使用组件传参的方式来同步`data`中的值，一旦项目变得很庞大，管理和维护这些值将是相当棘手的工作。为此，`Vue`为这些被多个组件频繁使用的值提供了一个统一管理的工具——`VueX`。在具有`VueX`的Vue项目中，我们只需要把这些值定义在VueX中，即可在整个Vue项目的组件中使用。

+ 初始化`store`下`index.js`中的内容

```js
import Vue from 'vue'
import Vuex from 'vuex'

//挂载Vuex
Vue.use(Vuex)

//创建VueX对象
const store = new Vuex.Store({
    state:{
        //存放的键值对就是所要管理的状态
        name:'helloVueX'
    }
})

export default store
```

+ 将store挂载到当前项目的Vue实例当中去

```js
// 打开main.js
import Vue from 'vue'
import App from './App'
import router from './router'
import store from './store'

Vue.config.productionTip = false

/* eslint-disable no-new */
new Vue({
  el: '#app',
  router,
  store,  //store:store 和router一样，将我们创建的Vuex实例挂载到这个vue实例中
  render: h => h(App)
})
```

+ 在组件中使用Vuex

```js
<template>
    <div id='app'>
        name:
        <h1>{{ $store.state.name }}</h1>
    </div>
</template>
...,
methods:{
    add(){
      console.log(this.$store.state.name)
    }
},
...
```

**VueX中的核心内容**

在VueX对象中，其实不止有`state`,还有用来操作`state`中数据的方法集，以及当我们需要对`state`中的数据需要加工的方法集等等成员。

成员列表：

- state     存放状态
- mutations   state成员操作 `mutations`是操作`state`数据的方法的集合，比如对该数据的修改、增加、删除等等。
- getters     加工state成员给外界
- actions     异步操作
- modules   模块化状态管理

例如，我们编写一个方法，当被执行时，能把下例中的name值修改为`"jack"`,我们只需要这样做

```js
//index.js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

const store = new Vuex.store({
    state:{
        name:'helloVueX'
    },
    mutations:{
        //es6语法，等同edit:funcion(){...}
        edit(state){
            state.name = 'jack'
        }
    }
})

export default store


......
而在组件中，我们需要这样去调用这个mutation——例如在App.vue的某个method中:
this.$store.commit('edit')

在实际生产过程中，会遇到需要在提交某个mutation时需要携带一些参数给方法使用。
this.$store.commit('edit',15)

....
当需要多参提交时，推荐把他们放在一个对象中来提交:
this.$store.commit('edit',{age:15,sex:'男'})

...
接收挂载的参数：
 edit(state,payload){
            state.name = 'jack'
            console.log(payload) // 15或{age:15,sex:'男'}
        }
...
this.$store.commit({
    type:'edit',
    payload:{
        age:15,
        sex:'男'
    }
})

```

**Getters**

可以对state中的成员加工后传递给外界

Getters中的方法有两个默认参数

- state 当前VueX对象中的状态对象
- getters 当前getters对象，用于将getters下的其他getter拿来用

```js
getters:{
    nameInfo(state){
        return "姓名:"+state.name
    },
    fullInfo(state,getters){
        return getters.nameInfo+'年龄:'+state.age
    }  
}
```

组件中调用

```js
this.$store.getters.fullInfo
```

**Actions**

由于直接在`mutation`方法中进行异步操作，将会引起数据失效。所以提供了Actions来专门进行异步操作，最终提交`mutation`方法。

`Actions`中的方法有两个默认参数

- `context` 上下文(相当于箭头函数中的this)对象
- `payload` 挂载参数

例如，我们在两秒中后执行`2.2.2`节中的`edit`方法

由于`setTimeout`是异步操作，所以需要使用`actions`

```js
actions:{
    aEdit(context,payload){
        setTimeout(()=>{
            context.commit('edit',payload)
        },2000)
    }
}
```

在组件中调用:

```
this.$store.dispatch('aEdit',{age:15})
...
改进:
  aEdit(context,payload){
        return new Promise((resolve,reject)=>{
            setTimeout(()=>{
                context.commit('edit',payload)
                resolve()
            },2000)
        })
    }
```

 **Models**

当项目庞大，状态非常多时，可以采用模块化管理模式。Vuex 允许我们将 store 分割成**模块（module）**。每个模块拥有自己的 `state、mutation、action、getter`、甚至是嵌套子模块——从上至下进行同样方式的分割。

```js
models:{
    a:{
        state:{},
        getters:{},
        ....
    }
}

```

组件内调用模块a的状态：

```js
this.$store.state.a

...
而提交或者dispatch某个方法和以前一样,会自动执行所有模块内的对应type的方法
this.$store.commit('editKey')
this.$store.dispatch('aEditKey')


规范目录结构
store:.
│  actions.js
│  getters.js
│  index.js
│  mutations.js
│  mutations_type.js   ##该项为存放mutaions方法常量的文件，按需要可加入
│
└─modules
        Astore.js
```

#### vue-router

##### vue-router实现原理

SPA(single page application):单一页面应用程序，只有一个完整的页面；它在加载页面时，不会加载整个页面，而是只更新某个指定的容器中内容。**单页面应用(SPA)的核心之一是: 更新视图而不重新请求页面**;vue-router在实现单页面前端路由时，提供了两种方式：Hash模式和History模式；根据mode参数来决定采用哪一种方式。

**1、Hash模式：**

**vue-router 默认 hash 模式 —— 使用 URL 的 hash 来模拟一个完整的 URL，于是当 URL 改变时，页面不会重新加载。** hash（#）是URL 的锚点，代表的是网页中的一个位置，单单改变#后的部分，浏览器只会滚动到相应位置，不会重新加载网页，也就是说**hash 出现在 URL 中，但不会被包含在 http 请求中，对后端完全没有影响，因此改变 hash 不会重新加载页面**；同时每一次改变#后的部分，都会在浏览器的访问历史中增加一个记录，使用”后退”按钮，就可以回到上一个位置；所以说**Hash模式通过锚点值的改变，根据不同的值，渲染指定DOM位置的不同数据。hash 模式的原理是 onhashchange 事件(监测hash值变化)，可以在 window 对象上监听这个事件**。

**2、History模式：**

由于hash模式会在url中自带#，如果不想要很丑的 hash，我们可以用路由的 history 模式，只需要在配置路由规则时，加入"mode: 'history'",**这种模式充分利用了html5 history interface 中新增的 pushState() 和 replaceState() 方法。这两个方法应用于浏览器记录栈，在当前已有的 back、forward、go 基础之上，它们提供了对历史记录修改的功能。只是当它们执行修改时，虽然改变了当前的 URL ，但浏览器不会立即向后端发送请求**。



```csharp
//main.js文件中
const router = new VueRouter({
  mode: 'history',
  routes: [...]
})
```

当你使用 history 模式时，URL 就像正常的 url，例如 [http://yoursite.com/user/id](https://links.jianshu.com/go?to=http%3A%2F%2Fyoursite.com%2Fuser%2Fid)，比较好看！
 不过这种模式要玩好，还需要后台配置支持。因为我们的应用是个单页客户端应用，如果后台没有正确的配置，当用户在浏览器直接访问 [http://oursite.com/user/id](https://links.jianshu.com/go?to=http%3A%2F%2Foursite.com%2Fuser%2Fid) 就会返回 404，这就不好看了。
 所以呢，**你要在服务端增加一个覆盖所有情况的候选资源：如果 URL 匹配不到任何静态资源，则应该返回同一个 index.html 页面，这个页面就是你 app 依赖的页面。**



```cpp
export const routes = [ 
  {path: "/", name: "homeLink", component:Home}
  {path: "/register", name: "registerLink", component: Register},
  {path: "/login", name: "loginLink", component: Login},
  {path: "*", redirect: "/"}]
```

**3、使用路由模块来实现页面跳转的方式**

- 方式1：直接修改地址栏
- 方式2：this.$router.push(‘路由地址’)
- 方式3：`<router-link to="路由地址"></router-link>`

**4.vue-router参数传递**

1.用name传递参数

在路由文件src/router/index.js里配置name属性

```bash
routes: [
    {
      path: '/',
      name: 'Hello',
      component: Hello
    }
]
...

模板里(src/App.vue)用$route.name来接收 比如：<p>{{ $route.name}}</p>
```

**通过`<router-link to="路由地址"></router-link>` 标签中的to传参**

这种传参方法的基本语法：



```ruby
<router-link :to="{name:xxx,params:{key:value}}">valueString</router-link>
```

比如先在src/App.vue文件中



```xml
<router-link :to="{name:'hi1',params:{username:'jspang',id:'555'}}">Hi页面1</router-link>
```

然后把src/router/index.js文件里给hi1配置的路由起个name,就叫hi1.

**利用url传递参数----在配置文件里以冒号的形式设置参数。**

```css
{
    path:'/params/:newsId/:newsTitle',
    component:Params
}
```

**使用path来匹配路由，然后通过query来传递参数**

```xml
<router-link :to="{ name:'Query',query: { queryId:  status }}" >
     router-link跳转Query
</router-link>

...
对应路由配置：
   {
     path: '/query',
     name: 'Query',
     component: Query
   }

...
this.$route.query.queryId
```

**vue-router配置子路由(二级路由)**



修改router/index.js代码，子路由的写法是在原有的路由配置下加入children字段。

```csharp
   routes: [
    {
      path: '/',
      name: 'HelloWorld',
      component: HelloWorld,
      children: [{path: '/h1', name: 'H1', component: H1},//子路由的<router-view>必须在HelloWorld.vue中出现
        {path: '/h2', name: 'H2', component: H2}
      ]
    }
  ]
```

**单页面多路由区域操作**

在一个页面里我们有两个以上``区域，我们通过配置路由的js文件，来操作这些区域的内容

1.App.vue文件，在``下面新写了两行``标签,并加入了些CSS样式



```xml
<template>
  <div id="app">
    <img src="./assets/logo.png">
       <router-link :to="{name:'HelloWorld'}"><h1>H1</h1></router-link>
       <router-link :to="{name:'H1'}"><h1>H2</h1></router-link>
    <router-view></router-view>
    <router-view name="left" style="float:left;width:50%;background-color:#ccc;height:300px;"/>
    <router-view name="right" style="float:right;width:50%;background-color:yellowgreen;height:300px;"/>
  </div>
</template>
```

2.需要在路由里配置这三个区域，配置主要是在components字段里进行



```css
export default new Router({
    routes: [
      {
        path: '/',
        name: 'HelloWorld',
        components: {default: HelloWorld,
          left:H1,//显示H1组件内容'I am H1 page,Welcome to H1'
          right:H2//显示H2组件内容'I am H2 page,Welcome to H2'
        }
      },
      {
        path: '/h1',
        name: 'H1',
        components: {default: HelloWorld,
          left:H2,//显示H2组件内容
          right:H1//显示H1组件内容
        }
      }
    ]
  })
```

上边的代码我们编写了两个路径，一个是默认的‘/’，另一个是‘/Hi’.在两个路径下的components里面，我们对三个区域都定义了显示内容。



**$route 是“路由信息对象”，包括 path，params，hash，query，fullPath，matched，name 等路由信息参数。**



**$router 是“路由实例”对象，即使用 new VueRouter创建的实例，包括了路由的跳转方法，钩子函数等。**



**`$router.push`和`$router.replace`的区别**：

- 使用push方法的跳转会向 history 栈添加一个新的记录，当我们点击浏览器的返回按钮时可以看到之前的页面。
- 使用replace方法不会向 history 添加新记录，而是替换掉当前的 history 记录，即当replace跳转到的网页后，‘后退’按钮不能查看之前的页面。



用户会经常输错页面，当用户输错页面时，我们希望给他一个友好的提示页面，这个页面就是我们常说的404页面。vue-router也为我们提供了这样的机制。

1. 设置我们的路由配置文件（/src/router/index.js）



```css
{
   path:'*',
   component:Error
}
```

这里的path:'*'就是输入地址不匹配时，自动显示出Error.vue的文件内容

1. 在/src/components/文件夹下新建一个Error.vue的文件。简单输入一些有关错误页面的内容。



```xml
<template>
    <div>
        <h2>{{ msg }}</h2>
    </div>
</template>
<script>
export default {
  data () {
    return {
      msg: 'Error:404'
    }
  }
}
</script>
```

此时我们随意输入一个错误的地址时，便会自动跳转到404页面



