#### 为什么app.locals定义的键值对能在模板中直接访问呢
***

不知道大家在使用express框架开发的过程中，有没有过这样的疑问，在app.locals这个对象字面量中定义的键值对，是可以直接在模板中使用的，就和res.render时开发者传入的模板渲染参数一样，那么为什么能这样操作呢，本文就是从源码角度浅析下这个问题。

#### res.render做了些什么
其实要探讨大标题的问题，必须先弄明白，当我们在路由函数中使用：
  ```
	res.render(view, data)
  ```
在服务端渲染输出页面的时候，express做了些什么。
这本身就是一个很有意思的问题，因为，大部分Express开发者都遇到过如下二种写法：

```
res.render(view, data);
res.render(view, data, (err, text)=>if(!err)=>res.send(text));
```

很有意思吧，也就是说，当我们调用res的render方法时，如果不传第三个回调函数，则render结束后将结果HTML自动发送给浏览器；如果我们传入第三个回调函数，则服务器端的render页面结果HTML字符串会以该回调函数的第二个参数的形式返回（上述代码样例中的text），此时何时返回调用`res.send`方法将HTML给浏览器由开发者自己决定。

接下来我们就深入Express的源码来理解下，为什么可以这样进行编码。经过查阅express的源码，可以发现`res.render`方法最终是在`express/lib/response.js`中实现的。至于在形如：
```
app.get(‘/index’, (req, res, next)=>res.render(‘index’))
```
的常规路由函数中，初始的http服务器处理句柄中传入的原始req和res何时被express自动包装，提供了诸如`res.send()`, res.json()等方法调用链，这里只提供大略的源码路线，不作详细展开：其实此处大家可以自己看`lib/application.js`，从第一个中间件加载开始（或者没有中间件，直接开始加载路由），都会执行一个lazyrouter方法，那么原始的req和res正是在这里使用原型链赋值的方式进行初始化包装了express提供的
`lib/request.js`和`lib/response.js`方法的。
有点扯远了，回归正途，首先贴一下`res.render`的精简后的源码：
```
function render(view, options, callback) {
	var app = this.req.app;
	var done = callback;
	var opts = options || {};
	var req = this.req;
	var self = this;
	done = done || function (err, str) {
		if (err) return req.next(err);
		self.send(str);
	};
	app.render(view, opts, done);
};
```
这段代码非常容易理解，也解释了本节最开始提出的问题，两种写法都支持的原因，就是：
```
var done = callback;
done = done || function (err, str) {
	if (err) return req.next(err);
	self.send(str);
};
```
显然，如果用户传入了回调函数，则done就是用户传入的回调函数，如果用户没有传入回调函数，则express框架自动给你添加了一个渲染完成没有错误自动将渲染后的HTML返回给浏览器的回调函数，当然这个自动添加的回调函数也提供了简单的异常处理，比如渲染出错，就走next(err)，返回500给浏览器。
接下来就是
```
var app = this.req.app;
app.render(view, opts, done);
```
这里的app可以通过`this.req.app`来获取来获取的原因，就是上面提到的原始req和res在lazyrouter中进行了一系列初始化的结果，具体不展开了。而得到的app，就是原始的express生成的app。
所以，`res.render`方法，最终调用的就是`app.render`方法，并且传入了三个参数`view`，`opts`，`done`。其中view依旧是模板路径，opts则是渲染该模板传入的参数，最后的done就是回调函数。

#### app.render

上面一节分析了这么多，其实正主在这里。我们可以看到，最终所有的render方法，收口的地方在`app.render`函数中。这个函数也能找到本文的主题：`app.locals`中定义的键值对为何能在模板中直接使用，真正原因。
核心的代码如下：
```
function render(name, options, callback) {
	…
	var renderOptions = {};
	var opts = options;
	…
	merge(renderOptions, this.locals);
	if (opts._locals) {
    		merge(renderOptions, opts._locals);
	}
	merge(renderOptions, opts);
	…
	tryRender(view, renderOptions, done);
}

```

我把和大标题无关的view生成代码都去掉了，可以看到，最后调用tryRender方法进行渲染传入的参数renderOptions，其实是由`app.locals`，`options._locals`(如果存在的话)和真正的由开发者传入的渲染页面所需要的参数options，这三者merge而成的。
那么在模板中，真正输出给模板的参数是这个包含上述三者的renderOptions，故而，我们可以在模板中和调用自己传入参数一样的方式，直接调用`app.locals`中定义的键值对。

#### 结语
存储在app.locals中的这些键值对一般是公共模板方法或者公共模板变量，express提供了这样的机制，便于公共数据和方法在模板中的使用，而无需每次render手动传入，这种思想值得我们设计代码框架时学习。

