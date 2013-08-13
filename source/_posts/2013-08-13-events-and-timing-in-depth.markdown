---
layout: post
title: "理解事件队列"
date: 2013-08-13 19:28
comments: true
categories: 
---

这篇文章的原型是来自于[JavaScript Tutorial](http://javascript.info/)（作者:Ilya Kantor）的其中一小节[Events and timing in-depth](http://javascript.info/tutorial/events-and-timing-depth)，不能算是翻译，因为我不会把一整节内容都搬过来，只写关键的事件队列部分。

浏览器中的JavaScript引擎是一种基于事件驱动的单线程模型，无论在什么时候都只且只有一个JavaScript线程在运行程序，**事件**可以看作是浏览器分发给JavaScript引擎的许多任务，这些任务可以是JavaScript引擎当前执行的代码块，也可以来自浏览器内核的其它线程，比如鼠标点击事件，定时器时间到达通知，异步请求状态变更通知等，JavaScript引擎一直等待着任务队列中任务的到来，由于JavaScript单线程的关系，这些任务必须得排队等着被引擎挨个收拾。

<!--more-->

浏览器本身是允许多个线程异步执行的，除了JavaScript引擎线程以外还有GUI渲染线程（负责界面渲染）、浏览器事件触发线程、定时触发线程、HTTP请求线程、AJAX请求线程、下载线程等等，其中前三个线程属于常驻线程，说到这里不得不提一下GUI渲染线程，虽说浏览器支持线程异步执行，但是JavaScript线程和GUI渲染线程是互斥的，也就是说在JavaScript脚本操作DOM时，GUI渲染线程处于挂起状态不会有任何动作，比如添加元素、删除元素或者改变元素外观等等，界面的更新并不会立即体现出来，所有的操作都会保存在一个队列中，直到脚本运行结束后，GUI渲染线程发现脚本执行触发了界面的Reflow或者Repaint动作（关于这两个动作的区别和触发时机不在本文详细说明，有兴趣的可以自行google），此时才会接手对界面进行渲染（这也是为什么网页优化建议中js文件要放在html内容的最后，就是因为加载js的时候，会阻塞DOM树的构建），下面我们看个小栗子：

	(function() {
    	var htmlStr = '';
    	var el = document.getElementById('main');
    
        for (var i = 0; i <= 100; i++) {
            var str = '<div id="test' + i + '">test' + i + '</div>';
            htmlStr += str;
        }

    	var el = document.getElementById('main');
    	el.innerHTML = htmlStr;
    	
    	// Timeout 2 seconds!
    	sleep(2000);

    	var last = document.getElementById('test100');
    	last.addEventListener('click', function(e) {
        	alert(this.id);
    	}, false);

    	function sleep(numberMillis) {
        	var now = new Date();
        	var exitTime = now.getTime() + numberMillis;
        	while (true) {
            	now = new Date();
            	if (now.getTime() > exitTime)
                	return;
        	}
    	}
	})();
	
代码中使用了一个小手段模拟挂起函数，此时浏览器的行为并不是先显示出插入的所有节点然后再执行事件绑定，而是会有两秒钟的等待时间，然后GUI渲染线程才会讲被插入的元素进行更新和显示。

接下来是见证奇迹的时刻，如果我们把代码改成下面这个样子你猜会发生什么事情？

	(function() {
    	var htmlStr = '';
    	var el = document.getElementById('main');
    	
    	// Wrapped by setTimeout!
    	setTimeout(function() {
        	for (var i = 0; i <= 100; i++) {
            	var str = '<div id="test' + i + '">test' + i + '</div>';
            	htmlStr += str;
        	}
        }, 0);

    	var el = document.getElementById('main');
    	el.innerHTML = htmlStr;
    	    	
    	var last = document.getElementById('test100');
    	last.addEventListener('click', function(e) {
        	alert(this.id);
    	}, false);		
	})();
	
不管是用firebug还是web developer tool，在控制台里都会看到 <span style="color:red">"Uncaught TypeError: Cannot call method 'addEventListener' of null"</span>，也就是说压根没找到last这个元素，这究竟是怎么回事？setTimeout是延迟执行某段脚本，但是如果延迟时间设置为0不是就等于没有延迟么？

这就和任务（事件）队列有关系了，前面说过JavaScript引擎会一直等待任务队列中任务的到来，而setTimeout就会使定时触发线程产生 **异步定时事件** 放在任务队列的最后，等队列中排在它前面的事件执行完了之后才会执行，setTimeout的执行时间点只是加入javascript主执行队列中的时间点，至于什么时候执行，是由js引擎线程按顺序执行的队列来决定，因此虽然我们设置了0毫秒延时，但是由于跳出了当前js执行线程的上下文环境，所以还是会有一个等待的时间，许多文章会说这个等待时间的极限（如果队列中没有其他事件的话）是16ms，但是现如今这个时间已经被大大缩短：

	在早期，js的callback执行，是依赖CPU的中断来进行控制的，如果两个中断之间时间太短会导致，CPU性能消耗很高，同时影响能耗，于是微软和英特公司为了解决这个问题，就约定每个中断之间的间隔是15.6ms（64 fps）所以就是我们常见的约等于16ms的间隔。不过随着web的要求不断增加，大家希望放宽这个时间，于是在高端浏览器，这个性能被提升了4倍左右，所以在chrome，ie10等浏览器，setTimeout的间隔缩短到了4ms（250 fps）。

	注：浏览器模型定时计数器并不是由JavaScript引擎计数的,因为JavaScript引擎是单线程的,如果处于阻塞线程状态就无法计时，因此它必须依赖外部来计时并触发定时。
	
利用setTimeout(callbakFunction, 0)这个特性，我们可以解决很多问题，比如：

	<input id='my' type="text">
	<script>
	document.getElementById('my').onkeypress = function(event) {
		this.value = this.value.toUpperCase();
	}
	</script>
	
这段代码实际上是无效的，因为keypress执行时浏览器还没有把输入值渲染到DOM结构中，因此也无法讲其转换为大写字母，但是如果我们使用 **setTimeout(callbackFunction, 0)** 就可以搞掂它：

	<input id='my' type="text">
	<script>
	document.getElementById('my').onkeypress = function(event) {
		var self = this;
  		setTimeout(function() {
    		self.value = self.value.toUpperCase()
  		}, 0);
	}
	</script>
	
最后，再说回GUI渲染线程和JavaScript线程互相阻塞的问题，有没有办法使二者无阻塞运行呢？答案是“有！”

随着HTML5技术的发展，在浏览器GUI线程外运行javascript代码成为了可能。[WebWorker规范](http://www.whatwg.org/specs/web-apps/current-work/multipage/workers.html) 提供了一个简单的方式让javascript代码在后台线程运行而不影响UI线程。每一个webworker间都是相互独立的，都在自己的线程中运行，现阶段各浏览器对规范的实现并不统一，但是我们仍然对其充满期待，因为它的多线程特性为基于Web系统开发的程序猿们提供了强大的并发程序设计功能，允许开发人员设计开发出性能和交互更好的富客户端应用程序。



	


