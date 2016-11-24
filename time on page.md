#页面停留时间

## 停留时间 = 用户离开页面时间-用户进入页面时间
用户进入页面时间:页面载入时间

## 离开具体指代的事件
* PC端浏览器
	* 页面消失
  		* 点击某个离开页面的链接
   		* 在地址栏中键入了新的URL
   		* 使用前进或后退按钮
   		* 关闭浏览器
   		* 重新加载页面

	* 页面不消失
  		* 最小化浏览器
		* 切换到别的tab页
		
* 移动端浏览器/APP（ios/android）

	* 页面消失
	
		* 点击某个离开页面的链接
   		* 在地址栏中键入了新的URL
   		* 使用前进或后退按钮
   		* 关闭浏览器/APP
   		* 重新加载页面(ios/android表现不同)
	
	* 页面不消失
	
		* 浏览器/APP切换到后台
		* 切换到别的tab页
		
##难点
	1. 捕捉离开页面的瞬间	
	2. 在离开页面的瞬间发送数据

###1.捕捉离开页面的瞬间
浏览器提供了很多接口（如事件）供`js`调用
![chrome.png](http://7i7h07.com1.z0.glb.clouddn.com/image/github/web-analytics/chrome.png)

![chrome.png](http://7i7h07.com1.z0.glb.clouddn.com/image/github/web-analytics/firefox.png)

我们只需给对应的离开事件绑上回调函数就可在响应的事件发生时执行对应的程序。

* 各个浏览器事件不一定相同
* 相同的事件不一定不支持
* 绑定事件的方法不一定相同
-->常说的浏览器兼容分`css`和`js`方面的，这个就是`js`兼容性中一些情况

###找到对应的事件 
####PC端
1. unload
在页面变化的一瞬间，时间极短，这时候发送请求发很容易丢包
1.1 beforeunload
在unload事件发生前调用，但是也不保证完全不会丢包
1.2 visibilitychange
切换tab页时，如果是离开时间-进入页面时间，停留时间就会不准确，因此监听切换tab页的事件（visibilitychange）来刨去中间的时间

**问题**：可能存在丢包

2.谷歌分析（Google Analytics）的方案
在谷歌分析中，一般只有网站停留时间，就算是页面停留时间，第一张页面也是默认放弃的。
谷歌的做法是利用cookie存下本次的时间信息，下次进入页面再将上一次的时间发送给后台
举例：
_utma （用来识别唯一身份访客）
_utma = 127635166.1360367272.1264374807.1264374807.1264374807.1

* 第一组数字被叫做“域哈希”，是GA表示这个域的唯一代码。同一域中每个cookie的第一组数据都是“域哈希”，并且值都是一样的。
* 第二组数字是一个随机产生的唯一ID。
* 第三，四，五组数字是时间戳，其中第三组数字表示初次访问的时间。第四组数字表示上一次访问的时间，第五组数字表示本次访问开始的时间。
* 第六组数字是访问次数计数器。这个数字随着访问次数的增加而增加。
如果是第一次访问，第三四五组时间戳都是相同的。

**问题**：损失了只访问一次的用户，这些用户还是具有价值的

####手机端
手机浏览器（Android/iOS）

* beforeunload不再支持--->pageshow,pagehide
```
	window.addEventListener("pageshow", pageShown, false);
    window.addEventListener("pagehide", pageHidden, false);
   
    function pageShown(evt){
        if (evt.persisted){
        	//The page was just restored from the Page Cache
			//应用切换到前台，再次记录时间
        	params.entrytime = Date.parse(new Date())/1000;
        }
        else{
        	//initial load
        	if(localStorage.time){
        		sendData();
        		localStorage.time = '';
        	}
        }
    }

    function pageHidden(evt){
        if (evt.persisted){
        	//The page was suspended and placed into the Page Cache
        	params.exittime = Date.parse(new Date())/1000;
        	setLocal();
        } else {
        	//page destruction
        	params.exittime = Date.parse(new Date())/1000;
        	setLocal();
        	sendData();
        }
    }
	function setLocal(){
    	localStorage.time = localStorage.time || '';
		localStorage.time += (params.entrytime + '_' + params.exittime + '^');
    }
    function sendData(){
    	var arg = '';
    	var timeOnPage = localStorage.time.split('^');
    	var difftime = 0;
    	for(var i = 0; i < timeOnPage.length && timeOnPage[i]; i++){
    		var time_arr = timeOnPage[i].split('_');
    		difftime += (parseInt(time_arr[1]) - parseInt(time_arr[0]));
		}
    	arg += ('difftime=' + difftime + '&');
    	arg += ('serialId=' + document.getElementById('serialId').value + '&');
    	arg += ('isCliked=true' + '&');
    	var img = new Image(1, 1);
        img.src = 'http://218.205.115.243:10092/1.gif?' + arg;
    }
```

####Hybrid APP
![混合app](http://7i7h07.com1.z0.glb.clouddn.com/image/github/web-analytics/hybrid-app.png)

**优点**：使用Hybrid开发方法，就能集Native 和web两者之所长。一方面，Native 让开发者可以充分利用现代移动设备所提供的全部不同的特性和功能。另一方面，使用 Web 语言编写的所有代码都可以在不同的移动平台之间共享，使得开发和日常维护过程变得集中式、更简短、更经济高效。

**缺点**：还不成熟，支持程度并不理想

ios和android支持pageshow，但是完全不支持pagehide，导致没法探知应用被关闭/切换到后台的状态。正常的做法是，写好app隐藏或者关闭的回调函数，当native层探知事件发生，去调用JS回调函数。

更改方案：
定时向后台发送字段，后面的覆盖前面的。

- 试验的时候是**每秒**向服务器请求一次，但是各个平台下，IOS下应用切换到后台，js会立即停止运行，但是android的机制是如果应用30s还不切换到前台，js就会停止运行，这样导致计时的不准确。

- 也有人说实践过用**裴波那契数列**（1、1、2、3、5、8、13、21、34、……）的形式向后台发请求，这样对后台压力相对小一些

此方案在用户基数很大的时候对服务器造成的压力太大，因此也放弃了。

###2.在离开页面的瞬间发送数据
尽量减缓刷新这个过程
 - 动态新建img，通过**src**属性向后台请求一张1*1的gif图片，请求的同时把收集到的数据带过去。
 - for循环 + ajax
 - navigator.sendBeacon
 ![beacon.png](http://7i7h07.com1.z0.glb.clouddn.com/image/github/web-analytics/beacon.png)
