#### 整体代码结构

我们拉下来代码后进入到`ajax.js`这个文件之后，一看哇，头皮发嘛。有点多，这个怎么操作？这样操作：

折叠代码！！！！

于是乎就成了这样：
```js
var Zepto = (function(){
    //...
    
    $.zepto = zepto

    return $
})()

window.Zepto = Zepto
window.$ === undefined && (window.$ = Zepto)
```
上面这段代码意思很明确，就是一个自执行函数返回值是`$`,这个函数上挂载`zepto`对象。这个zepto对象包含了很多内部的API

最后再将`Zepto`对象挂载载`window`对象上了。

基本架子算是看到了，所以非常重要的变量有`$,zepto`这两个。

那么我们可以接着把骨架在剔出来下。

```js
var zepto,$
//...
function Z(dom, selector) {
    var i, len = dom ? dom.length : 0
    for (i = 0; i < len; i++) this[i] = dom[i]
    this.length = len
    this.selector = selector || ''
}
zepto.Z = function(dom, selector) {
    return new Z(dom, selector)
}
//...
zepto.init = function(){
    //...
}
$ = function(selector, context){
    return zepto.init(selector, context)
}
//...
$.fn = {
    //...
}
//...
zepto.Z.prototype = Z.prototype = $.fn

```
主要的结构就是这样的，内部的API之类的东西就基本是挂在这些上的。特别是$和`$.fn`上挂载了特别多的属性和方法

我们平时在使用zepto的时候，就是用`$`去获得`dom`的，获得`dom`上也挂载各种操作方法。上面的代码中可以看到,`$`其实调用了`zepto.init()`方法，在这个方法中基本就是我们平时各种获取`dom`的方式的判断。在初始化获取dom的函数中会调用`zepto.Z`函数，这个函数只做了返回实例`Z`。`Z`中做的就是将获取的dom集合展开，然后再添加`length`和`selector`属性。

#### 主要代码的细节

上面是一个大概，然后看看主要的细节代码了
>1、zepto.init()函数

前面已经说了，这个函数主要是各种获取dom的相关操作，看看代码：

```js
zepto.init = function(selector, context) {
    var dom
    // 如果没有选择器，返回为空
    if (!selector) return zepto.Z()
    // selector是字符串
    else if (typeof selector == 'string') {
      selector = selector.trim()
      // 如果是html片段，那么就创建相关节点
      // Note: In both Chrome 21 and Firefox 15, DOM error 12
      // is thrown if the fragment doesn't begin with <
      if (selector[0] == '<' && fragmentRE.test(selector))
        dom = zepto.fragment(selector, RegExp.$1, context), selector = null
      // 如果context字断存在的话。就会在这个上下文中创建一个集合，然后从这个集合中选取节点，相当于 $(context).find(selector) 
      else if (context !== undefined) return $(context).find(selector)
      // 如果是css选择器的话，直接选择节点
      else dom = zepto.qsa(document, selector)
    }
    // 如果是个方法的话，那么就是等dom加载完毕调用它
    else if (isFunction(selector)) return $(document).ready(selector)
    // 如果本身就是一个zepto集合的话直接返回。
    else if (zepto.isZ(selector)) return selector
    else {
      // normalize array if an array of nodes is given
      if (isArray(selector)) dom = compact(selector)
      // 包装dom节点
      else if (isObject(selector))
        dom = [selector], selector = null
      // 如果是html片段,创建节点
      else if (fragmentRE.test(selector))
        dom = zepto.fragment(selector.trim(), RegExp.$1, context), selector = null
      else if (context !== undefined) return $(context).find(selector)
      else dom = zepto.qsa(document, selector)
    }
    return zepto.Z(dom, selector)
  }
```
>2、zepto.Z.prototype = Z.prototype = $.fn的意义

读到这里，你可能会有点疑惑，$ 最终返回的是 Z 函数的实例，但是 Z 函数明明没有 dom 的操作方法啊，这些操作方法都定义在 $.fn 身上，为什么 $ 可以调用这些方法呢？

其实关键在于这句代码 Z.prototype = $.fn ，这句代码将 Z 的 prototype 指向 $.fn ，这样，Z 的实例就继承了 $.fn 的方法。

既然这样就已经让 Z 的实例继承了 $.fn 的方法，那 zepto.Z.prototype = $.fn 又是为什么呢？

如果我们再看源码，会发现有这样的一个方法：

```js
zepto.isZ = function(object) {
  return object instanceof zepto.Z
}
```
这个方法是用来判读一个对象是否为 zepto 对象，这是通过判断这个对象是否为 zepto.Z 的实例来完成的，因此需要将 zepto.Z 和 Z 的 prototype 指向同一个对象。 isZ 方法会在 init 中用到，后面也会介绍。

#### 参考链接
[zepto源码中关于`zepto.Z.prototype = $.fn`的问题](https://segmentfault.com/q/1010000005782663)