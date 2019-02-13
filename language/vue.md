# vue

* <span style="color:orange">侦听对象的某个值</span>
  * <span style="color:orange">用computed先监听该值</span>

```javascript
computed: {
    aaa() {
        return obj.value
    }
}
```

   * * <span style="color:orange">再用watch侦听aaa</span>

```javascript
watch: {
    aaa (newVaule, olderValue) {

    }
}
```
</br>

* <span style="color:orange">watch使用loadsh的_.debounce延时函数不会有延时效果, 需要直接写在method上</span>

```javascript
method: {
  methodName: _.debounce(function () {}, 1000)
}
```
</br>

* <span style="color:orange">JavaScript的真假值</span>
```text
在JavaScript中，Truthy(真值)指的是在Boolean上下文中转换后的值为真的值。
所有值都是真值，除非它们被定义为 falsy (即， 除了false，0，“”，null，undefined和NaN 外)。
```
</br>

* <span style="color:orange">子组件直接调用父组件的方法</span>

```javascript
v-on:change="$emit('change', $event.target.checked)"
```
</br>

* <span style="color:orange">子组件双向绑定父组件属性(修改子组件的value, 父组件内容也对应改变)</span>
    * <span style="color:orange">在父组件定义子组件</span>
```javascript
Vue.component('custom-input', {
  props: ['value'],
  template: `
    <input  v-bind:value="value"  v-on:input="$emit('input', $event.target.value)">
  `
})
```
* * <span style="color:orange">现在 v-model 就应该可以在这个组件上完美地工作起来了</span>
```javascript
<custom-input v-model="searchText"></custom-input>
```
</br>

* <span style="color:orange">自定义组件, 继承原生事件</span>
```javascript
<template>
  <base-input></base-input>
</template>
<script>
Vue.component('base-input', {
  inheritAttrs: false,  // 	取消非props的默认行为, 比如input
  props: ['label', 'value'],
  computed: {
    inputListeners: function () {
      var vm = this
      // `Object.assign` 将所有的对象合并为一个新对象
      return Object.assign({},
        // 我们从父级添加所有的监听器
        this.$listeners,
        // 然后我们添加自定义监听器,
        // 或覆写一些监听器的行为
        {
          // 这里确保组件配合 `v-model` 的工作
          input: function (event) {
            vm.$emit('input', event.target.value)
          }
        }
      )
    }
  },
  template: `
    <label>
      {{ label }}
      <input
        v-bind="$attrs"   // 继承所有原生属性
        v-bind:value="value"
        v-on="inputListeners"
      >
    </label>
  `
})
</script>
```
</br>


* <span style="color:orange">事件冒泡</span>
```html
<div @click='doThis' style="width:100px;height: 100px; background: red;">
    点击父元素
  <a v-on:click.stop="doThis">点击子元素</a>
</div>
```
```text
当点击父元素的时候，执行doThis,当点击子元素a的时候，这个点击动作不单单触发了a标签，
同时也触发了div标签，这就是事件冒泡，所以假设上述例子中a标签为v-on:click='doThis',则doThis会被执行两次，
父元素和子元素都执行了一次click事件，而.stop则是阻止事件冒泡，再次点击a标签，click事件只会执行一次*/
.prevent阻止默认事件触发, .self只有点击自身才能触发,  .capture先执行外部事件, 再执行内部
```
</br>


* <span style="color:orange">监听键盘事件</span>
```javascript
<input v-on:keyup.13="submit"/> 13对应按键数字
```
* <span style="color:orange">全部的按键别名</span>
```text
.enter  .tab  .delete (捕获 “删除” 和 “退格” 键)  .esc  .space  .up  .down  .left  .right .ctrl .alt .shift .meta(windows: 田)
```
* <span style="color:orange">可以通过全局 config.keyCodes 对象自定义按键修饰符别名</span>
```text
  Vue.config.keyCodes.f1 = 112
    Alt + C ：<input @keyup.alt.67="clear">
  @keyup.ctrl="", 单单释放ctrl没有用, 只有按住ctrl, 释放其他按键, 才会触发; 应该写成@keyup.17="", 或者@keyup.ctrl
```
</br>


* <span style="color:orange">Vue命名</span>
```text
字符串模板: template:"<myComponent></myComponent>";  
非字符串模板: <template></template> 只能使用kebab-case(短横线分割命名)
```

* * <span style="color:orange">camelCase (驼峰命名法)的 prop 名需要使用其等价的 kebab-case (短横线分隔命名) 命名(HTML对大小写不敏感)</span>
```javascript
Vue.component('blog-post', {
  // 在 JavaScript 中是 camelCase 的
  props: ['postTitle'],
  template: '<h3>{{ postTitle }}</h3>'
})
<!-- 在 HTML 中是 kebab-case 的 -->
<blog-post post-title="hello!"></blog-post>
```
```text
事件名需要完全v-on:myEvent 将会变成 v-on:myevent——导致 myEvent 不可能被监听到。所以始终使用 kebab-case 的事件名。
data的属性名不能为kebab-case, template中标签调用data数据也不能用kebab-case
```
</br>


* <span style="color:orange">加载本地json文件</span>
* * <span style="color:orange">注册路由(在dev-server.js, data.json放在最外部, src同级)</span>
```javascript
var data = require('../data.json')
var router = express.Router()
router.get("/data", function (req, res) {
  res.json(data)
})
```
* * <span style="color:orange">使用axios异步获取(data.json放在src同级下的static包中)</span>
```javascript
this.$axios.get('/static/data.json').then(res => {
	console.log(res.data)
})
```
</br>

* <span style="color:orange">v-model</span>
> 一个组件上的 v-model 默认会利用名为 value 的 prop 和名为 input 的事件
> model对象包含两个属性。prop默认是value，event默认是input
```javascript
<my-component v-model="obj"></my-component>
Vue.component('my-component', {
	props: {
		obj: Object,
	},
	model: {
		prop: 'obj',
		event: 'change'
	},
	methods: {
		onchange: function() {
			this.$emit('change', this.obj);
		}
	}
});
app.use(router)
```

> 用v-model双向绑定数据,源码是使用compositionstart, input, compositionend三个事件完成对value值改变
> 用v-bind:value绑定事件, 如果input事件触发自定义method的话, 在输入中文情况下, 会出现异常。
> 比如 this.name = $event.target.value.toUpperCase() (首字母大写), 输入woshishui, 会出现  WWOWO'SWO'我是谁
> 所以要添加两个事件, compsitionstart, compositionend对其加锁。 或者format对change触发
```javascript
<input :value="name" @input="format" @compositionstart="start" @compositionend="end">
var app = new Vue({
  el: '#app',
  data: {
    name: '',
    composing: false
  },
  methods: {
    format ($event) {
      if (this.composing) {
        return
      }
      this.name = $event.target.value.toUpperCase()
    },
    start () {
      this.composing = true
    },
    end ($event) {
      this.composing = false
      this.name = $event.target.value.toUpperCase()
    }
  }
})
```
或者
```javascript
<input :value="name" @input="name = $event.target.value" @change="format">
```
</br>

* <span style="color:orange">native和syna</span>
```text
.native通常给自定义组件添加原生事件
.syna组件间数据绑定
```
</br>

* <span style="color:orange">keep-alive</span>
> keep-alive保存组件的状态, keep-alive标签的第一个子标签就是需要缓存的组件
```javascript
<keep-alive>
	<component :is="activeCmpCurr"></component>
</keep-alive>
```
</br>

* <span style="color:orange">非父子组件间传递数据</span>
```javascript
定义一个中转站, bus.js
发送端引入bus.js, 动作中使用this.$emit('自定义事件', data)
接收端也引入bus.js, 动作中使用this.$on('相同事件', data => {})
```
</br>

* <span style="color:orange">访问根组件的属性</span>
> this.$root.属性
</br>

* <span style="color:orange">访问高德地图</span>
> 在webpack.base.conf.js中引入AMap, externals{'AMap': 'AMap'}
> HTML中引入<script src="https://webapi.amap.com/maps?v=1.4.8"></script>如果访问公用地图(不是自定义地图),不需要key
> 在vue中import AMap from 'AMap'
> 组件中初始化地图:
```javascript
// 地图组件使用
<amap-comp>
  <amap-mark-comp></amap-mark-comp>
</amap-comp>

mounted () {
  // 初始化高德地图
  this.initMap()
},
methods: {
  // 初始化地图
  initMap: function () {
    this.map = new AMap.Map('syncMap', {
  	center: [116.397428, 39.90923],
	resizeEnable: true,
	zoom: 10
	})
  }
}
```
> amap-comp是地图组件, amap-mark-comp只是一个标记点组件, 必不需要任何渲染(template), 可以使用render(h) {return null}
</br>


* <span style="color:orange">处理父组件的属性判断后, 再交给子组件渲染</span>
```javascript
vue:
  data () {return {places: {'name': '北京', position: [119.300669, 26.098182]}}}
  template: 
    `<fu>
	   <zi :places='places'></zi>
	</fu>`
fu: {
  data () {return {map: null}}
  methods: {
    // found用来接收子组件的function(子组件的匿名函数)
	getMap: function(found) { 
	  var vm = this
	  // 判断map的初始化是否成功, 成功就让子组件渲染, 不成功就过50毫秒再判断一次
	  function checkForMap() {
		if (vm.map) {
		  found(vm.map)
		} else {
		  setTimeout(checkForMap, 50)
		}
	  }
	  checkForMap()
	}
  }
}
zi: {
  props: ['places'],
  created () {
	var vm = this
	// 调用父组件的getMap
	vm.$parent.getMap(function(map) {
	  // 遍历places
	  vm.places.forEach(function(place)) {
		var marker = new AMap.Marker({
		  position: place.position,
		  title: place.name,
		  icon: '//vdata.amap.com/icons/b18/1/2.png'
		)}
		map.add(marker)
	  }
	})
  },
  render(h): {return null}
}
```
> 还有一种是使用依赖注入的方式, 父组件provide注册, 子组件inject注入, 替代了$parent更好得保护父组件的内容
```javascript
fu: {
  provide: () {return { getMap: this.getMap}},
  data () {return {map: null}}
  methods: {
    // found用来接收子组件的function(子组件的匿名函数)
		getMap: function(found) { 
			var vm = this
			// 判断map的初始化是否成功, 成功就让子组件渲染, 不成功就过50毫秒再判断一次
			function checkForMap() {
			if (vm.map) {
				found(vm.map)
			} else {
				setTimeout(checkForMap, 50)
			}
			}
			checkForMap()
		}
  }
}
zi: {
  inject: ['getMap'],
  props: ['places'],
  created () {
	var vm = this
	// 调用父组件的getMap
	vm.getMap(function(map) {
	  // 遍历places
	  vm.places.forEach(function(place)) {
		var marker = new AMap.Marker({
		  position: place.position,
		  title: place.name,
		  icon: '//vdata.amap.com/icons/b18/1/2.png'
		)}
		map.add(marker)
	  }
	})
  },
  render(h): {return null}
}
```
</br>


* <span style="color:orange">组件间的循环引用, 不用懒加载的方式, 会出现渲染错误, 在beforeCreate内加载组件</span>
* * <span style="color:orange">index.vue</span>

```javascript
<tree-folder :folders="folders"></tree-folder>

<script>
    import TreeFolder from './component/treeFolder'
</script>
```

* * <span style="color:orange">TreeFolder.vue</span>

```javascript
<template>
  <div>
    <div v-for="folder of folders" :key="folder.name">
      <span>{{folder.name}}</span>
      <tree-folder-content :folders="folder"></tree-folder-content>
    </div>
  </div>
</template>

<script>
export default {
  name: 'treeFolder',
  props: ['folders'],
  beforeCreate () {
    this.$options.components.TreeFolderContent = () => import('./treeFolderContent')
  }
}
</script>
```

* * <span style="color:orange">TreeFolderContent.vue</span>

```javascript
<template>
  <div>
    <div v-if="folders.children">
      <tree-folder :folders="folders.children"></tree-folder>
    </div>
  </div>
</template>

<script>
import TreeFolder from './TreeFolder'
export default {
  name: 'treeFolderContent',
  props: ['folders'],
  components: {
    TreeFolder
  }
}
</script>
```
</br>

* <span style="color:orange">$router, $route区别</span>
> $route为当前router跳转对象里面可以获取name、path、query、params等
> \$router为VueRouter实例，想要导航到不同URL，则使用\$router.push方法返回上一个history也是使用$router.go方法

* * <span style="color:orange">发送端</span>
```javascript
this.$router.push({
  name: 'ComponentDepth',
  params: {
	registCmp: {postTitle: '数据'}
  }
})
```
* * <span style="color:orange">接收端</span>
```javascript
this.$route.params.registCmp.postTitle
```
</br>

* <span style="color:orange">三种动画效果</span>
* * <span style="color:orange">transition过渡效果函数</span>
> transition: property(变化属性) duration(持续时间) timing-function(速率变化函数) delay(延迟执行时间);

```css
{
	<transition name="rightMove">
		<p v-if="rightMove" class="show-data">Hello</p>
	</transition>
	/*右移消失*/
	.right-move-enter-active {
		/*加速进入*/
		transition: .3s ease-in;
	}
	.right-move-leave-active {
		/*自定义曲线函数*/
		transition: .8s cubic-bezier(1.0, 0.5, 0.8, 1.0);
	}
	.right-move-enter, .right-move-leave-to {
		transform: translateX(10px);
		opacity: 0;
	}
}
```

* * <span style="color:orange">animation</span>
> animation: property(变化属性) duration(持续时间) timing-function(速率变化函数) delay(延迟执行时间) iteration-count(动画播放次数) direction(反向播放)

```css
{
	<transition name="enlarge">
		<p v-if="enlarge" class="show-data">Hello</p>
	</transition>
	/*放大过渡*/
	.enlarge-enter-active {
		animation: enlarge-in .5s;
	}
	.enlarge-leave-active {
		animation: enlarge-in .8s reverse;
	}
	@keyframes enlarge-in {
		0% {
			transform: scale(0.5);
		}
		50% {
			transform: scale(2);
		}
		100% {
			transform: scale(1);
		}
	}
}
```
> animate: 官网:https://daneden.github.io/animate.css/
> enter-active-class="animated tada" 将标签进入效果使用animated的tada效果
```javascript
{
	<div class="col-md-1 col-md-push-2">
		<transition enter-active-class="animated tada" leave-active-class="animated bounceOutRight">
			<p v-if="animate" class="show-data">Hello</p>
		</transition>
	</div>
	@import 'https://cdn.jsdelivr.net/npm/animate.css@3.5.1';
}
```

* * <span style="color:orange">自定义过渡效果</span>
> 安装Velocity插件, 使用自定义过渡, 最好在标签加上 :css="false" 不让css影响效果

```javascript
{
	<transition @before-enter="beforeEnter" @enter="enter" @leave="leave" :css="false">
		<p v-if="demo" class="show-data">Hello</p>
	</transition>
	
	methods: {
    // 自定义过滤进入之前效果(以左边为中心)
    beforeEnter (el) {
      el.style.opacity = 0
      el.style.transformOrigin = 'left'
    },
    // 自定义过滤进入效果(放大进入)
    enter (el, done) {
      Velocity(el, { opacity: 1, fontSize: '2em' }, { duration: 1000 })
      Velocity(el, { fontSize: '1.6em' }, { complete: done })
    },
    // 自定义过滤离开效果(以左边为中心, 顺时针旋转50度, 再50~100度之间旋转两个循环, 最后向下掉落)
    leave (el, done) {
      el.style.transformOrigin = 'left'
      Velocity(el, { translateX: '20px', rotateZ: '50deg' }, { duration: 600 })
      Velocity(el, { rotateZ: '100deg' }, { loop: 2 })
      Velocity(el, { translateX: '40px', translateY: '30px', opacity: 0 }, { complete: done })
    }
  }
}
```

* * <span style="color:orange">v-if, v-show</span>
```text
当有相同标签名的元素切换时(v-if, v-show)，需要通过 key 特性设置唯一的值来标记以让 Vue 区分它们，否则 Vue 为了效率只会替换相同标签内部的内容。
当值变了, Vue只会改变内容, 并不会把整个内容修改, 就不会有过渡效果, 可以使用key来让Vue区分.组件就不需要了. :is="componentCurr"切换组件
可以使用下面的标签来区分标签进入模式, out-in是让原标签消失后再进入, 相反有in-out, 默认是同时进行, 同时进入和消失
```
```html
<transition name="enlarge" mode="out-in"></transition>
```


* * <span style="color:orange">直接用data指定组件, 方便组件的变化</span>

```html
<component :is='dataName'></component>
data () { 
    dataName: 'componentName'
},
component () { 
    componentName: 'aaa'
}
```