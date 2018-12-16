iframe的基本概念

iframe通常作为网页中的内联框架在网站中使用。

优点：

1.iframe 可以和主页面并行加载而不会阻塞主页面的渲染加载

2.可以和主页面进行相互的事件沟通和dom操作

iframe.contentWindow, 获取iframe的window对象
iframe.contentDocument, 获取iframe的document对象

 

在同域下，父页面可以获取子iframe的内容，那么子iframe同样也能操作父页面内容。

window.parent 获取上一级的window对象，如果还是iframe则是该iframe的window对象

window.top 获取最顶级容器的window对象，即，就是你打开页面的文档

window.self 返回自身window的引用。可以理解 window===window.self(脑残)

 

不同域的情况下，主页面可以通过路由给iframe传递信息，而且html5支持的postMessage也允许来自不同源的脚本采用异步方式进行有限的通信，可以实现跨文本档、多窗口、跨域消息传递。

发送消息: 使用postmessage方法

window.frames['name'].postMessage(data, origin)，data:要传递的数据（最好是字符串）；origin：字符串参数，指明目标窗口的源，协议+主机+端口号，这个参数是为了安全考虑，someWindow.postMessage方法只会在someWindow所在的源(url的protocol, host, port)和指定源一致时才会成功触发message event，当然如果愿意也可以将参数设置为" * "，someWindow可以在任意源，如果要指定和当前窗口同源的话设置为"/"

接受消息: 监听message事件

window.addEventListener('message', function(message) {

　　//todo  message有三个属性，data：顾名思义，是传递来的message；source：发送消息的窗口对象；origin：发送消息窗口的源（协议+主机+端口号）

}, false)

缺点：

1.iframe会阻塞主页面的onload事件，iframe加载渲染完之后才会触发主页面的onload事件

2.iframe和主页面共享同一个连接池

 

作用：

1.利用它的特性可以提供跨域的解决办法；

2.它还可以模拟实现ajax；

3.可以利用它独立封闭的特点来作为网页中的广告弹窗以及第三方网页的展示框架；

 

这里主要探究 1 和 2 的实现

 

在跨域中的应用

跨域是指js代码在不同域之间进行数据传输和通讯的情况。只要是协议、域名、端口任一不同都是这里的不同域的情况。另外的情况，域名和对应的ip地址也属于不同域，统一域名、不同二级域名也属于不同域

这里跨域主要指一个域的网页js对另一个域下的服务器进行网络请求。

为了演示方便，这里不同域的情况为端口不同。

跨域演示步骤：

这里的示例是hostA（127.0.0.1:3005）域下的 requestA.html 页面对 hostB (127.0.0.1:3006) 域下的 /getUserInfo 接口请求用户信息。

首先将requestA.html放在hostA域下

// hostA.js

let Http = require('http')
let fs = require('fs')

const server = Http.createServer((req, res) => {
    res.setHeader('Content-Type', 'text/html') //设置响应格式为html页面
    res.writeHead(200, 'ok')
    if (req.url === '/requestA.html') {
        res.write(fs.readFileSync('./requestA.html')) //返回requestA页面
    }
    res.end()
})

server.listen(3005, '127.0.0.1', () => { //服务器为本机，端口3005
    console.log('serverA listen in 127.0.0.1:3005')
})
然后在requestA页面向hostB服务器发送请求，并将响应打印出来

// requestA.html

<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>requestA</title>
    <link rel="stylesheet" href="">
</head>
<body>
    <h1>我是requestA页面</h1>
    <button type="" onclick="request('getUserInfo')">getUserInfo</button> //点击按钮向hostB发送请求
    <script type="text/javascript">
        function request(api) {
            var ajaxObj
            if (window.XMLHttpRequest) {
                ajaxObj = new XMLHttpRequest
            } else {
                ajaxObj = new ActiveXObject("Microsoft.XMLHTTP")
            }
            ajaxObj.onreadystatechange = function() {
                if (ajaxObj.readyState === 4 && ajaxObj.status === 200) {
                    document.write(ajaxObj.responseText) //打印返回信息
                }
            }
            ajaxObj.open('get', 'http://127.0.0.1:3006/' + 'api', true)
            ajaxObj.send()
        }
    </script>
</body>
</html>
设置hostB服务器的处理

// hostB.js

let Http = require('http')
let fs = require('fs')

const server = Http.createServer((req, res) => {
    res.setHeader('Content-Type', 'application/json')
    res.writeHead(200, 'ok')
    if (req.url === '/getUserInfo') {
        res.write(JSON.stringify({name: 'zzp', age: 22})) //返回用户信息
    }
    res.end()
})

server.listen(3006, '127.0.0.1', () => {
    console.log('serverA listen in 127.0.0.1:3006')
})
请求结果，如下图提示因为发生了跨域而不能正常的完成http请求响应（从127.0.0.1:3005到127.0.0.1:3006的http请求因为CROS(跨域资源共享)协议被锁定）

 

当然解决这个跨域问题最简单的方法就是根据这个错误提示 (No 'Access-Control-Allow-Origin' header is present on the requested resource.) 来在服务端 hostB.js 设置 'Access-Control-Allow-Origin' 响应头信息

// hostB.js

res.writeHead(200, 'ok', {'Access-Control-Allow-Origin': 'http://127.0.0.1:3005'}) //允许127.0.0.1:3005对本服务器进行http跨域请求
//或者

res.writeHead(200, 'ok', {'Access-Control-Allow-Origin': '*'}) //允许所有域对本服务器进行http跨域请求


可见，这样就解决了跨域的问题，但是因为这里我们想要研究的是iframe在跨域中的应用，而且这种方法需要后端来实现，所以这种方法不再予以考虑。

下面接着研究如何通过iframe来解决上面的跨域问题

 

第一种方法：通过一个主页面和2个嵌套的iframe来实现跨域的解决方案

技术要点：同域下的主页面和iframe可以互相拿到window对象，进而传递事件操作DOM；同域下的html页面可以向服务器发送ajax请求并不被跨域协议限制；不同域下的主页面可以通过路由向iframe传递信息

首先在requestA.html页面里设置一个iframeB，这个iframeB链接向hostB下的一个html页面iframeB.html，将requestA.html请求的参数通过路由传递给iframeB.html，然后由iframeB.html向hostB发送ajax请求并接收响应结果，之后在iframeB.html页面里面再嵌套

一个iframeA，iframeA又链接向hostA下的iframeA.html页面，iframeB同样将响应信息通过路由传递给iframeA，最后再由iframeA将响应信息直接传回requestA.html。

 

将iframeA和iframeB分别添加到hostA和hostB服务器下面

// hostA.js

if (req.url === '/iframeA.html') {
    res.write(fs.readFileSync('./iframeA.html'))
}
// hostB.js

if (req.url === '/iframeB.html') {
    res.write(fs.readFileSync('./iframeB.html'))
}
 

对requestA.html页面做一些修改，删除ajax函数，添加iframeB生成函数，并通过路由将请求参数传给iframeB，最好将iframeB设为隐藏，以防影响主页面布局

// requestA.html

<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>requestA</title>
    <link rel="stylesheet" href="">
    <style type="text/css">
        #iframeB {
            width: 500px;
            height: 400px;
            display: none;
        }
    </style>
</head>
<body>
    <h1>我是requestA页面</h1>
    <button type="" onclick="request('getUserInfo')">getUserInfo</button>
    <script type="text/javascript">
        function request(api) {
            let iframeB = document.createElement('iframe')
            iframeB.src = "http://127.0.0.1:3006/iframeB.html#" + api // 将请求参数传给iframeB.html页面
            iframeB.id = "iframeB"
            document.body.appendChild(iframeB)
        }
    </script>
</body>
</html>
 

在iframeB.html页面里首先拿到requestA传过来的请求参数，然后根据请求参数向hostB发送ajax请求，拿到响应数据之后在页面里面嵌套一个iframeA，同时将响应数据通过路由传给iframeA。

// iframeA.html

<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>iframeB</title>
    <link rel="stylesheet" href="">
    <style type="text/css">
        #iframeA {
            width: 400px;
            height: 300px;
            display: none;
        }
    </style>
</head>
<body>
    <h1>iframeB</h1>
    <script type="text/javascript">
        window.onload = function() {
            let api = window.location.hash.split('#')[1] //拿到请求参数
            request(api) //向hostB发送ajax请求
        }
        function request(api) {
            var ajaxObj
            var data
            if (window.XMLHttpRequest) {
                ajaxObj = new XMLHttpRequest
            } else {
                ajaxObj = new ActiveXObject("Microsoft.XMLHTTP")
            }
            ajaxObj.onreadystatechange = function() {
                if (ajaxObj.readyState === 4 && ajaxObj.status === 200) {
                    data = ajaxObj.responseText
                    callbackIframeA(data)
                }
            }
            ajaxObj.open('get', 'http://127.0.0.1:3006/' + api, true)
            ajaxObj.send()
        }
        function callbackIframeA(data) { //嵌套iframeA，将响应数据传给iframeA
            let iframeA = document.createElement('iframe')
            iframeA.src = "http://127.0.0.1:3005/iframeA.html#" + data
            iframeA.id= "iframeA"
            document.body.appendChild(iframeA)
        }
    </script>
</body>
</html>
 

在iframeA里面拿到路由传过来的响应数据并在主页面requestA.html里面显示出来

<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>iframeA</title>
    <link rel="stylesheet" href="">
</head>
<body>
    <h1>iframeA</h1>
    <script type="text/javascript">
        window.onload = function() {
            let data = decodeURIComponent(window.location.hash).split('#')[1]
            let p = document.createElement('p')
            p.innerHTML = data
            window.top.document.body.appendChild(p) //window.top可以直接拿到最顶层容器的window对象
        }
    </script>
</body>
</html>
实现成果



可以看出，在requestA页面成功拿到了向hostB请求的信息并且没有任何错误警告。

至此，就完成了requestA.html对hostB的跨域请求，事件路线为：requestA.html————iframeB.html————hostB————iframeB.html————iframeA.html————requestA.html

 

第二种方法：使用CDM（cross document messaging）进行不同域下的主页面和iframe的跨域消息传递，hostA域下的主页面requestA.html通过postmessage将请求参数传给内嵌的hostB域下的iframeB.html，iframeB.html收到请求参数之后向hostB服务器发送

ajax请求，接收到响应后将响应信息通过postmessage再传回主页面。

技术要点：不同域下主页面和iframe可以通过postmessage进行通信。

 

在requestA.html里面给iframeB发送请求参数

// requestA.html

<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>requestA</title>
    <link rel="stylesheet" href="">
    <style type="text/css">
        #iframeB {
            width: 500px;
            height: 400px;
            display: none;
        }
    </style>
</head>
<body>
    <h1>我是requestA页面</h1>
    <button type="" onclick="request('getUserInfo')">getUserInfo</button>
    <iframe src="http://127.0.0.1:3006/iframeB.html" id="iframeB" name="iframeB"></iframe>
    <script type="text/javascript">
        function request(api) {
            window.frames['iframeB'].postMessage(api, 'http://127.0.0.1:3006/') //给iframeB发送请求参数，需要指定目标窗体iframeB的源'http://127.0.0.1:3006'
        }

        window.addEventListener('message' ,function(message) { //监听iframeB传回的响应信息
            let p = document.createElement('p')
            p.innerHTML = message.data
            document.body.appendChild(p)
        }, false)
    </script>
</body>
</html>
 

在iframeB拿到请求参数并向hostB发送ajax请求并将响应信息再通过postMessage传回requestA.html

// iframeB.html

<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <title>iframeB</title>
    <link rel="stylesheet" href="">
    <style type="text/css">
        #iframeA {
            width: 400px;
            height: 300px;
            display: none;
        }
    </style>
</head>
<body>
    <h1>iframeB</h1>
    <script type="text/javascript">
        window.addEventListener('message', function (message) { //接受请求参数
            request(message.data)
        }, false)
        function callbackRequestA(data) {
            parent.postMessage(data, 'http://127.0.0.1:3005/') //将响应信息通过postMessage传给requestA.html
        }
        function request(api) {
            var ajaxObj
            var data
            if (window.XMLHttpRequest) {
                ajaxObj = new XMLHttpRequest
            } else {
                ajaxObj = new ActiveXObject("Microsoft.XMLHTTP")
            }
            ajaxObj.onreadystatechange = function() {
                if (ajaxObj.readyState === 4 && ajaxObj.status === 200) {
                    data = ajaxObj.responseText
                    callbackRequestA(data)
                }
            }
            ajaxObj.open('get', 'http://127.0.0.1:3006/' + api, true)
            ajaxObj.send()
        }
    </script>
</body>
</html>
实现成果



和第一种方法相比明显的简洁了很多，不过这种方法只能在html5中使用，兼容性在ie8+。

实现路线总结：requestA.html————iframeB.html————hostB————iframeB.html————requestA.html

 

第二种方法：对于相同主域，不同子域的情况，比如 http://www.a.com 和 http://a.com 。可以通过设置两个页面 document.domain = 'http://a.com' 使他们指向相同的域名，之后就可以直接获取相互的window进行通信了。因为这种方式需要配合域名来验证，

所以就不再这里进行演示。
