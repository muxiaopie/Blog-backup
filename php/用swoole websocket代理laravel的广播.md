# 用swoole websocket代理laravel的广播



​	其实这篇文章应该4月份就应该写的，但是囿于懒惰一直拖到现在，直到今天有个朋友问起swoole的websocket，而且恰好最近在写一个接口通信是全部基于websocket的spa后台的开源项目。我才想起这篇悬而未决的文章。

​         即时通行对于现在web开发来说必不可少。比如一些实时消息和广播。其实laravel框架自己带有基于事件的广播系统，但是如果要求系统的实时性的话，需要利用一些其他的服务或者框架，比如官方推荐的pusher，虽然简单方便好用，但免费版本还有有一些限制。再如laravel提供的laravel-echo-server其实是用nodejs写的，这其实提高了项目的运维成本。php作为一个非常驻内存的语言，写起websocket其实挺蛋疼的，所以我上面那个开源项目是基于python写的。因为写起来方便快捷。好在php有swoole这样的一个重新定义了php的开源项目。

​	laravel的广播其实基于事件的，我们可以先参考官方文档，你必须先注册必须先注册 `App\Providers\BroadcastServiceProvider` ，然后定义一个事件定义一个事件，比如我定义了一个message的事件，让其继承`ShouldBroadcast`接口。源码如下

```php
<?php

namespace App\Events;
use Illuminate\Broadcasting\Channel;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

class Message implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    public $data;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct(array $data)
    {
        $this->data = $data;
    }
    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new Channel('message');
    }
}
```

​	其实官方默认的广播驱动式log，比如我们可以尝试一下。

```php
Route::get('/', function () {
    $array = [
        'name' => "木小撇",
    ];
    event(new \App\Events\Message($array));
    return view('welcome');
});
```



​	这时候我们刷新页面，然后打开你的日志文件。这时候不能发现你的记录

```
[2018-06-08 14:53:44] local.INFO: Broadcasting [App\Events\Message] on channels [message] with payload:
{
    "data": {
        "name": "\u6728\u5c0f\u6487"
    },
    "socket": null
}  
```

​	当然了，我们最终要实现的还是要用其他的驱动，比如官方推荐的pusher，真的是简单好用。那么我们尝试着将驱动换成redis试试。进入redis-cli然后订阅所有频道的消息 ，然后刷新页面。我们不难发现屏幕上打印这样的信息

```
127.0.0.1:6379> PSUBSCRIBE *
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "*"
3) (integer) 1
1) "pmessage"
2) "*"
3) "message"
4) "{\"event\":\"App\\\\Events\\\\Message\",\"data\":{\"data\":{\"name\":\"\\u6728\\u5c0f\\u6487\"},\"socket\":null},\"socket\":null}"
```

​	

​	既然这样我们其实可以利用swoole的websocket来监听这个发布订阅，然后广播出去就好了。比如我们可以定义一个laravel的Command。

​	然后在handle函数里面定义swoole_websocket_server的初始化设置以及回调函数。

```php
public function handle()
    {
        $this->server = new \swoole_websocket_server('0.0.0.0', 9527);
         $this->server->set([
//             'daemonize' => true,
             'worker_num'   => 2,
             'task_worker_num'  =>  10,
         ]);
        // 设置回调函数
        $this->server->on("Start",[$this,'onStart']);
        $this->server->on('WorkerStart',[$this,'onWorkerStart']);
        $this->server->on("Task",[$this,'onTask']);
        $this->server->on('Finish',[$this,'onFinish']);
        $this->server->on('Open', [$this, 'onOpen']);
        $this->server->on('Message', [$this, 'onMessage']);
        $this->server->on('Close', [$this, 'onClose']);
        $this->server->start();
    }
```



​	这里多补充几句，swoole_websocket_server是基于swoole_server的所以在里面依然可以使用worker和task_worker进程的，这样的话我们一样可以把耗时比较长的逻辑投递到task_worker进程里面去。

​	我们在前端定义一个视图，写我们的js代码。

```javascript
var  ws = new WebSocket('ws://192.168.10.10:9527?token=hello');
ws.onopen = function (evt) { onOpen(evt) };
ws.onclose = function (evt) { onClose(evt) };
ws.onmessage = function (evt) { onMessage(evt) };
ws.onerror = function (evt) { onError(evt) };

function onOpen(evt) {
    var message = '{"channel": "message", "event": "Message"}';
    ws.send(message);
}
function onClose(evt) {
    console.log("Disconnected");
}
function onMessage(evt) {
    var data = JSON.parse(evt.data);
    console.log(data);
    console.log('Retrieved data from server: ' + evt.data);
}
function onError(evt) {
    console.log('Error occured: ' + evt.data);
}
```



​	这里再多补充一句，websocket也支持get传参，我们可以再ws的url后面拼上我们我的需要传递的参数，比如token啊用户Id神马的。

​	前端每次连接websocket的时候都会传递一个channel过来，那么我们服务端就很好处理了。我们可以尝试这读取这个channel，然后去监听这个发布订阅。

```php
// 收到数据时回调函数
public function onMessage(\swoole_websocket_server $server, \swoole_websocket_frame $frame)
{
    $data = $frame->data;
    $data = json_decode($data,true);
    $data['fd'] = $frame->fd;
    if(isset($data['channel'])){
        $this->server->task($data);
    }
}
```



​	其实这里可以看到 ，我将任务投递到了task进程里面去了，然后我们就在onTask里面书写我们的逻辑。

```
public function onTask(\swoole_websocket_server $server, $task_id, $src_worker_id, $data)
    {
        if (isset($data['channel'])) {
            $channel = $data['channel'];
            Redis::psubscribe([$channel], function($message,$chan) use ($server,$data) {
                foreach ($server->connections as $key => $value) {
                    if($data['fd'] != $value){
                        $server->push($value, $message);
                    }
                }
            });
        }
    }
```

​	这里将遍历所有的连接然后将广播push出去。这样一来一个简单的广播就实现了。

​	我们可以启动一下swoole_websocket_server看看。

​	然后我们刷新页面触发事件，不难再websocket的客户端的console里面发现一行信息。

```{event: "App\Events\Message", data: {…}, socket: null}
master:47 Retrieved data from server: {"event":"App\Events\Message","data":{"data":{"name":"\u6728\u5c0f\u6487"},"socket":null},"socket":null}
```

​	好了，一个简单的swoole_websocket_server代理laravel的广播demo就这样完成了，当然这仅仅只是一个简单尝试，其实里面还有一些坑和尚未解决的问题。等待着你们发现，如有问题别来找我。

​	反正我写websocket从来不用php。pythoner了解下。

