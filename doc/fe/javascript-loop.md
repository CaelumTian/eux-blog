---
title: "Javascript事件轮询"
author: "sucer"
datetime: 2015-10-11 08:00:00
cover: "http://ww1.sinaimg.cn/large/43b712ebjw1ey4ahrbx8hj20e80803yl.jpg"
---

原作者： [erin](http://blog.carbonfive.com/author/erin/)  



#### <span class="s1">这篇文章关于什么？</span>

<span class="s1">Javascript作为浏览器脚本语言，已经逐渐变得无处不在，它让你对事件驱动模型有了基本理解，以及它与request-response模型的典型语言，如Ruby，Python和Java的区别，我将阐述一些关于Javascript一致的核心概念，包括它的事件轮询和消息队列，希望能帮助你理解这门或许你不是完全理解的语言。</span>  



#### <span class="s1">致读者：</span>

<span class="s1">这篇文章写给准备使用Javascript进行服务端/客户端开发的web开发工作者（或者准备从事该职业者），如果你对事件轮询机制有比较好的了解，这篇文章或许你会觉得很熟悉。对于不是很了解事件轮询机制的读者，我希望能帮助你对这每天读写的代码有一个基本的理解。</span>  



### <span class="s1">非阻塞I/O</span>

<span class="s1">在Javascript中，几乎所有的I/O都是非阻塞的，其中包括http请求，数据操作以及磁盘读写；单线程询问任务运行时机以及执行任务，通过使用回调函数，能让Javascript线程在回调完成之前执行其它任务。当一个执行完成的时候，会去执行由回调函数提供的有序消息队列中的下一个任务。</span>  


<span class="s1">当开发者熟悉了这种交互模式，用户习惯了这种界面</span> <span class="s1">—</span> <span class="s1">当事件发生，例如“mousedown”，“click”这种随时可能被触发的事件，它不同于同步机制，请求-响应模型很少用在服务端应用上。</span>  


<span class="s1">让我们比较两块代码，它们分别发起HTTP请求于www.google.com然后在控制台输出响应结果。首先，Ruby，使用Faraday：</span>  



```
response = Faraday.get 'http://www.google.com'
puts response
puts 'Done!'
```

<span class="s1">执行结果如下所示：</span>  




2. 
<span class="s1">get方法已执行，然而该线程直到有响应才被回收</span>  



4. 
<span class="s1">该来自Google的响应返回的数据并没有存在变量中</span>  



6. 
<span class="s1">响应结果输出在控制台中</span>  



8. 
<span class="s1">直至最后，“任务完成”才出现在控制台中</span>  





<span class="s1">让我们用Javascript的Node.js及它的Request库来做同样的事：</span>  



```
request('http://www.google.com', function(error, response, body) {
  console.log(body);
});

console.log('Done!');
```

<span class="s1">看上去差不多，但结果截然不同：</span>  




2. 
<span class="s1">请求已被执行，在请求得到响应之前则已跳过一个匿名回调函数（该函数并未执行）</span>  



4. 
<span class="s1">“任务完成”马上出现在控制台中</span>  



6. 
<span class="s1">一段时间之后，收到响应，此时回调函数才被执行—在控制台输出响应结果</span>  






### <span class="s1">事件轮询</span>

<span class="s1">非耦合机制使得Javascript线程能在等待异步操作完成及其回调函数执行之前执行其它任务。那么，在内存中在哪激活回调？回调按什么规则执行？什么会让回调执行呢？</span>  


<span class="s1">Javascript线程包括一个储存了待执行任务的消息列表的消息队列，以及与它们相关联的回调函数，这些消息按照它们的响应顺序排列（例如鼠标点击，或者收到来自HTTP请求的响应）每条消息都有回调函数，如果没有提供回调函数：例如当用户点击一个按钮但是没有提供回调函数，则没有消息会被添加到消息队列。</span>  


<span class="s1">在每一次轮询中，任务队列会记录下一条消息（每次记录会返回一个“tick”），当轮询到这条消息时，该消息所对应的回调函数则被执行。</span>  


<img src="http://ww1.sinaimg.cn/large/43b712ebjw1ewkmap1po0j20m00cktat.jpg" alt="" width="792" height="452" />
  


<span class="s1">在最初的架构中，回调函数通过调用栈来实现，由于Javascript是单线程的，消息队列是阻塞的，对于后续任务，必须等待之前的任务返回栈中所有回调函数，才能将新任务的回调函数加入到栈中。在随后的架构中加入了函数（同步的）对栈的新的调用方法（此处例举一个初始化为changeColor的函数）。</span>  



```
function init() {
  var link = document.getElementById("foo");

  link.addEventListener("click", function changeColor() {
    this.style.color = "burlywood";
  });
}

init();
```

<span class="s1">在这个例子中，当用户点击“foo”这个元素，“onclick”事件被触发，一个消息（以及回调函数changeColor）加入消息队列。当队列按序执行到该消息时，它的回调函数changeColor被唤起。</span>  


<span class="s1">当回调函数changeColor返回（或者出错被丢掉），事件轮询继续执行。只要与”foo”元素的onclick事件绑定的回调函数changeColor存在，随后在该元素上的click事件都会使得更多的消息（及其回调函数changeColor）被加入队列。</span>  



### <span class="s1">消息队列的添加</span>

<span class="s1">如果你申明了一个异步函数（例如setTimeout），其回调函数最终会在一个不同的消息队列中执行，在未来的事件轮询中的某个时刻。例如:</span>  



```
function f() {
  console.log("foo");
  setTimeout(g, 0);
  console.log("baz");
  h();
}

function g() {
  console.log("bar");
}

function h() {
  console.log("blix");
}

f();
```

<span class="s1">由于setTimeout非阻塞的本质，它的回调函数的未来的若干毫秒后执行并且等待期间不占用该消息的进程。在这个例子中，setTimeout跳过它的回调函数g和一段事件的延迟后被唤起。当预先声明的时间结束（在这个例子中几乎是立即执行）被分离出去的消息又被重新加回队列，包括其回调函数g。这个回调函数被激活就好比:”foo”,”baz”,”blix”然后执行下一个事件轮询的tick:”bar”。如果在一个框架中同时声明了两个setTimeout，并且他们的第二个参数（执行时间）想同，他们的回调将会按照其定义顺序执行。</span>  



### <span class="s1">Web Workers</span>

<span class="s1">利用Web Workers能让你丢掉昂贵的多线程执行方式，释放主线程去做其他的事。Web Workers包括单独的消息队列，事件轮询，以及实例化了一个独立于最初的主线程的储存空间。利用消息传递来建立消息与主线程之间的联系，这种联系非常像我们刚才的代码示例。</span>  


<img src="http://ww1.sinaimg.cn/large/43b712ebjw1ewkm02zdhtj20ol0a240q.jpg" alt="" width="885" height="362" />
  


首先，我们的worker:  



```
// our worker, which does some CPU-intensive operation
var reportResult = function(e) {
  pi = SomeLib.computePiToSpecifiedDecimals(e.data);
  postMessage(pi);
};

onmessage = reportResult;
```

 然后，这是在html中的代码内容：  



```
// our main code, in a <script>-tag in our HTML page
var piWorker = new Worker("pi_calculator.js");
var logResult = function(e) {
  console.log("PI: " + e.data);
};

piWorker.addEventListener("message", logResult, false);
piWorker.postMessage(100000);
```

 该示例中，主线程产生一个worker然后将一个logResult回调函数注册到消息队列中。在worker中，reportResult函数被注册到它自己的消息事件中。当worker线程从主线程接收消息时，worker将消息及其相应的回调函数加入队列中。当消息队列按顺序执行到该消息时，主线程将发回一条消息并将一条新的消息加入队列（按照logResult的回调排序）由此开发者能让CPU集中处理分线程，释放主线程继续处理消息任务及其绑定事件。  



### <span class="s1">关于闭包</span>

<span class="s1">Javascript支持闭包，准许注册回调，当我们执行回调时，通过执行回调创造的新的完全调用栈来维持我们创造的环境的入口。回调函数作为不同于我们创造的消息的一部分被调用。考虑如下示例：</span>  



```
function changeHeaderDeferred() {
  var header = document.getElementById("header");
  
  setTimeout(function changeHeader() {
    header.style.color = "red";

    return false;
  }, 100);

  return false;
}

changeHeaderDeferred();
```

<span class="s1">在这个示例中，以头变量方式声明的changeHeaderDeferred函数被执行。setTimeout函数被唤醒，导致消息（加在changeHeader回调中的）大约在100毫秒之后（时间偏差源于每台计算机内置原子钟差异）添加到消息队列，changeHeaderDeferred返回false，结束第一条消息的进程，然而头变量依然通过闭包的方式存在，没有被垃圾回收机制回收。当第二条消息执行（changeHeader函数）维持头变量声明的外部函数域的入口。一旦第二条消息（changeHeader函数）执行完毕，头变量则被回收。</span>  



### <span class="s1">另外</span>

<span class="s1">Javascript的事件驱动交互模型不同于大多数编程人员习惯的请求-响应模型，但是你能看到，该技术也不是那么高不可攀。一个简单的消息队列及事件轮询，Javascript使得开发者能够围绕收集异步回调的形式来建立他们的系统，在等待外部事件发生的同时释放主线程去做其它操作。它将越来越流行。</span>