# React、Vue和Angular的比较

[TOC]

# 渲染机制
## React
在页面打开时，React会调用render函数构建一棵`真实的Dom树`，在state/props发生改变的时候，render函数会被再次调用渲染出一棵`Virtual Dom树`，接着，React会用对两棵树进行对比，找到需要更新的地方批量改动。在这个过程中，如何计算出将Virtual Dom树转换成真实Dom树的最少操作— —**Diff算法**。最后，将计算出的全部差异放入`差异队列`后，再一次性去执行`patch`方法完成真实的Dom更新。
 
### Diff算法的三个策略
> 策略一：Dom节点跨层级的移动操作特别少，可忽略不计。  
策略二：两个相同的组件产生类似的Dom结构，不同组件产生不同Dom结构。  
策略三：对于同一层次的一组子节点，它们可以通过唯一的id区分。

#### tree diff
1. 基于策略一，React通过updateDepth对Virtual Dom树进行层级控制，只会对相同层级的Dom节点进行比较，即同一父节点下的所有子节点。当发现节点已经不存在时，则该节点及其子节点会被完全删除掉，不会用于进一步的比较。因此，只需要对树进行一次遍历，就能完成整个Dom树的比较。
2. 当出现节点跨层级移动时，以该节点为根节点的整个树会被重新创建。

#### component diff
1. 基于策略二，对于不同类型的组件，替换整个组件下所有的子节点。
2. 对于同一类型的组件，可以通过`shouldComponentUpdate()`来判断该组件是否需要进行diff算法分析。
```
<A>
  <C/>
</A>
// 由shape1到shape2
<B>
  <C/>
</B>
```
从shape1到shape2节点的生命周期：
```
Shape1 :
A is created 
A render
C is created
C render
C componentDidMount
A componentDidMount

Shape2 :
A componentWillUnmount
C componentWillUnmount
B is created
B render
C is created
C render
C componentDidMount
B componentDidMount
```
由此可以看出，A与其子节点C会被直接删除，然后重新建一个B，C插入。

#### element diff
1. 基于策略三，对于同一层的同组子节点，添加唯一`key`进行区分，若通过`key`发现新旧集合中的节点是相同的节点，则只需进行位置移动。
2. 在开发过程中，尽量减少将最后一个节点移动到列表首部的操作。
```
<div>
  <A />
  <B />
</div>
// 列表一到列表二
<div>
  <A />
  <C />
  <B />
</div>
```
从shape1到shape2节点的生命周期：（给每个节点配一个`key`以后）
```
A render
C is created
C render
B render
A componentDidUpdate
C componentDidMount
B componentDidUpdate
```
给每个节点配一个key，只在指定的位置创建C节点插入。
> 这里要注意的一点是，key值必须是稳定（所以我们不能用Math.random()去创建key），可预测，并且唯一的。

## Vue
与React的区别— —**计算Virtual Dom的差异**。Vue在渲染过程中，会跟踪每个组件的依赖关系，因此系统会准确地知道状态更改时实际需要重新渲染的组件，不需要重新渲染整个组件树。**Vue.js采用数据劫持结合发布者-订阅者模式的方式**，通过`Object.defineProperty()`来劫持各个属性的`setter/getter`，通过watcher监听数据的变化，在数据变动时发布消息给订阅者，触发相应的监听回调。vue渲染的过程如下：
- new Vue，执行初始化
- 挂载$mount方法，通过自定义Render方法、template、el等生成Render函数
- 通过watcher监听数据的变化
- 当数据发生变化时，Render函数执行生成VNode对象
- 通过patch方法，对比新旧VNode对象，通过DOM Diff算法，添加、修改、删除真正的DOM
 

## Angular
 AngularJS是通过**脏值检测**的方式比对数据是否有变更，来决定是否更新视图。AngularJS为scope模型上设置了一个监听队列，用来监听数据变化并更新view。每次绑定一个东西到 view上时AngularJS就会往 $watch 队列里插入一条 $watch，用来检测它监视的 model 里是否有变化的东西。当浏览器接收到可以被 Angular context 处理的事件时，`$digest` 循环就会触发。$digest 会遍历所有的 $watch，从而更新Dom。  

# 数据流
## React
1. React本身是严格的view层，并不是完整的 MVC/MVVM 框架。  
2. React是**单项数据流**，数据主要从父节点传递到子节点（通过props）。如果顶层（父级）的某个props改变了，React会重渲染所有的子节点。
3. props和state的区别

属性 | 值 | 作用 |
:---:|:---:|:---:
props | 只读，不可改变              | 用于组件树中传递数据和配置
state | 可以通过this.setState()修改 | 用于管理组件自身的状态

4. 组件之间的通信
    * 父子组件的通信：
        *  父组件更新组件状态 —–props—–>　子组件更新 
        *  子组件更新父组件状态 —–需要父组件传递回调函数—–> 子组件调用触发
    * 非父子组件的通信：
        * 嵌套不深的非父子组件，可以通过将数据提升到它们最近共同的父组件中，触发事件函数传形参的方式来改变组件的props。
        * 当组件层次很深的时候，可以通过上下文方式（`context`），让子组件直接访问祖先的数据或函数，不需要一层层的传递数据到子组件中。（ps：一般不要用它，使用`redux`管理应用状态）


5. React组件生命周期

<img src="http://static.codeceo.com/images/2016/03/ajs-life.png" width=700 />



- `construtor()` ：创建组件  
- `componentWillMount()` ：组件挂载之前，即组件调用render方法之前调用 
- `componentDidMount()` ： 组件挂载之后，即DOM元素已经插入页面后调用  
- `componentWillReceiveProps()`：父组件发生render的时候，子组件从父组件接收到新的 props 之前调用
- `shouldComponentUpdate()`：组件挂载之后每次调用setState后都会调用该函数判断是否需要重新渲染组件，默认返回true 
- `componentWillUpdate()`：组件开始重新渲染之前调用
- `componentDidUpdate()`：组件重新渲染并且把更改变更到真实的 DOM以后调用
- `render()` ：渲染，react中的核心函数  
- `componentWillUnmount()`：组件对应的DOM元素从页面中删除之前调用，一般在componentDidMount注册的事件需要在这里删除


## Vue
1. Vue是**MVVM**（Model-View-ViewModel）模式。
2. Vue是**双向数据绑定**，Vue实例中的data与其渲染的DOM元素的内容保持一致，无论谁被改变，另一方会相应的更新为相同的数据。双向绑定是在同一个组件内，将数据和视图绑定起来，和父子组件之间的通信并无什么关联。
3. **组件之间的通信采用单向数据流**，为了组件间更好的解耦，Vue不推荐子组件修改父组件的数据，直接修改props会抛出警告。

### React vs Vue
> 数据流的区别

<img src="http://5b0988e595225.cdn.sohucs.com/images/20180730/ba4b02d044dd414ba4778935fe02647a.jpeg" width=600 />

> 组件通信的区别

<img src="http://5b0988e595225.cdn.sohucs.com/images/20180730/76674802ec584f97b27d360b71b25a67.jpeg" width=600 />

## Angular
**MVW**（Model-View-Whatever）模式，双向数据绑定。

# HTML&&CSS
## React
1. JSX
```
render () {
   let { items } = this.props；
   let children；
   if ( items.length >0) {
     children = ( 
      <ul> 
        { items.map( item => <li key={item.id}>{item.name}</li> ) } 
      </ul>
      )
   } else {
     children = <p>No items found.</p>
   }
   return(
      <div className=‘list-container’>
        {children}
      </div>
  )
}
```
2. 组件作用域内的CSS  
通过 CSS-in-JS 的方案实现。即：把 HTML 和 CSS 全都整进 JavaScript 了。

## Vue
1. Templates
```
<template>
<div class=“list-container”>
 <ul v-if=“items.length”>
   <li v-for=“item in items”>
{{ item.name }}
   </li>
</ul>
<p v-else>No items found.</p>
</div>
</template>
```
2. 单文件组件CSS  
同一个文件里完全控制CSS，将其作为代码的一部分，类似 style 的标签。即：template，script，style写在一个vue文件里作为一个组件。

## Angular
AngularJS 的指令（用于渲染指令的DOM模板） 可用于创建自定义的 HTML 标记。

# 状态管理
## React
> Redux /Mobx  

Redux三个基本原则：  
- 单一数据源（Single source of truth）  
- State 是只读的（State is read-only）  
- 使用纯函数执行修改（Changes are made with pure functions）

Redux的主要关注点：
1. **Action**：一个JavaScript对象，描述动作相关信息，主要包含`type`属性和`payload`属性：
    - `type`：action 类型
    - `payload`：负载数据
2. **Reducer**：定义应用状态如何响应不同动作（action），如何更新状态
3. **Store**：管理action和reducer及其关系的对象，主要提供以下功能：
    - 维护应用状态并支持访问状态（`getState()`）
    - 支持监听action的分发，更新状态（`dispatch(action)`）
    - 支持订阅store的变更（`subscribe(listener)`）
4. **异步流**：由于Redux所有对store状态的变更，都应该通过action触发，异步任务（通常都是业务或获取数据任务）也不例外，而为了不将业务或数据相关的任务混入React组件中，就需要使用其他框架配合管理异步任务流程，如`redux-thunk`，`redux-saga`等。

### Redux vs Mobx：
1. **函数式和面型对象**  
    - Redux遵循函数式编程（FP），如reducer就是一个纯函数，对于相同的输入总是输出相同的结果。
    - Mobx提倡面向对象编程（OOP）和响应式编程（RP），通常将状态包装成可观察对象，一旦状态对象变更，就能自动获得更新。
2. **单一store和多store**  
    - Redux将所有共享的应用数据集中在一个store中。
    - Mobx通常按模块将应用状态划分，在多个独立的store中管理。
3. **JavaScript对象和可观察对象**  
    - Redux默认以JavaScript原生对象形式存储数据，需要手动追踪所有状态对象的变更
    - Mobx使用可以监听的可观察对象，当其变更时将自动触发监听
4. **不可变（Immutable）和可变（Mutable）**  
    - Redux状态对象通常是不可变的（Immutable）
    - Mobx中可以直接使用新值更新状态对象
5. **React-redux和Mobx-react**  
    - 使用Redux和React应用连接时，需要使用`react-redux`提供的`Provider`和`connect`：
        - `Provider`：负责将store注入React应用；
        - `connect`：负责将store state注入容器组件，并选择特定状态作为容器组件props传递；
    - 对于Mobx而言，同样需要两个步骤：
        - `Provider`：使用`mobx-react`提供的`Provider`将所有stores注入应用；
        - 使用`inject`将特定store注入某组件，store可以传递状态或action；然后使用`observer`保证组件能响应store中的可观察对象（observable）变更，即store更新，组件视图响应式更新。

## Vue
> Vuex
### Redux vs Vuex

对比 | Redux | Vuex |
:---:|:---:|:---:
核心对象         | store    | store
数据存储         | state    | state
状态更新提交接口 | dispatch | commit
状态更新提交参数 | 状态更新提交参数 | 带type和payload的mutation提交对象/参数
状态更新计算     | reducer  | mutation handler
限制             | reducer必须是纯函数，不支持异步 | mutation handler必须是非异步方法
特性             | 支持中间件 | 支持带缓存的getter，用于获取state经过某些计算后的值
流向             | view—>actions—>reducer—>state变化—>view变化（同步异步一样） |  view—>commit—>mutations—>state变化—>view变化（同步操作）<br>view—>dispatch—>actions—>mutations—>state变化—>view变化（异步操作）

###  React-redux vs Vuex
React-redux
- 状态注入组件：`<Provider/>组件结合connect方法`
- 容器组件：`通过connect关联了state的组件，并被传入dispatch接口`
- 展示组件：不与state或dispatch直接产生关系
- 特性：connect支持`mapStatesToProps`方法，用于自定义映射

Vuex
- 状态注入组件：Vue.use(Vuex)将Vuex应用为全局的plugin，再将store对象传入根Vue实例
- 容器组件：`没有这个概念`
- 展示组件：在组件中可以获取this.$store.state.*，也进行this.$store.commit()等等
- 特性：Vuex提供mapState，mapGetter，mapMutation等方法，用于生成store内部属性对组件内部属性的映射

## Angular
> ngrx/redux

# 路由
## React
react-router
- 使用时，路由器Router就是React的一个组件。
- Router组件本身只是一个容器，真正的路由要通过Route组件定义。
- Route组件定义了URL路径与组件的对应关系。你可以同时使用多个Route组件。
```
<Router history={hashHistory}>
  <Route path="/" component={App}/>
  <Route path="/repos" component={Repos}/>
  <Route path="/about" component={About}/>
</Router>
```
## Vue
vue-router 
- 使用时，将组件(components)映射到路由(routes)，然后告诉 vue-router 在哪里渲染它们。
```
<div id="app">
  <h1>Hello App!</h1>
  <p>
    <router-link to="/foo">Go to Foo</router-link>
    <router-link to="/bar">Go to Bar</router-link>
  </p>
  <router-view></router-view>
</div>
```
## Angular
angular-route
- Angular2以及以后的版本路由器使用浏览器的history.pushState 进行导航。
```
const appRoutes: Routes = [
  { path: 'add', component: MenuAddComponent },
  { path: 'detail/:id', component: MenuDetailComponent },
  { path: 'home', component: EmptyComponent ，data:{title:'测试1'}},
  { path: '', redirectTo: '/home', pathMatch: 'full' },  //空目录，重定向处理
  { path: '**', component: EmptyComponent } //通配符，其他路由失败的，会跳转到这个视图
];
```

# 构建工具
## React
create-react-app
```
全局安装 create-react-app：npm install -g create-react-app
新建项目：create-react-app 文件名
进入项目：cd 文件名
启动项目：npm start
```
## Vue
vue-cli
```
全局安装 vue-cli：npm install -g vue-cli    
新建项目：vue init webpack 文件名
进入项目：cd 文件名
启动项目：npm run dev
```
## Angular
angular-cli
```
全局安装 angular-cli：npm  install @angular/cli –g   
新建项目：ng new 文件名
进入项目：cd 文件名
启动项目：npm start或者ng server
```

# 其他方面
对比 | React | Vue | Angular
:---:|:---:|:---:|:---:
出现年月     | 2013-3    | 2013-7 | 2010-1
开发与维护   | Facebook  | 尤雨溪 | Google
团队人数     | 未知      | 16     | 36
支持原生开发 | react-native/react-native-renderer | Weex  | NativeScript、Ionic
服务端预渲染 | next.js | nuxt.js | Angular Universal
依赖标准     | ES6 | ES5/ES6 | TypeScript
单元测试     | Jest+Enzyme | karma + mocha + chai | Jasmine
学习曲线     | 中等 | 简单 | 陡峭
Github star<br>（统计于2018/5/25）           | 96651 | 95284 | 58492
Github 代码贡献者人数<br>（统计于2018/5/25） | 1184 | 190 | 635
日评星数<br>（最近一年）  | 80.2 | 111.8 | 33.5

# 总结
React
- 灵活
- 拥有大型的技术生态系统
- 良好的组件化设计
- 函数式编程
- 同时适用于Web端和原生APP

Vue
- 平缓的学习曲线
- 干净的代码
- 轻量级的框架。

Angular
- 基于 TypeScript
- 面向对象编程。
