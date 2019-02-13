# angularJs

## 基本标签

|    指令    |       作用      |
|:-------:|:------------- |
| ng-init | 初始化数据 |
| ng-app | 指定根元素 |
| ng-model | 数据绑定 |
| ng-repeat | 有循环圆点的遍历数据(ul的圆点) |
| ng-disabled | 指令绑定应用程序数据 "mySwitch" 到 HTML 的 disabled 属性 |
| ng-model | 指令绑定 "mySwitch" 到 HTML input checkbox 元素的内容（value）|
| ng-show | 指令根据 value 的值来显示（隐藏）HTML 元素 |
| ng-click | 在点击按钮时调用 |
</br>

* <span style="color:orange">自定义指令</span>

```javascript
app.directive("自定义指令", function(){
	return {
		template : "<h1>这是指令demo！<h1>"
	};
});
```

```text
指令可以放在 
元素名<自定义指令>, 
属性 <div 自定义指令></div>, 
类名 <div class="自定义指令"></div>(function的return添加restrict : "C"),
注释 <!-- directive: 自定义指令 -->(function的return添加restrict : "M"和replace : true)

restrict值可以是以下几种:
E 作为元素名使用
A 作为属性使用
C 作为类名使用
M 作为注释使用
restrict 默认值为 EA, 即可以通过元素名和属性名来调用指令。
```
</br>


* <span style="color:orange">ng-show</span>

```text
<span ng-show="position.$error.email">验证这不是一个合法的邮箱地址</span>

{{position.$valid}}输入是否合法, 比如type:email
{{position.$dirty}}值是否修改
{{position.$touched}}通过屏幕点击
```
</br>

* <span style="color:orange">required标识该元素value必需, 配合 .ng-invalid的class可以显示value为空的有required修饰的元素设定css</span>
|    指令    |       作用      |
|:-------:|:------------- |
 | ng-valid | 验证通过 |
| ng-invalid | 验证失败 |
| ng-valid-[key] | 由$setValidity添加的所有验证通过的值 |
| ng-invalid-[key] | 由$setValidity添加的所有验证失败的值 |
| ng-pristine | 控件为初始状态 |
| ng-dirty | 表单有填写记录 |
| ng-touched | 控件已失去焦点 |
| ng-untouched | 控件未失去焦点 |
| ng-pending | 任何为满足$asyncValidators的情况 |
</br>


* <span style="color:orange">$scope.names</span>
```text
$scope.names = ["a", "b", "c"];
$scope.names.splice(index, 1); 删除index下标的元素
$scope.names.push("d"); 添加元素
```
</br>


* <span style="color:orange">指令</span>
|    指令    |       作用      |
|:-------|:------------- |
| \|currency |  转化字符串为金额格式(结果: $25.00) |currency:"￥" (结果: ￥25.00) |
| \|date | “yyyy-MM-dd HH:mm:ss”|
| \|order | 'county' 排序 (后面可以加上 :false为升序, :true为降序或者 '-county') |
| \|number:2 |  保留小数 |
| \| filter : {"name":"zhangs"}}} | 查询name为zhangs |
| \|limitTo |  整数从前截取, 负数从后截取 |
</br>


* <span style="color:orange">自定义过滤器filter</span>

```javascript
app.filter('reverse', function() { //可以注入依赖
    return function(text) {
        return text.split("").reverse().join("");
    }
});
```
</br>


* <span style="color:orange">自定义服务</span>

```javascript
app.service('hexafy', function() {
	this.myFunc = function (x) {
        return x.toString(16);
    }
});
app.controller('myCtrl', function($scope, hexafy) {
  $scope.hex = hexafy.myFunc(255);
});
```
</br>


```text
 $location.absUrl()输出当前url
 $apply修改数据
 $watch监听数据变化
 ng-if="\$odd" ($even) 列表奇偶行
```
</br>


* <span style="color:orange">列表项初始化</span>

```html
<select ng-init="selectedName = names[0]" ng-model="selectedName" ng-options="x for x in names"></select>
<select><option ng-repeat="x in names">{{x}}</option></select>
```
</br>


* <span style="color:orange">switch-case</span>

```html
<div ng-switch="myVar">
  <div ng-switch-when="dogs">
     <h1>Dogs</h1>
     <p>Welcome to a world of dogs.</p>
  </div>
  <div ng-switch-when="tuts">
     <h1>Tutorials</h1>
     <p>Learn from examples.</p>
  </div>
  <div ng-switch-when="cars">
     <h1>Cars</h1>
     <p>Read about cars.</p>
  </div>
</div>
```

