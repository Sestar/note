# js

* <span style="color:orange">js原生获取css文件中的style</span>

```javascript
printStyle: function (obj) {
	console.log(this.getCurrentStyle(obj.getElementsByClassName('ivu-poptip-popper')[0], 'display'))
},
getCurrentStyle:  function (obj, prop)
{
	if (obj.currentStyle) //IE
	{
		return obj.currentStyle[prop];
	}
	else if (window.getComputedStyle) //非IE
	{
		let propprop = prop.replace (/([A-Z])/g, "-$1");
		propprop = prop.toLowerCase ();
		return document.defaultView.getComputedStyle(obj,null)[propprop];
	}
	return null;
}
```
</br>


* <span style="color:orange">js原生增加删除class</span>

```javascript
// 判断元素是否有class
hasClass: function (ele, cls) {
	return ele.className.match(new RegExp('(\\s|^)'+cls+'(\\s|$)'));
},
// 添加class
addClass: function (elements,cName){
	if (!this.hasClass(elements,cName)){
		elements.className += " " + cName;
	};
},
// 移出class
removeClass: function (ele, cls) {
	if (this.hasClass(ele, cls)) {
		var reg = new RegExp('(\\s|^)'+cls+'(\\s|$)');
		ele.className = ele.className.replace(reg, ' ');
	}
}
```