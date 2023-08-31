# Vue 快速入门

1. 导入vue.js

```html
<script src="https://unpkg.com/vue@next"></script>
```

2. 在页面中声明一个将要被 vue 所控制的 DOM 区域，即 MVVM 中的 View

```html
<div id="app">
  {{message}}
</div>
```

3. 创建 view model 实例对象（vue 实例对象）

```html
const hello = {
	// 指定数据源，即 MVVM 中的 Model
	data: function() {
		return {
			message: 'Hello Vue!'
		}
	}
}
const app = Vue.createApp(hello)
app.mount('#app') // 指定当前 Vue 实例要控制页面的那个区域
```

4. 渲染绑定v-bind语法,等一系列基础语法

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
</head>
<body>
    <div id="app">
        {{ message }}
        <p v-html="link"></p>
        <a :href="link2">Google</a>
        <p><input type="text" :placeholder="inputValue"></p>
        <img :src="imgSrc" :style="{width:w}" alt="">

        <!--js表达式-->
        <hr/><!--分割线-->
        <p>计算= {{number + 1}}</p>
        <p>三元表达式= {{ok ? 'True' : 'False'}}</p>
        <p>js内置函数= {{message.split('').reverse().join('')}}</p>
        <p :id="'list-' + id">id选择器名称定义</p>
        <p>取属性= {{user.name}}</p>

        <!--事件绑定-->
        <hr/><!--分割线-->
        <h3>count = {{count}}</h3>
        <button v-on:click="addCount">+1</button>
        <button @click="count-=1">-1</button>

        <!--条件渲染指令-->
        <!--
            v-show是通过css方式隐藏，
            而v-if如果为flag=false标签都不会创建
        -->
        <hr/><!--分割线-->
        <p v-if="flag">请求成功 -- 被 v-if 控制</p>
        <p v-show="flag">请求成功 -- 被 v-show 控制</p>
        <p><button @click="flag = !flag">改变flag值</button></p>

        <!-- v-else和v-else-if -->
        <hr/><!--分割线-->
        <p v-if="count > 5">count > 5</p>
        <p v-else>count <= 5</p>
        <p v-if="type === 'A'">优秀</p>
        <p v-else-if="type === 'B'">良好</p>
        <p v-else-if="type === 'C'">一般</p>
        <p v-else>差</p>

        <!-- 列表渲染指令 ,v-model双向绑定，页面改变影响js中的值，js中的值改变也影响页面
            使用列表组件化开发必须有 :key="user.id" 绑定唯一索引,不要用index索引，勾选后，有可能变化
        -->
        <hr/><!--分割线-->
        <p><input type="text" v-model="name"></p>
        <button @click="addNewUser">添加用户</button>
        <ul>
            <li v-for="(u, index) in userList" :key="u.id">索引是={{index}}, 姓名={{u.name}}</li>
        </ul>
    </div>


    <script>
    const { createApp, ref } = Vue

    const app = createApp({
        data() {
            return {
                message: 'Hello, Vue!',
                link: '<a href="http://www.baidu.com">百度</a>',
                link2: 'http://www.google.com',
                inputValue: '请输入用户名',
                imgSrc: 'https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/Jbne6y.jpg',
                w: '500px',
                // js表达式
                number: 9,
                ok: false,
                id: 3,
                user: {
                    name: 'yunqing'
                },
                count: 0,
                flag: false,
                type: 'A',
                userList: [
                    {id:1, name:'张三'},
                    {id:2, name:"李四"},
                    {id:3, name:'王五'}
                ],
                name: '',
                nextId: 4,
            }
        },
        methods: {
            // 点击按钮count++
            addCount() {
                this.count += 1
            },
            addNewUser() {
                // 列表头部添加
                this.userList.unshift({id:this.nextId, name:this.name})
                this.name=''
                this.nextId++
            },
        },
    }).mount('#app')
    </script>

</body>
</html>
```

## 组件化开发

1. npm前端包管理器

![](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/bxLfOC.png)

2. Vue cli 脚手架

> 1. vue cli 是官方提供的脚手架
> 2. 用于快速搭建一个热加载及构件生产版本等功能的单页面应用
> 3. vue cli 基于 webpack 构件，也可以通过项目内配置文件进行配置
> 4. 安装 sudo npm install -g @vue/cli
> 5. 创建 sudo vue create demo
> 6. 新手不要选带eslint检查的 选最后一个manually手动
> 7. 按空格取消掉 Linter/ Formatter
> 8. 3.x  In package.json 相当于 pom.xml
> 9. N 不需要保存作为将来的选项

3. vue中组件的构成

> 1. vue中规定组件的后缀是 .vue
> 2. 每个.vue组件都由3部分组成分别是
> 3. template,组件的模板结构，可以包含 html标签及其他组件
> 4. script,组件的js代码
> 5. style,组件的css样式

## 第三方组件 element-ui

1. 组件之间的传值

- 可以由内部的 Data 提供数据，也可以由父组件通过 prop 的方式传值
- 兄弟组件之间可以通过 Vuex 等统一数据源提供数据共享

```vue
<template>
  <div>
    <h1>{{ title }}</h1>
    <h2>{{ custom }}</h2>
  </div>
</template>

<script>
export default {
    // props 定义一个组件标签的属性，相当于 Hello 标签下有一个 custom 属性
    props: ["custom"],
    name: "Hello",
    data() {
        return {
            title: "孤注一掷"
        }
    }
}
</script>
```

2. element-ui介绍

> 1. Element 是饿了吗开源的前端框架，简洁优雅，提供了Vue/React/Angular多个版本；
>
> 2. 文档地址：https://element.eleme.cn/#/zh-CN
>
> 3. 安装 sudo npm i element-ui
>
> 4. 完整引入在 main.js 中写入以下内容：
>
>    ```javascript
>    import Vue from 'vue';
>    import ElementUI from 'element-ui';
>    import 'element-ui/lib/theme-chalk/index.css';
>    import App from './App.vue';
>    
>    Vue.use(ElementUI);
>    
>    new Vue({
>      el: '#app',
>      render: h => h(App)
>    });
>    ```

3. 第三方图标库

> 1. 由于 Element-ui 提供的字体图标较少，Font Awesome 是很好的选择
>
> 2. Font Awesome 提供了675个可缩放矢量图标，可以使用 CSS 更改大小、颜色、阴影等；
>
> 3. 文档 https://fontawesome.com/docs/web/use-with/vue/add-icons
>
> 4. ```bash
>    # for Vue 2.x
>    sudo npm i --save @fortawesome/vue-fontawesome@latest-2
>    ```

## Axios网络请求

1. Axios简介

> 1. Axios是一个基于promise 网络请求库
> 1. 在浏览器端使用 XMLHttpRequests 发送网络请求，完成JSON自动解析
> 1. 安装 sudo npm i axios
> 1. 地址 https://www.axios-http.cn/
> 1. 在main.js全局引入，或者在具体组件.vue中引入  import axios from 'axios'

2. 发送网络请求

```js
axios.get('/user?id=1')
	.then(function(response) {
     // 处理成功的情况
  	 console.log(response);
   })
	 .catch(function(error) {
     // 处理错误的情况
     console.log(error);
   })
   .then(function() {
  	 // 总会执行
   });
```

上述请求也可以写成这样

```js
axios.get('/user', {
  	params: {
      id:1
    }
  })
	.then(function(response) {
     // 处理成功的情况
  	 console.log(response);
   })
	 .catch(function(error) {
     // 处理错误的情况
     console.log(error);
   })
   .then(function() {
  	 // 总会执行
   });
```

post请求

```js
// 自动把请求体中的参数转化成json格式，后端 @RequestBody接收
axios.post('/user', {
  	id:1,
    name:"axios"
  })
	.then(function(response) {
     // 处理成功的情况
  	 console.log(response);
   })
	 .catch(function(error) {
     // 处理错误的情况
     console.log(error);
   })
   .then(function() {
  	 // 总会执行
   });
```

// 或者这样发一个post请求

```js
axios({
  method: 'post',
  url: '/user',
  data: {
    id:1,
    name:'axios'
  }
});
```



异步回调问题

```js
// 支持 async/await用法
async function getUser() {
  try {
    const res = await.axios.get('/user?id=1');
    console.log(res);
  } catch(error) {
    console.log(error);
  }
}
```

3. 生命周期

```vue
<script>
export default {
    // props 定义一个组件标签的属性，相当于 Hello 标签下有一个 custom 属性
    props: ["custom"],
    name: "Hello",
    methods: {
      tableRowClassName({row, rowIndex}) {
        if (rowIndex % 2 === 1) {
          return 'warning-row';
        } else {
          return 'success-row';
        }
      }
    },
    created:function() {
      console.log('初始化时加载,一般axios调用在这里写')
      this.$http.get('/record/getAll', {
        params: {
          page:1,
          pageSize:5
        }
      })
      .then((response)=> {
        // 处理成功的情况
        let data = response.data;
        let record = data.data.records;
        console.log('--------' + record)
        this.tableData = record
      })
      .catch(function(error) {
        // 处理错误的情况
        console.log(error);
      })
      .then(function() {
        // 总会执行
      });
    },
    mounted:function() {
      console.log('被挂载时加载')
    },
    data() {
        return {
            title: "孤注一掷",
            tableData: []
        }
    }
}
</script>
```

4. 与 Vue 整合

> 1. 多个组件用到 axios,不要每一个导入一遍
> 2. 每次发送请求都需要填写完整路径，`http://localhost:port`可以全局配置

```js
// main.js
// 配置请求根路径
axios.defaults.baseURL='http://localhost:8080'

// 将axios 作为全局自定义属性，每个组件可在内部直接访问 （Vue2）直接this.$http.post()
Vue.prototype.$http = axios
// 将axios 作为全局自定义属性，每个组件可在内部直接访问 （Vue3）直接this.$http.get()
app.config.globalProperties.$http = axios
```

5. axios 表格取值三元表达式

```vue
<!--后端返回 0 或 1 表示是或否-->

<el-table-column
  :formatter="formatType"
  prop="type"
  label="类型"
  width="180">
</el-table-column>

<script>
	methods: {
    // 表格颜色渲染
    tableRowClassName({row, rowIndex}) {
      if (rowIndex % 2 === 1) {
        return 'warning-row';
      } else {
        return 'success-row';
      }
    },
    formatType(row, column) {
      return row.type == '0' ? "出账" : row.type == 1 ? "入账" : "NULL";
    },
  },
</script>
```

## VueRouter 路由安装使用

1. 介绍

> 1. Vue 路由 vue-router 是官方的路由插件，轻松管理 SPA 项目中组件的切换
> 2. Vue 单页面应用是基于路由和组件的，路由用于设定访问路径，并将路径和组件映射起来
> 3. vue-router 目前有 3.x 和 4.x 版本， vue-router 3.x 只能结合 vue2 进行使用，4.x只能结合 vue3 进行使用
> 4. 安装 npm i vue-router@3

2. 声明路由和占位标签

```vue
<!--App.vue 写在根组件中-->
<template>
  <div id="app">
    <div id="router">
      <p>
        <!-- 使用 router-link 组件来导航. -->
        <!-- 通过传入 `to` 属性指定链接. -->
        <!-- <router-link> 默认会被渲染成一个 `<a>` 标签 -->
        <router-link to="/hello">首页</router-link>
        <router-link to="/show">数据展示</router-link>
        <router-link to="/test">充数测试</router-link>
      </p>
      <!-- 路由出口 -->
      <!-- 路由匹配到的组件将渲染在这里 -->
      <router-view></router-view>
    </div>
    <!--<img alt="Vue logo" src="./assets/logo.png">-->
    <Hello custom="孤注一掷100"/>
  </div>
</template>
```

3. 创建路由模块

  在项目中创建 index.js 路由模块，加入以下代码

```js
// 0. 如果使用模块化机制编程，导入Vue和VueRouter，要调用 Vue.use(VueRouter)
import VueRouter from "vue-router"
import Vue from "vue"

// 1. 定义 (路由) 组件。
// 可以从其他文件 import 进来
// const Foo = { template: '<div>foo</div>' }
import Test from '../components/Test.vue'
import Show from '../components/Show.vue'
import Hello from '../components/Hello.vue'

Vue.use(VueRouter)

// 2. 定义路由
// 每个路由应该映射一个组件。 其中"component" 可以是
// 通过 Vue.extend() 创建的组件构造器，
// 或者，只是一个组件配置对象。
// 我们晚点再讨论嵌套路由。
const routes = [
    // 首页重定向到 /hello 对应的组件
    { path: '/', redirect: '/hello' },
    { path: '/hello', component: Hello },
    { path: '/show', component: Show },
    { path: '/test', component: Test },
  ]
  
  // 3. 创建 router 实例，然后传 `routes` 配置
  // 你还可以传别的配置参数, 不过先这么简单着吧。
  const router = new VueRouter({
    routes // (缩写) 相当于 routes: routes
  })

  // 4. 创建和挂载根实例。
// 记得要通过 router 配置参数注入路由，
// 从而让整个应用都有路由功能
// 这里导出 router ,去 main.js中导入、挂载
export default router
```

4. 挂载到 main.js

```js
// 导入配置好的路由
import router from './router/index.js';

new Vue({
  render: h => h(App),
  // 挂载
  router
}).$mount('#app')
```

5. 嵌套路由

在 Show.vue 中，还存在子路由和子路由占位符

```vue
<!--Show.vue-->
<template>
  <div>
    <h1>数据统计</h1>
      <p>
        <!-- 使用 router-link 组件来导航. -->
        <!-- 通过传入 `to` 属性指定链接. -->
        <!-- <router-link> 默认会被渲染成一个 `<a>` 标签 -->
        <router-link to="/show/son1">子路由1</router-link>
        <router-link to="/show/son2">子路由2</router-link>
      </p>
      <!-- 路由出口 -->
      <!-- 路由匹配到的组件将渲染在这里 -->
      <router-view></router-view>
  </div>
</template>
```

在src/router/index.js中，导入需要的组件，并使用 children 属性声明子路由

```js
// 1. 定义 (路由) 组件。
// 可以从其他文件 import 进来
// const Foo = { template: '<div>foo</div>' }
import Test from '../components/Test.vue'
import Show from '../components/Show.vue'
import Hello from '../components/Hello.vue'

// *************************************
import Son1 from '../components/router-test/Son1.vue'
import Son2 from '../components/router-test/Son2.vue'
//***************************************

Vue.use(VueRouter)

// 2. 定义路由
// 每个路由应该映射一个组件。 其中"component" 可以是
// 通过 Vue.extend() 创建的组件构造器，
// 或者，只是一个组件配置对象。
// 我们晚点再讨论嵌套路由。
const routes = [
    // 首页重定向到 /hello 对应的组件
    { path: '/', redirect: '/hello' },
    { path: '/hello', component: Hello },
  // **************************************
    { 
        path: '/show',
        component: Show,
        children: [
          	// 注意此处子路由 path 两种写法
            { path: '/show/son1', component: Son1 },
            { path: 'son2', component: Son2 },
        ]
    },
  // **************************************
    { path: '/test', component: Test },
  ]
```

6. 动态路由

例如商品详情，不可能给每一个商品创造一个详情页面，创建一个所有商品都跳转到这个页面

```vue
<!--Test.vue-->
<template>
  <div>
    <h1>充数测试vue-router</h1>

    <p>
        <!-- 使用 router-link 组件来导航. -->
        <!-- 通过传入 `to` 属性指定链接. -->
        <!-- <router-link> 默认会被渲染成一个 `<a>` 标签 -->
        <router-link to="/test/1">商品1</router-link>
        <router-link to="/test/2">商品2</router-link>
        <router-link to="/test/3">商品3</router-link>
      </p>
      <!-- 路由出口 -->
      <!-- 路由匹配到的组件将渲染在这里 -->
      <router-view></router-view>
  </div>
</template>
```

index.js路由配置

```js
// //-------------导入---------
import Product from '../components/router-test/Product.vue'

Vue.use(VueRouter)

// 2. 定义路由
// 每个路由应该映射一个组件。 其中"component" 可以是
// 通过 Vue.extend() 创建的组件构造器，
// 或者，只是一个组件配置对象。
// 我们晚点再讨论嵌套路由。
const routes = [
    // 首页重定向到 /hello 对应的组件
    { path: '/', redirect: '/hello' },
    { path: '/hello', component: Hello },
    { 
        path: '/show',
        component: Show,
        children: [
            // 此处path不能以/开头
            { path: '/show/son1', component: Son1 },
            { path: '/show/son2', component: Son2 },
        ]
    },
  	//-------------下面---------
    { 
        path: '/test',
        component: Test,
        children: [
            { path: '/test/:id', component: Product },
        ] 
    },
  ]
```

Product路由页面

```vue
<template>
  <div>
    <h1>通用Product详情页</h1>

    <br/>

    <h3>
        <span>通过 $route.params.id 获取不同商品 id ===></span>
        {{ $route.params.id }}
    </h3>
  </div>
</template>
```

7. 编程式导航, 路由跳转的另一种方式

```vue
<template>
	<button @click="getProduct(2)">
    跳转到商品2
  </button>
</template>

<script>
	export default {
    methods: {
      getProduct(id) {
        this.$route.push('/test/${id}')
      }
    }
  }
</script>
```

8. 导航守卫

导航守卫可以控制路由的访问权限；

全局导航守卫会拦截每个路由规则，从而对每个路由进行访问权限的控制；

你可以使用 router.beforeEach 注册一个全局前置守卫：

```vue
router.beforeEach(to, from, next) => {
	if (to.path === '/main' && !isAuthenticated) {
		next('/login')
	} else {
		next()
	}
}
```

- to: 即将进入的目标地址
- from: 当前导航正要离开的路由
- 在守卫方法中如果声明了 next 形参，则必须调用 next() 函数，否则不允许用户访问任何一个路由
  - 直接方向： next()
  - 强制留在当前页面： next(false)
  - 强制跳转到登录页面： next('/login')

## Vuex 介绍

> 1. 对于组件化开发来说，多层嵌套父子组件传递数据已经十分麻烦，而Vue 更是没有为兄弟组件提供传递数据状态的办法；
> 2. 基于这个问题，Vuex 提供了全局状态管理器，将所有分散的数据交给状态管理器保管；
> 3. 简单来说，Vuex 用于管理分散在Vue 各个组件中的数据；
> 4. 安装sudo npm install vuex@3 --save
> 5. 官网： https://v3.vuex.vuejs.org/zh/

1. 状态管理

> 1. 每一个 Vuex 应用的核心都是一个 store, 与普通全局对象不同，基于 Vue数据与视图绑定的特点，当 store 中的状态发生变化时，与之绑定的视图也会被重新渲染
> 2. store 中的状态不允许被直接修改，改变 store 中状态的唯一途径就是显式的提交（COMMIT） MUTATION, 这可以让我们方便的跟踪每一个状态的变化
> 3. 在大型复杂应用中，如果无法有效的跟踪到状态的变化，将会对理解和维护代码带来很大的困难
> 4. Vuex中的5个重要概念 State/Getter/Mutation/Action/Module

![](https://yunqing-img.oss-cn-beijing.aliyuncs.com/hexo/article/202303/Pm5txO.png)

2. State 用于维护所有应用层的状态，确保应用只有唯一的数据源

```vue
// 创建一个新的 store 实例
const store = createStore({
	state() {
		return {
			count:0
		}
  },
	// 修改 count 需要调用 mutations方法
	mutations: {
		increment(state) {
			state.count++
		}
  }
}) 
```

在组件中，可以直接使用 this.$store.state.count 访问数据，也可以先用 mapState 辅助函数将其映射下来

```vue
import {mapState} from 'vuex'

export default {
	computed: mapState({
		// 箭头函数使代码更简练
		count: state => state.count,

		// 传字符串参数 ’count‘ 等同于 ’state => state.count‘
		countAlias: 'count',

		// 为了能使用 this 获取局部状态，必须使用常规函数
		countPlusLocalState(state) {
			return state.count + this.localCount
		}
	})
}
```

3. Mutations

 mutations.commit  会触发下面的方法

```vue
mutations: {
		increment(state) {
			state.count++
		}
  }
```

使用方法如下

```
methods: {
	increment() {
		this.$store.commit('increment')
		// 改变count之后
		console.log(this.$store.state.count)
	}
}
```

也可以使用辅助函数mapMutation 辅助函数

```vue
methods: {
	...mapMutations([
		// 将this.increment() 映射为 this.$store.commit('increment')
		'increment',
		// mapMutations 也支持载荷
		// 将 this.incrementBy(amount) 映射为 ’this.$store.commit('incrementBy', amount)‘
		’incrementBy‘
	])
}
```



4. Action

​	Action 类似于 Mutation 不同在于：

​	Action 不能直接修改状态，只能通过提交 mutation 来修改

​	Action 可以支持异步操作

```vue
// 创建一个新的 store 实例
const store = createStore({
	state() {
		return {
			count:0
		}
  },
	// 修改 count 需要调用 mutations方法
	mutations: {
		increment(state) {
			state.count++
		}
  },
	actions: {
		increment(context) {
			context.commit('increment')
		}
	}
}) 
```

如何触发action？ 在组件中直接 this.$store.dispatch('xxx') 分发 action 或者使用 mapActions 辅助函数

```vue
methods: {
	...mapActions([
		// 将this.increment() 映射为 this.$store.dispatch('increment')
		'increment',
		// mapMutations 也支持载荷
		// 将 this.incrementBy(amount) 映射为 ’this.$store.dispatch('incrementBy', amount)‘
		’incrementBy‘
	])
}
```



4. Getter

   维护由 State 派生的一些状态，这些状态随着 State 状态的变化而变化

```vue
const store = createStore({
	state: {
		todos: [
			{id:1, text:'1', done: true},
			{id:2, text:'2', done: false}
		]
	},
	getters: {
		doneTodos:(state) => {
			return state.todos.filter(todo => todo.done)
		}
	}
})
```

在组件中，可以直接使用 this.$store.getters.doneTodos.页可以用mapState 辅助函数将其映射下来

```vue
import {mapState} from 'vuex'

export default {
	computed: {
		// 使用对象展开运算符将 getter 混入 computed 对象中
		...mapGetters({
			'doneTodosCount',
			'anotherGetter',
		})
	}
}
```

### Vuex 使用

1. 创建 store/index.js, 创建并导出 store

```js
import Vue from 'vue';
// 导入vuex
import Vuex from 'vuex';
// 全局注册 vuex
Vue.use(Vuex)

const store = new Vuex.Store({
    state: {
        // state 里的数据交给各个组件使用
        count: 0,
    },

    mutations: {
        increment (state) {
            state.count++
        }
    },
})

// 导出store
export default store
```

2. 在main.js中导入store，并挂载

```js
// 导入 vuex 的store
import store from './store/index.js'

new Vue({
  render: h => h(App),
  // 挂载，已省略
  router,
  // 挂载,因为名称相同可以省略，未省略
  store: store
}).$mount('#app')
```

3. 在任意组件中使用

```vue
<h1>
  {{this.$store.state.count}}
</h1>
```

4. 如何修改

```vue
<script>
export default {
    methods: {
        add() {
            // 方案一：不推荐直接修改
            // this.$store.state.count = this.$store.state.count + 1
            // 方案二：更优雅, 交给 store 统一管理
            this.$store.commit('increment')
        },
    }
}
</script>
```

5. 映射用法：

```js
// store/index.js

import Vue from 'vue';
// 导入vuex
import Vuex from 'vuex';
// 全局注册 vuex
Vue.use(Vuex)

const store = new Vuex.Store({
    state: {
        // state 里的数据交给各个组件使用
        count: 0,
        todos: [
			{id:1, text:'吃饭', done: true},
			{id:2, text:'上班', done: false}
		]
    },
    mutations: {
        increment (state) {
            state.count++
        },
        incrementBy (state, n) {
            state.count += n
        },
    },
    // 对数据做一些过滤
    getters: {
		doneTodos:(state) => {
			return state.todos.filter(todo => todo.done)
		}
	},
})

// 导出store
export default store
```

```vue
// *.vue

<template>
	     <span>下面取值store.state.count值 = </span>
      <el-button type="primary" @click="add">count</el-button>
      <h1>{{ this.$store.state.count }}</h1>
      <h2>{{ count }}</h2>

      <ul>
        <li v-for="todo in doneTodos" :key="todo.id">{{ todo.text }}</li>
      </ul>
  </div>
</template>

<script>
import { mapState, mapGetters, mapMutations } from 'vuex'

export default {
    computed: {
        ...mapState([
            'count',
            'todos'
        ]),
        ...mapGetters([
            'doneTodos'
        ]),
    },
    methods: {
        add() {
            // 方案一：不推荐直接修改
            // this.$store.state.count = this.$store.state.count + 1
            // 方案二：更优雅, 交给 store 统一管理
            //this.$store.commit('increment')
            // 方案三：mapMutations 映射之后，荷载参数
            this.incrementBy(2)
        },
        // 注意此映射写在 methods 里， 而不是computed 里
        ...mapMutations([
            'increment', // 将 `this.increment()` 映射为 `this.$store.commit('increment')`

            // `mapMutations` 也支持载荷：
            'incrementBy' // 将 `this.incrementBy(amount)` 映射为 `this.$store.commit('incrementBy', amount)`
        ]),
    }
}
</script>

<style>

</style>
</template>
```

还要注意，一定在 main.js中引入挂载

```js
// main.js
// 导入 vuex 的store
import store from './store/index.js'

new Vue({
  render: h => h(App),
  // 挂载，已省略
  router,
  // 挂载,因为名称相同可以省略，未省略
  store: store
}).$mount('#app')
```

###  Action 使用和 Mutations 几乎一致

### Modules 是把各个模块数据分开,一个store包含多个模块数据

使用方式参考文档：

https://v3.vuex.vuejs.org/zh/guide/modules.html



## mockjs 介绍

> 1. 当后端代码没写好时，前端可以用 mockjs模拟接口响应数据
> 1. 官网： http://mockjs.com/
> 1. 支持随机生成文本、数字、布尔、日期、邮箱、连接、图片、颜色等
> 1. 安装 sudo npm i mockjs



1. 基本使用

在项目中创建mock目录，新建index.js

```js
// 引入mockjs
import Mock from 'mockjs'

const url = {
    tableDataOne: 'http://20181024Mock.com/product/search',
}

// 使用 mockjs 模拟数据
Mock.mock(url.tableDataOne, 'get', {
    "ret":0,
    "data": {
        "mtime": "@datetime", // 随机生成时间
        "score|1-800": 800, // 随机生成1-800的数字
        "rank|1-100":100,
        "starts|1-5":5,
        "nickname":"@cname",// 随机生成中文名字
        "img":"@image('200x100', '#ffcc33', '#FFF', 'png', 'Fast Mock')"
    }
})
```

在 main.js中引入

```js
// 导入mock, 测试环境使用
import './mock'
```

在 Vue 页面请求

```vue
<el-button type="primary" @click="mockRequest">mock</el-button>
      <h2><span>mock的数据是 => </span><span> {{ mockData }}</span></h2>

<script>
	methods: {
        mockRequest() {
            this.$http.get('http://20181024Mock.com/product/search').then(res => {
                console.log(res.data)
                this.mockData = res.data
            })
        },
          
  }
</script>
```

## Vue-element-admin 介绍

> 1. 企业级后端管理系统轮子
> 2. 官网： https://panjiachen.gitee.io/vue-element-admin-site/zh/
> 3. template 是基础模板，admin提供所有功能
> 4. 最好以template模板进行开发，以admin版本进行参考集成



