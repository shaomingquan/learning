记录遗漏的小tips...不定期更新~

***bind 绑定默认参数***
```
var a = function (arg1, arg2) { console.log(arg1, arg2); }
var b = a.bind(null, 2);
b(); //2,undefined
b(3); //2,3
var bb = b.bind(null, 3); //继续bind
bb(); //2,3
```
之前一直这样写curry的。
```
function curry (func) {
    var args = [];
    var argLength = func.length;
    var ctx = this;
    
    return function _ () {
        var _args = [].slice.call(arguments);
        args = args.concat(_args);
        if(args.length >= argLength) {
            var ret = func.apply(ctx, args);
            args = [];
            return ret;
        } else {
            return _;
        }
    }
}
```
如今利用这个遗漏的tip。貌似这次是真的curry吧。
```
function curry2 (func) {
    var argLength = func.length;
    var counter = 0;
    var _func = func;
    var ctx = this;

    return function _ () {
        counter += arguments.length;
        _func = _func.bind(ctx, ...arguments);
        if(counter >= argLength) {
            var ret = _func();
            _func = func;
            counter = 0;
            return ret;
        } else {
            return _;
        }
    }

}
```

***“提前执行”***
如下代码，当我点一下`#logo`，会打印什么？
```
document.getElementById('logo').onclick = function () {
	console.log('logo clicked');
    document.addEventListener('click', function () {
        console.log('document clicked');
    }, false);
};
```
目的是点`#logo`在全局绑一个点击。但两条都打印，看起来很奇葩，其实很合理。只因为冒泡，事件没冒泡完毕就绑定了document的事件，document不幸被冒泡。如下nextTick一下就ok了。如果不知道事件冒泡的绝对百思不得其解，即使知道事件冒泡也难免踩坑。
```
document.getElementById('logo').onclick = function () {
	console.log('logo clicked');
	setTimeout(function () {
		document.addEventListener('click', function () {
            console.log('document clicked');
        }, false);
	})
};
```

***timer small diff***
不管你如何理解`requestAnimationFrame`这个方法，你需要知道的是`requestAnimationFrame`在当前tab被切走的时候不能继续运行。`setInterval`会继续运行。

但是，在Iphone上面，根据app的不同，切到后台的时候`requestAnimationFrame`的表现也有所不同。如果app有定制化的容器，那么可能会在后台继续执行`requestAnimationFrame`，但是Safari浏览器的表现符合预期，切tab表现与浏览器一致。

***real fps***
浏览器真的总是保持60fps吗？不是。
以transition为例，当动画过快或者动画过慢的时候fps会降低，降低的原因有所不同，过快而降低是因为即使60fps也会发生视觉中断，过慢而降低是即使降低fps也够顺滑。这是js不好控制的（或许有的库已经有类似优化），而浏览器原生支持的。

***event 变量***
之前看过同事这样一段代码。
```
dom.onclick = function (e) {
    func(event);
}
```
我说这不对啊应该是`func(e)`。没成想真是对的。
```
document.onclick = function (e){
	console.log(e === event); // true   event 是个特殊的变量，在event发生的时候会自动赋值。
}
```
***image preview***
一种最简单的图片preview的方法。使用浏览器默认的preview效果，报一个url即可。
```html
<a href="/statics/images/example.png">
  ![](/statics/images/example.png)
</a>
```

***hack curry***

idea from [my boss](https://ljw.me/).
```
function sum (a, b, c) {
	return a + b + c;
}

function* currySum () {
	return sum(yield, yield, yield)
}

var _ = currySum();
_.next();
_.next(1);
_.next(2);
_.next(3); // Object {value: 6, done: true}
```
原理大概是上面那样。可以改装成通用的curry。
```js
function curry (func) {
	function* _ () {
		var args = [];
		while(true) {
            var arg = yield;
            if(arg) {
                args.push(arg);
            } else {
                return func(...args);
            }
		}
	}
	var gene = _();
	gene.next();
	return gene;
}

function join(...args) {
	return args.join('');
}

var _ = curry(join);
_.next(1)
// Object {value: undefined, done: false}
_.next(2)
// Object {value: undefined, done: false}
_.next(3)
// Object {value: undefined, done: false}
_.next()
// Object {value: "123", done: true}
```
这里已经有curry的3种实现了。。

***checked 伪类注意事项 ***

使用checked伪类即可自定义选框样式。但有一点需要注意。
- 不要用`display: none;` 使选框丧失tab键的可访问性

见此例http://dabblet.com/gist/e269f10328615254e29e

***resolve 一个Promise实例***
记在这里，加深印象。如下。
```
var p1 = new Promise(function (resolve, reject) {
  setTimeout(() => reject(new Error('fail')), 5000)
})

var p2 = new Promise(function (resolve, reject) {
  setTimeout(() => resolve(p1), 4000)
})

p2
  .then(result => console.log(result))
  .catch(error => console.log(error))
// Error: fail
```
当p2 resolve p1，那么p1离开pending状态，且继承了p2的then和catch调用。注意Promise一旦被实例出来，异步调用就已经开始执行，所以是5000ms后打印，而不是9000。

***Promise.all 可传任意Iterator实现***
关于Iterator是使用ES6应该活用的点，解构，展开，传参，遍历...很多操作都是Iterator通用。同期推出的Promise.all也支持Iterator，这让我觉得这个思路很重要。所以像下面的代码也不奇怪了。
```
function* pros () {
	yield Promise.resolve(1);
	yield Promise.resolve(2);
	yield Promise.resolve(3);	
}
Promise.all(pros()).then(_ => console.log(_))
// [1, 2, 3]
```

***culc 与预处理器***
最大的区别在于，预处理器只能算绝对值，无法处理动态的相对情况。下面效果预处理器就做不到。
```
#wrapper {
  min-height: calc(100vh - 7em);
}
```
***冷门调速方法***
cubic-bezier，是css3动画中最普遍的调速方法。极少有人会知道还有个调速方式是steps。它可以作为序列帧的实现，与cubic-bezier统称为调速方法。还是要多刷官方文档，少刷快餐文。https://developer.mozilla.org/en-US/docs/Web/CSS/single-transition-timing-function。

***tcp***
一定超时才重发吗？
没收到ack一定会重发吗？
上面两者在tcp滑动窗口控制的情况下皆为否定答案。（图解TCP6.4.7）