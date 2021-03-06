# -websocket-
心跳重连缘由（原链接：http://www.cnblogs.com/1wen/p/5808276.html）

websocket是前后端交互的长连接，前后端也都可能因为一些情况导致连接失效并且相互之间没有反馈提醒。因此为了保证连接的可持续性和稳定性，websocket心跳重连就应运而生。

在使用原生websocket的时候，如果设备网络断开，不会触发websocket的任何事件函数，前端程序无法得知当前连接已经断开。这个时候如果调用websocket.send方法，浏览器就会发现消息发不出去，便会立刻或者一定短时间后（不同浏览器或者浏览器版本可能表现不同）触发onclose函数。

后端websocket服务也可能出现异常，连接断开后前端也并没有收到通知，因此需要前端定时发送心跳消息ping，后端收到ping类型的消息，立马返回pong消息，告知前端连接正常。如果一定时间没收到pong消息，就说明连接不正常，前端便会执行重连。

为了解决以上两个问题，以前端作为主动方，定时发送ping消息，用于检测网络和前后端连接问题。一旦发现异常，前端持续执行重连逻辑，直到重连成功。
 
如何实现

在websocket实例化的时候，我们会绑定一些事件：


var ws = new WebSocket(url);

ws.onclose = function () {
    //something
};
ws.onerror = function () {
    //something
};
        
ws.onopen = function () {
   //something
};
ws.onmessage = function (event) {
   //something
}

如果希望websocket连接一直保持，我们会在close或者error上绑定重新连接方法。

ws.onclose = function () {
    reconnect();
};
ws.onerror = function () {
    reconnect();
};
    

这样一般正常情况下失去连接时，触发onclose方法，我们就能执行重连了。

那么针对断网的情况的心跳重连，怎么实现呢。

简单的实现：

var heartCheck = {
    timeout: 60000,//60ms
    timeoutObj: null,
    reset: function(){
        clearTimeout(this.timeoutObj);
　　　　 this.start();
    },
    start: function(){
        this.timeoutObj = setTimeout(function(){
            ws.send("HeartBeat");
        }, this.timeout)
    }
}

ws.onopen = function () {
   heartCheck.start();
};
ws.onmessage = function (event) {
    heartCheck.reset();
}
复制代码
如上代码，heartCheck 的 reset和start方法主要用来控制心跳的定时。

什么条件下执行心跳：

当onopen也就是连接上时，我们便开始start计时，如果在定时时间范围内，onmessage获取到了后端的消息，我们就重置倒计时，

距离上次从后端获取到消息超过60秒之后，执行心跳检测，看是不是断连了，这个检测时间可以自己根据自身情况设定。

判断前端ws断开(断网但不限于断网的情况）：

当心跳检测send方法执行之后，如果当前websocket是断开状态(或者说断网了)，发送超时之后，浏览器的ws会自动触发onclose方法，重连也执行了（onclose方法体绑定了重连事件），如果当前一直是断网状态，重连会2秒（时间是自己代码设置的）执行一次直到网络正常后连接成功。

如此一来，我们判断前端主动断开ws的心跳检测就实现了。为什么说是前端主动断开，因为当前这种情况主要是通过前端ws的事件来判断的，后面说后端主动断开的情况。

 

我本想测试websocket超时时间，又发现了一些新的问题

1. 在chrome中，如果心跳检测 也就是websocket实例执行send之后，15秒内没发送到另一接收端，onclose便会执行。那么超时时间是15秒。

2. 我又打开了Firefox ，Firefox在断网7秒之后，直接执行onclose。说明在Firefox中不需要心跳检测便能自动onclose。

3.  同一代码， reconnect方法 在chrome 执行了一次，Firefox执行了两次。当然我们在几处地方（代码逻辑处和websocket事件处）绑定了reconnect()，

所以保险起见，我们还是给reconnect()方法加上一个锁，保证只执行一次

 

目前来看不同的浏览器，有不同的机制，无论浏览器websocket自身会不会在断网情况下执行onclose，加上心跳重连后，已经能保证onclose的正常触发。

 

判断后端断开：

    如果后端因为一些情况断开了ws，是可控情况下的话，会下发一个断连的消息通知，之后才会断开，我们便会重连。

如果因为一些异常断开了连接，我们是不会感应到的，所以如果我们发送了心跳一定时间之后，后端既没有返回心跳响应消息，前端又没有收到任何其他消息的话，我们就能断定后端主动断开了。

一点特别重要的发送心跳到后端，后端收到消息之后必须返回消息，否则超过60秒之后会判定后端主动断开了。再改造下代码:

var heartCheck = {
    timeout: 60000,//60ms
    timeoutObj: null,
    serverTimeoutObj: null,
    reset: function(){
        clearTimeout(this.timeoutObj);
        clearTimeout(this.serverTimeoutObj);
　　　　 this.start();
    },
    start: function(){
        var self = this;
        this.timeoutObj = setTimeout(function(){
            ws.send("HeartBeat");
            self.serverTimeoutObj = setTimeout(function(){
                ws.close();//如果onclose会执行reconnect，我们执行ws.close()就行了.如果直接执行reconnect 会触发onclose导致重连两次
            }, self.timeout)
        }, this.timeout)
    },
}

ws.onopen = function () {
   heartCheck.start();
};
ws.onmessage = function (event) {
    heartCheck.reset();
}
ws.onclose = function () {
    reconnect();
};
ws.onerror = function () {
    reconnect();
};
 
PS：

    因为目前我们这种方式会一直重连如果没连接上或者断连的话，如果有两个设备同时登陆并且会踢另一端下线，一定要发送一个踢下线的消息类型，这边接收到这种类型的消息，逻辑判断后就不再执行reconnect，否则会出现一只相互挤下线的死循环。
