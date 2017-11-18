>上期中我们看了元素选择器、attr和class的源码，今天我们来看下其append等的操作是如何进行的。

```javascript
clone: function(){
  return this.map(function(){ return this.cloneNode(true) })
}
```

```this.cloneNode(flag)``` 其返回this的节点的一个副本flag为true表示深度克隆，为了兼容性，该flag最好填写。

```javascript
children: function(selector){
  return filtered(this.map(function(){ return children(this) }), selector)
}
function filtered(nodes, selector) {
//当selector不为空的时候。
  return selector == null ? $(nodes) : $(nodes).filter(selector)
}
function children(element) {
//为了兼容性
  return 'children' in element ?
//返回 一个Node的子elements，在IE8中会出现包含注释节点的情况，
//但在此处并不会调用该方法。
    slice.call(element.children) :
//不支持children的情况下
    $.map(element.childNodes, function(node){ if (node.nodeType == 1) return node })
}
filter: function(selector){
  if (isFunction(selector)) return this.not(this.not(selector))
  return $(filter.call(this, function(element){
    return zepto.matches(element, selector)
  }))
}
//这也是一个重点，下面将重点分拆这个
zepto.matches = function(element, selector) {
  if (!selector || !element || element.nodeType !== 1) return false
  var matchesSelector = element.matches || element.webkitMatchesSelector ||
                        element.mozMatchesSelector || element.oMatchesSelector ||
                        element.matchesSelector
  if (matchesSelector) return matchesSelector.call(element, selector)
  var match, parent = element.parentNode, temp = !parent
  if (temp) (parent = tempParent).appendChild(element)
  match = ~zepto.qsa(parent, selector).indexOf(element)
  temp && tempParent.removeChild(element)
  return match
}
```

```children方法中的'children' in element是为了检测浏览器是否支持children方法，children兼容到IE9```。
```Element.matches(selectorString) ```如果元素被指定的选择器字符串选择，否则该返回true，selectorString为css选择器字符串。该方法的兼容性不错[element.matches的兼容性一览](https://caniuse.com/#search=matches)，在移动端中完全可以放心使用，当然在一些老版本上就不可以避免的要添加上一些前缀。如``` element.matches || element.webkitMatchesSelector ||
element.mozMatchesSelector || element.oMatchesSelector ||element.matchesSelector```所示。

注：其实在IE8中也可以通过polyfill的形式去实现该方法，如下所示：
```javascript
if (!Element.prototype.matches) {
    Element.prototype.matches =
        Element.prototype.matchesSelector ||
        Element.prototype.mozMatchesSelector ||
        Element.prototype.msMatchesSelector ||
        Element.prototype.oMatchesSelector ||
        Element.prototype.webkitMatchesSelector ||
        function(s) {
            var matches = (this.document || this.ownerDocument).querySelectorAll(s),
                i = matches.length;
            while (--i >= 0 && matches.item(i) !== this) {}
            return i > -1;
        };
}
```
children方法就如上面所示，其实其内部只是调用了几个不同的函数而已。

### closest
```javascript
closest: function(selector, context){
  var nodes = [], collection = typeof selector == 'object' && $(selector)
  this.each(function(_, node){
//当node存在同时（collection中拥有node元素或者node中匹配到了selector)
    while (node && !(collection ? collection.indexOf(node) >= 0 : zepto.matches(node, selector)))
//如果给定了context，则node不能等于该context
//注意只要node===context或者isDocument为true，那么node则为false
      node = node !== context && !isDocument(node) && node.parentNode
    if (node && nodes.indexOf(node) < 0) nodes.push(node)
  })
  return $(nodes)
}
//检测其是否为document节点
//DOCUMENT_NODE的更多用法可以前往https://developer.mozilla.org/zh-CN/docs/Web/API/Node/nodeType
function isDocument(obj)
  { return obj != null && obj.nodeType == obj.DOCUMENT_NODE }
```
### after prepend before append
这几种方法都是先通过调用zepto.fragment该方法统一将content的内容生成dom节点，再进行插入动作。同时需要注意的是传入的内容可以为html字符串，dom节点或者节点组成的数组，如下所示：'<p>这是一个dom节点</p>',document.createElement('p'),['<p>这是一个dom节点</p>',document.createElement('p')]。

```javascript
var adjacencyOperators = [ 'after', 'prepend', 'before', 'append' ];
adjacencyOperators.forEach(function(operator, operatorIndex) {
//prepend和append的时候inside为1
  var inside = operatorIndex % 2
  $.fn[operator] = function(){
    var argType, nodes = $.map(arguments, function(arg) {
          var arr = []
//判断arg的类型，var arg = document.createElement('span');此时div类型为object
//var arg = $('div'),此时类型为array
//var arg = '<div>这是一个div</div>',此时类型为string
          argType = type(arg)
//传入为一个数组时
          if (argType == "array") {
            arg.forEach(function(el) {
              if (el.nodeType !== undefined) return arr.push(el)
              else if ($.zepto.isZ(el)) return arr = arr.concat(el.get())
              arr = arr.concat(zepto.fragment(el))
            })
            return arr
          }
          return argType == "object" || arg == null ?
            arg : zepto.fragment(arg)
        }),
        parent, copyByClone = this.length > 1
    if (nodes.length < 1) return this

    return this.each(function(_, target){
//当为prepend和append的时候其parent为target
      parent = inside ? target : target.parentNode
//将所有的动作全部调成before的动作，其只是改变parent和target的不同
      target = operatorIndex == 0 ? target.nextSibling :
               operatorIndex == 1 ? target.firstChild :
               operatorIndex == 2 ? target :
               null
//检测parent是否在document
      var parentInDocument = $.contains(document.documentElement, parent)
      nodes.forEach(function(node){
        if (copyByClone) node = node.cloneNode(true)
        else if (!parent) return $(node).remove()
//parent.insertBefore(node,parent)其在当前节点的某个子节点之前再插入一个子节点
//如果parent为null则node将被插入到子节点的末尾。如果node已经在DOM树中，node首先会从DOM树中移除
        parent.insertBefore(node, target)
//如果父元素在 document 内，则调用 traverseNode 来处理 node 节点及 node 节点的所有子节点。主要是检测 node 节点或其子节点是否为 script且没有src地址。
        if (parentInDocument) traverseNode(node, function(el){
          if (el.nodeName != null && el.nodeName.toUpperCase() === 'SCRIPT' &&
             (!el.type || el.type === 'text/javascript') && !el.src){
//由于在iframe中有独立的window对象
//同时由于insertBefore插入脚本，并不会执行脚本，所以要通过evel的形式去设置。
            var target = el.ownerDocument ? el.ownerDocument.defaultView : window
            target['eval'].call(target, el.innerHTML)
          }
        })
      })
    })
  }
  //该方法生成了方法名，同时对after、prepend、before、append、insertBefore、insertAfter、prependTo八个方法。其核心都是类似的。
  $.fn[inside ? operator+'To' : 'insert'+(operatorIndex ? 'Before' : 'After')] = function(html){
    $(html)[operator](this)
    return this
  }
})
zepto.fragment = function(html, name, properties) {
  var dom, nodes, container,
  containers = {
    'tr': document.createElement('tbody'),
    'tbody': table, 'thead': table, 'tfoot': table,
    'td': tableRow, 'th': tableRow,
    '*': document.createElement('div')
  },
    singleTagRE = /^<(\w+)\s*\/?>(?:<\/\1>|)$/,
    tagExpanderRE = /<(?!area|br|col|embed|hr|img|input|link|meta|param)(([\w:]+)[^>]*)\/>/ig
//如果其只为一个节点，里面没有文本节点和子节点外，类似<p></p>，则dom为p元素。
  if (singleTagRE.test(html)) dom = $(document.createElement(RegExp.$1))
  if (!dom) {
//对html进行修复，如果其为<p/>则修复为<p></p>
    if (html.replace) html = html.replace(tagExpanderRE, "<$1></$2>")
//设置标签的名字。
    if (name === undefined) name = fragmentRE.test(html) && RegExp.$1
    if (!(name in containers)) name = '*'
    container = containers[name]
    container.innerHTML = '' + html
    dom = $.each(slice.call(container.childNodes), function(){
//从DOM中删除一个节点并返回删除的节点
      container.removeChild(this)
    })
  }
//检测属性是否为对象，如果为对象的化，则给元素设置属性。
  if (isPlainObject(properties)) {
    nodes = $(dom)
    $.each(properties, function(key, value) {
      if (methodAttributes.indexOf(key) > -1) nodes[key](value)
      else nodes.attr(key, value)
    })
  }
  return dom
}
```
## css

```javascript
css: function(property, value){
//为读取css样式时
  if (arguments.length < 2) {
    var element = this[0]
    if (typeof property == 'string') {
      if (!element) return
      return element.style[camelize(property)] || getComputedStyle(element, '').getPropertyValue(property)
    } else if (isArray(property)) {
      if (!element) return
      var props = {}
      var computedStyle = getComputedStyle(element, '')
      $.each(property, function(_, prop){
        props[prop] = (element.style[camelize(prop)] || computedStyle.getPropertyValue(prop))
      })
      return props
    }
  }

  var css = ''
  if (type(property) == 'string') {
    if (!value && value !== 0)
      this.each(function(){ this.style.removeProperty(dasherize(property)) })
    else
      css = dasherize(property) + ":" + maybeAddPx(property, value)
  } else {
    for (key in property)
      if (!property[key] && property[key] !== 0)
        this.each(function(){ this.style.removeProperty(dasherize(key)) })
      else
        css += dasherize(key) + ':' + maybeAddPx(key, property[key]) + ';'
  }

  return this.each(function(){ this.style.cssText += ';' + css })
}
//css将font-size转为驼峰命名fontSize
camelize = function(str){ return str.replace(/-+(.)?/g, function(match, chr){ return chr ? chr.toUpperCase() : '' }) }
//将驼峰命名转为普通css:fontSize=>font-size
function dasherize(str) {
  return str.replace(/::/g, '/')
         .replace(/([A-Z]+)([A-Z][a-z])/g, '$1_$2')
         .replace(/([a-z\d])([A-Z])/g, '$1_$2')
         .replace(/_/g, '-')
         .toLowerCase()
}
//可能对数值需要添加'px'
function maybeAddPx(name, value) {
  return (typeof value == "number" && !cssNumber[dasherize(name)]) ? value + "px" : value
}
```

```getComputedStyle```
props[prop] = (element.style[camelize(prop)] || computedStyle.getPropertyValue(prop))
其是一个可以获取当前元素所有最终使用的CSS属性值，返回一个样式对象，只读，其具体用法如下所示，同时读取属性的值是通过getPropertyValue去获取：

其与style的区别在于，后者是可写的，同时后者只能获取元素style属性中的CSS样式，而前者可以获取最终应用在元素上的所有CSS属性对象。该方法的兼容性不错，能够兼容IE9+,但是在IE9中，不能够读取伪类的CSS。
