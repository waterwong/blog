>之前有尝试着去阅读jQuery源码，但是由于源码过长再加上自己技术上有点不到家，在尝试过几遍之后不得不遗憾的选择放弃。对dom的操作一直都使用jQuery，如果让我用原生的JS去操作dom，会发现自己不能很快的实现需求，所以这次选择Zepto的源码去深入挖掘dom操作。注意，此次选用的Zepto选用最新1.2.0的版本，所以不太适用于PC浏览器。

### selector选择器

jQuery有一个非常强大的DOM选择器引擎Sizzle，选择速度堪称业内顶尖，Zepto号称移动端的jQuery，那么其肯定需要一个自己的选择器，现在我们来看看这个非常小巧轻便的选择器。

```javascript
// 该正则匹配a-z,A-Z,下划线,-所连着的单词，验证是否为单个类名
// 'app-532'为true  'app app123'为false
var simpleSelectorRE = /^[\w-]*$/;
zepto.qsa = function(element, selector){
//找到的元素
  var found,
//开头元素为ID
      maybeID = selector[0] == '#',
//开头元素为class
      maybeClass = !maybeID && selector[0] == '.',
//将开头元素为ID或class的#或.去掉
      nameOnly = maybeID || maybeClass ? selector.slice(1) : selector,
//是否为单个选择还是层级选择，注意这里有一个bug（Zepto的bug)。
      isSimple = simpleSelectorRE.test(nameOnly)
  return (element.getElementById && isSimple && maybeID) ?
    ( (found = element.getElementById(nameOnly)) ? [found] : [] ) :
//当element不为元素节点，文档节点和文档片段节点时返回为[]
    (element.nodeType !== 1 && element.nodeType !== 9 && element.nodeType !== 11) ? [] :
    [].slice.call(
//为单个选择且不为ID同时支持getElementsByClassName时，当为class则使用getElementsByClassName，不为class，则使用element.getElementsByTagName
//否则为querySelectorAll
      isSimple && !maybeID && element.getElementsByClassName ?
        maybeClass ? element.getElementsByClassName(nameOnly) :
        element.getElementsByTagName(selector) :
        element.querySelectorAll(selector)
    )
}
dom = zepto.qsa(document, selector)
//querySelectorAll方法支持IE8+,不过在IE8中只支持css2.1选择器。
```
Zepto的选择器非常的小巧，但是又带来了一些问题，当一个元素其id为'app.item'时，使用$('#app.item')往往会出现选择不上的情况，同时Zepto选用querySelectorAll来进行层级选择，而querySelectorAll自身的性能缺陷会导致Zepto选择器的性能过慢，具体可去[选择器API没有性能优化，慎用](https://www.web-tinker.com/article/20377.html)详细了解该性能问题。

解决了选择元素的问题，我们现在可以来看看具体的dom操作了。

### attr
```javascript
attr:function(name,value){
  var result;
//1 in arguments的作用就是判断其是否传入了value
//只传入name且为string的时候则为读取匹配到的第一个元素name属性
  return (typeof name == 'string' && !(1 in arguments)) ?
    (0 in this && this[0].nodeType == 1 && (result = this[0].getAttribute(name)) != null ? result : undefined) :
    this.each(function(idx){
      if (this.nodeType !== 1) return
//如果为name为对象的话，则可以进一步的循环name
      if (isObject(name)) for (key in name) setAttribute(this, key, name[key])
//由于value为非函数，所以funcArg直接返回的是value
      else setAttribute(this, name, funcArg(this, value, idx, this.getAttribute(name)))
    })
}
function funcArg(context, arg, idx, payload) {
  return isFunction(arg) ? arg.call(context, idx, payload) : arg
}
function setAttribute(node,name,value){
  value == null ? node.removeAttribute(name) : node.setAttribute(node,value)
}

```

### class

```javascript
var classCache = {};
//classCache中其初始值为{},当调用该函数时，会返回一个正则表达式，其匹配开头或者空白（包括空格、换行、tab缩进等）+name+结尾或者空白（包括空格、换行、tab缩进等）。不过还是没有弄懂为什么要将class的正则存储起来。
function classRE(name) {
  return name in classCache ?
    classCache[name] : (classCache[name] = new RegExp('(^|\\s)' + name + '(\\s|$)'))
}
//去获取className
function className(node, value){
  var klass = node.className || '',
      svg   = klass && klass.baseVal !== undefined;
//读取
  if (value === undefined) return svg ? klass.baseVal : klass
//设置
  svg ? (klass.baseVal = value) : (node.className = value)
}
//判断是否有该class
hasClass: function(name){
  if (!name) return false
//当this中只要有一个元素包含name这个class，即返回true
  return [].some.call(this, function(el){
    return this.test(className(el))
  }, classRE(name))
},
//添加元素
addClass: function(name){
  if (!name) return this
  return this.each(function(idx){
//排除非dom元素
    if (!('className' in this)) return
    classList = []
    var cls = className(this), newName = funcArg(this, name, idx, cls)
//对当前元素的className进行遍历，如果没有改元素，则将元素push进classList。
    newName.split(/\s+/g).forEach(function(klass){
      if (!$(this).hasClass(klass)) classList.push(klass)
    }, this)
    classList.length && className(this, cls + (cls ? " " : "") + classList.join(" "))
  })
},
//移除元素
removeClass: function(name){
  return this.each(function(idx){
    if (!('className' in this)) return
//当name为空，则移除所有的class
    if (name === undefined) return className(this, '')
    classList = className(this)
    funcArg(this, name, idx, classList).split(/\s+/g).forEach(function(klass){
      classList = classList.replace(classRE(klass), " ")
    })
    className(this, classList.trim())
  })
},
toggleClass: function(name, when){
  if (!name) return this
  return this.each(function(idx){
    var $this = $(this), names = funcArg(this, name, idx, className(this))
    names.split(/\s+/g).forEach(function(klass){
//存在则删除，不存在则添加
      (when === undefined ? !$this.hasClass(klass) : when) ?
        $this.addClass(klass) : $this.removeClass(klass)
    })
  })
},
```
在函数className中有个判断svg的特殊方法，即```svg= klass && klass.baseVal !== undefined```。svg这个元素比较特殊，其通过```document.getElementsByClassName('Icon--logo')[0].className```获取到的内容如下显示：

通过读取className.baseVal的形式来判断是否为svg元素。同时设置其class的方式也不能通过className的形式直接设置，也要通过className.baseVal的形式去设置才能生效。

Zepto的class设置方式非常的巧妙，兼容性也比较好，但是如果只是在移动端使用的话，完全可以用HTML5中提供的```classList```接口去取代，该接口的兼容性非常的棒，目前（2017年11月）能够直接使用，无需考虑兼容性问题，是的，svg元素也能够使用。下面将用classList的形式改写下这些函数(注：下面的函数只能设置读取一个class，如果要实现多个class，稍微在这基础上改写下就行了)：
```javascript
addClass:function(name){
  if(!name)  return this
  return this.forEach(funciton(item){
//classList的add方法：如果这些类已经存在于元素的属性中，那么它们将被忽略
    item.classList.add(name)
  })
}
removeClass:function(name){
  return this.forEach(funciton(item){
    if(!name){
      var klassList = item.className,
          svg   = klassList && klassList.baseVal !== undefined;
      svg ? klassList.baseVal = '' : klassList = ''
    }
    item.classList.remove(name)
  })
}
toggleClass:function(name){
  return this.forEach(function(item){
    item.classList.toggle(name)
  })
}
```
