# IE<8循环引用导致的内存泄露


![leakage](./images/leakage.png)

在IE<8版本中，JScript垃圾回收器仅管理JScript对象生命周期而不会管理DOM对象的(即DOM对象有自己的垃圾回收器)。因此JScript回收器不会解除掉DOM对象与Jscript对象之间的相互引用，这从而导致内存泄露。
在IE6中，循环引用只在IE浏览器程序退出时才会被解除，而在IE7中，离开当前页面时，才会解除页面中的循环引用。IE8修复该问题，JScript垃圾回收器会将引用的DOM对象视为JScript对象，从而避免循环引用不能被解除的问题（_注：这里循环引用解除是指浏览器自动解除循环引用_）。

IE6-7中管理DOM对象的垃圾回收器是基于引用计数策略，如果DOM对象与js对象存在循环引用。需要将DOM对象上的expando属性设置为null或重新赋值。这样才能回收DOM对象。(_注：这里只是针对DOM的回收_)
```html
<html>
    <head>
        <script language="JScript">
        var myGlobalObject;
        function SetupLeak()
        {
            myGlobalObject = document.getElementById("LeakedDiv");// ← js对象引用DOM对象
            document.getElementById("LeakedDiv").expandoProperty = myGlobalObject;// ← DOM对象的expando属性引用js对象
        }
        function BreakLeak()
        {
            document.getElementById("LeakedDiv").expandoProperty = null;// ← 设置为null或重新赋新值
        }
        </script>
    </head>

    <body onload="SetupLeak()" onunload="BreakLeak()">
        <div id="LeakedDiv"></div>
    </body>
</html>
```
上述代码的循环引用不是很复杂，很容易发现。不过实际项目中可能会存在循环链很长的情况：
```js
    (function(){
        var d={b:document.body}
        var obj={doc:d}; // ← obj.doc.b === document.body
        document.body.o=obj; // ← 循环引用: document.body.o.doc.b === document.body
    })();
```
另外一种常见的循环引用发生在闭包中：
```js
(function(){
    var b=document.body; // ← 创建一个引用document.body的变量"b"
    b.onclick=function() { // ← b.onclick引用function
    // 通过闭包能在函数中能访问到"b"
    // do something...
    };
})();
```
## 为什么DOM的垃圾回收器是基于引用计数策略？
IE中有一部分对象并不是原生js对象。例如，其内存泄露DOM和BOM中的对象就是使用C++以COM(component object model，组件对象模型)对象的形式实现的，而COM对象的垃圾回收机制采用的就是引用计数策略。因此，即使IE的js引擎采用标记清除策略来实现，但js访问的COM对象依然是基于引用计数策略的。换句话说，只要在IE中涉及COM对象，就会存在循环引用的问题。在IE9把BOM和DOM对象转换为真正的js对象。

除了低版本的IE外，在低版本的Firefox(如Firefox3.0)中也存在类似问题。Firefox通过自己的XPCOM来管理DOM，与Windows的COM类似，Mozilla的XPCOM也是基于引用计数策略。
 
## 如何避免及修复？
循环引用是导致低版本IE和Firefox浏览器内存泄露的真正原因，因此最直接的方法是避免在DOM和JS之间创建相互引用。确保总是JS对象单向引用DOM对象，或DOM对象单向引用JS对象。虽然说起来简单，但实际情况是很难做到的。那如何修复循环引用就很重要了，可以维护一个存在循环引用DOM对象的队列，在页面unload时，做如下处理：
```js
(function(){
    var unLoaders=[];
    myDomNode.object=new myObject(); // ← 假设这里创建一个循环引用，会引起内存泄露
    unLoaders.push(myDomNode); // ← 缓存myDomNode

    var unload=function(){ // ← unload回调函数
    for(var i=unLoaders.length-1;i>-1;i–){
        unLoaders[i].object=null; // ← 切断循环引用
    }
    };
    window.addEvnetListener(’unload’, unload); // ← 绑定unload
})();
```
jquery在处理IE低版本类似问题时，采用的也是上述方法，[jquery源码](https://github.com/jquery/jquery/blob/1.4.4rc1/src/event.js#L1169)：
```js
// Prevent memory leaks in IE
// Window isn't included so as not to unbind existing unload events
// More info:
//  - http://isaacschlueter.com/2006/10/msie-memory-leaks/
if ( window.attachEvent && !window.addEventListener ) {
	jQuery(window).bind("unload", function() {
		for ( var id in jQuery.cache ) {
			if ( jQuery.cache[ id ].handle ) {
				// Try/Catch is to handle iframes being unloaded, see #4280
				try {
					jQuery.event.remove( jQuery.cache[ id ].handle.elem );
				} catch(e) {}
			}
		}
	});
}
```



>This memory leak occurs because DOM objects are non-JScript objects. DOM objects are not in the mark-and-sweep garbage collection scheme of JScript. Therefore, the circular reference between the DOM objects and the JScript handlers will not be broken until the browser completely tears down the page. This memory leak will end when the browser opens a new Web page or when the browser window is closed.


## 总结
随着移动互联网的飞速发展，IE也向W3C标准靠拢，低版本浏览器正逐渐被淘汰。不过实际业务中，还是存在对某些低版本进行支持，虽然概率很少，不过为了保证用户体验，需要对本文涉及的知识点有所了解。也许你的业务中，不需要对低版本浏览器支持。那就把本文当成浏览器发展史来了解吧，也希望你能从文章有所收获。


## 参考文章：
- [Circular Memory Leak Mitigation](https://msdn.microsoft.com/en-us/library/dd361842(v=vs.85).aspx)
- [DOM: why is this a memory leak?](https://stackoverflow.com/questions/15761094/dom-why-is-this-a-memory-leak)
- [Internet Explorer Event Handler Leaks](http://www.reigndropsfall.net/2011/01/05/internet-explorer-event-handler-leaks/)
- [Avoiding leaks in JavaScript XPCOM components](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Guide/Avoiding_leaks_in_JavaScript_components)
- [JScript Memory Leaks](http://www.crockford.com/javascript/memory/leak.html)
- [Memory Leaks in Microsoft Internet Explorer](http://isaacschlueter.com/2006/10/msie-memory-leaks/trackback/index.html)