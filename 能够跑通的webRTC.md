# 端对端连接基本流程图示

![img](https://img-blog.csdnimg.cn/20200802011057863.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3h5cGhm,size_16,color_FFFFFF,t_70)







![webrtc_p2p_squence](https://img-blog.csdnimg.cn/20200618134458727.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FnZ3Jlc3Nz,size_16,color_FFFFFF,t_70)



![img](https://img2018.cnblogs.com/blog/27612/201907/27612-20190707160212778-1946469521.png)

# 端对端连接流程文字描述

1. 呼叫者通过 [`navigator.mediaDevices.getUserMedia()`](https://developer.mozilla.org/zh-CN/docs/Web/API/Navigator/mediaDevices/getUserMedia) 捕捉本地媒体。
2. 呼叫者创建一个`RTCPeerConnection` 并调用 [`RTCPeerConnection.addTrack()`](https://developer.mozilla.org/zh-CN/docs/Web/API/RTCPeerConnection/addTrack)(注： `addStream` 已经过时。)
3. 呼叫者调用 ("RTCPeerConnection.createOffer()")来创建一个提议(offer).
4. 呼叫者调用 ("RTCPeerConnection.setLocalDescription()") 将提议(Offer)  设置为本地描述 (即，连接的本地描述).
5. setLocalDescription()之后, 呼叫者请求 STUN 服务创建ice候选(ice candidates)。（机制可参考该文档：[RTCPeerConnection: icecandidate event](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/icecandidate_event)）
6. 呼叫者通过信令服务器将提议(offer)传递至 本次呼叫的预期的接受者.
7. 接受者收到了提议(offer) 并调用 ("RTCPeerConnection.setRemoteDescription()") 将其记录为远程描述 (也就是连接的另一端的描述).
8. 接受者做一些可能需要的步骤结束本次呼叫：捕获本地媒体，然后通过[`RTCPeerConnection.addTrack()`](https://developer.mozilla.org/zh-CN/docs/Web/API/RTCPeerConnection/addTrack)添加到连接中。
9. 接受者通过("RTCPeerConnection.createAnswer()")创建一个应答。
10. 接受者调用 ("RTCPeerConnection.setLocalDescription()") 将应答(answer)  设置为本地描述. 此时，接受者已经获知连接双方的配置了.
11. 接受者通过信令服务器将应答传递到呼叫者.
12. 呼叫者接受到应答.
13. 呼叫者调用 ("RTCPeerConnection.setRemoteDescription()") 将应答设定为远程描述. 如此，呼叫者已经获知连接双方的配置了.

附录：

[RTCPeerConnection常用方法](https://developer.mozilla.org/zh-CN/docs/Web/API/RTCPeerConnection#%E6%96%B9%E6%B3%95_2)

[webRTC文档](https://developer.mozilla.org/zh-CN/docs/Web/API/WebRTC_API)

一些有用的记录：

[`RTCPeerConnection.onnegotiationneeded`](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/onnegotiationneeded)

This function is called whenever the WebRTC infrastructure needs you to start the session negotiation process anew. Its job is to create and send an offer, to the callee, asking it to connect with us. See [Starting negotiation](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Signaling_and_video_calling#starting_negotiation) to see how we handle this.

# server端

```javascript
//@author :  bilibili:一只斌  /   mail: tuduweb@qq.com
var express = require('express');
var app = express();
var http = require('http').createServer(app);

var fs = require('fs');
let sslOptions = {
    key: fs.readFileSync('C:/privkey.key'),//里面的文件替换成你生成的私钥
    cert: fs.readFileSync('C:/cacert.pem')//里面的文件替换成你生成的证书
};

const https = require('https').createServer(sslOptions, app);

var io = require('socket.io')(https);

//var path = require('path');
//app.use(express.static(path.join(__dirname, 'public')));


app.get('/', (req, res) => {
    res.sendFile(__dirname + '/index.html');
  });

app.get('/camera', (req, res) => {
    res.sendFile(__dirname + '/camera.html');
});


io.on("connection", (socket) => {
    //连接加入子房间
    socket.join( socket.id );

    console.log("a user connected " + socket.id);

    socket.on("disconnect", () => {
        console.log("user disconnected: " + socket.id);
        //某个用户断开连接的时候，我们需要告诉所有还在线的用户这个信息
        socket.broadcast.emit('user disconnected', socket.id);
    });

    socket.on("chat message",(msg) => {
        console.log(socket.id + " say: " + msg);
        //io.emit("chat message", msg);
        socket.broadcast.emit("chat message", msg);
    });

    //当有新用户加入，打招呼时，需要转发消息到所有在线用户。
    socket.on('new user greet', (data) => {
        console.log(data);
        console.log(socket.id + ' greet ' + data.msg);
        socket.broadcast.emit('need connect', {sender: socket.id, msg : data.msg});
    });
    //在线用户回应新用户消息的转发
    socket.on('ok we connect', (data) => {
        io.to(data.receiver).emit('ok we connect', {sender : data.sender});
    });

    //sdp 消息的转发
    socket.on( 'sdp', ( data ) => {
        console.log('sdp');
        console.log(data.description);
        //console.log('sdp:  ' + data.sender + '   to:' + data.to);
        socket.to( data.to ).emit( 'sdp', {
            description: data.description,
            sender: data.sender
        } );
    } );

    //candidates 消息的转发
    socket.on( 'ice candidates', ( data ) => {
        console.log('ice candidates:  ');
        console.log(data);
        socket.to( data.to ).emit( 'ice candidates', {
            candidate: data.candidate,
            sender: data.sender
        } );
    } );

});


https.listen(443, () => {
    console.log('https listening on *:443');
});
```



# index页面

```html
<!doctype html>
<html>
  <head>
    <title>Socket.IO chat</title>
    <style>
      * { margin: 0; padding: 0; box-sizing: border-box; }
      body { font: 13px Helvetica, Arial; }
      form { background: #000; padding: 3px; position: fixed; bottom: 0; width: 100%; }
      form input { border: 0; padding: 10px; width: 90%; margin-right: 0.5%; }
      form button { width: 9%; background: rgb(130, 224, 255); border: none; padding: 10px; }
      #messages { list-style-type: none; margin: 0; padding: 0; }
      #messages li { padding: 5px 10px; }
      #messages li:nth-child(odd) { background: #eee; }
    </style>
  </head>
  <body>
    <ul id="messages"></ul>
    <form action="">
      <input id="m" autocomplete="off" /><button>Send</button>
    </form>
  </body>
  <script src="//cdn.bootcdn.net/ajax/libs/jquery/3.4.1/jquery.js"></script>
  <script src="//cdn.bootcdn.net/ajax/libs/socket.io/3.0.4/socket.io.js"></script>
  
  <script>
      //当页面准备完成时, $是jQuery的一个函数
      $(document).ready(function(){
          //令 socket 等于 socket.io 对象，io() 中的参数为socket.io服务器地址，在这里和web服务器相同。
          var socket = io('//');
          //在控制台输出hello字符串 console.log 和print类似。
          console.log('hello');
          //监听connect事件，当socket连接建立时，会触发后面第二个参数中的匿名函数内容
          socket.on('connect', () => {
              //输出内容 其中 socket.id 是当前socket连接的唯一ID
              console.log('connect ' + socket.id);
          });

        //监听form的提交操作
        $('form').submit(function(e) {
            //禁止页面重新加载
            e.preventDefault(); //prevents page reloading
            //发送事件，其值为文本框中输入的值
            socket.emit('chat message', $('#m').val());
            //清空文本框的值
            $('#m').val('');
            //返回false 禁止原始的提交
            return false;
        });

        //监听 chat message事件，当监听到事件发生时执行第二个参数中的匿名函数
        socket.on('chat message', function(msg){
            //在网页中id为messages的对象中，插入li标签，其内容为msg
            $('#messages').append($('<li>').text(msg));
        });


      });
  </script>
</html>
```

# camera端

```html
<!doctype html>
<html>
<!-- @author :  bilibili:一只斌  /   mail: tuduweb@qq.com-->
<head>
    <title>Camera</title>
    <style>
        #user-list>li {
            font-size: 24px;
        }
    </style>
</head>

<body>
    <h1 id="user-id">用户名称</h1>
    <ul id="user-list">
        <li>用户12</li>
        <li>用户23</li>
        <li>用户34</li>
    </ul>
    <video id="video-local" controls autoplay></video>

    <canvas id="capture-canvas" style="display: none;"></canvas>

    <button id="capture">拍照</button>

    <ul id="capture-list"></ul>


    <div id="videos"></div>

    <script src="//cdn.bootcdn.net/ajax/libs/socket.io/3.0.4/socket.io.js"></script>
    <script src="//cdn.bootcdn.net/ajax/libs/jquery/3.4.1/jquery.js"></script>
    <script>
        //封装一部分函数
        function getUserMedia(constrains, success, error) {
            if (navigator.mediaDevices.getUserMedia) {
                //最新标准API
                promise = navigator.mediaDevices.getUserMedia(constrains).then(success).catch(error);
            } else if (navigator.webkitGetUserMedia) {
                //webkit内核浏览器
                promise = navigator.webkitGetUserMedia(constrains).then(success).catch(error);
            } else if (navigator.mozGetUserMedia) {
                //Firefox浏览器
                promise = navagator.mozGetUserMedia(constrains).then(success).catch(error);
            } else if (navigator.getUserMedia) {
                //旧版API
                promise = navigator.getUserMedia(constrains).then(success).catch(error);
            }
        }

        function canGetUserMediaUse() {
            return !!(navigator.mediaDevices.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia || navigator.msGetUserMedia);
        }


        const localVideoElm = document.getElementById("video-local");
        $('document').ready(() => {


            $('#capture').click(() => {
                let video = localVideoElm//原生dom
                let isPlaying = !(video.paused || video.ended || video.seeking || video.readyState < video.HAVE_FUTURE_DATA)

                if (isPlaying) {
                    let canvas = $('#capture-canvas')
                    canvas.attr('width', localVideoElm.clientWidth);//设置canvas的宽度
                    canvas.attr('height', localVideoElm.clientHeight);//设置canvas的高度

                    let img = $('<img>')
                    img.attr('width', localVideoElm.clientWidth);//设置图像的宽度
                    img.attr('height', localVideoElm.clientHeight);//设置图像的高度

                    //canvas[0] //jQuery对象转dom
                    var context = canvas[0].getContext('2d');
                    //在canvas上绘图，其绘图坐标为0,0; 
                    //绘图大小为摄像头内容的宽度，高度（全局绘制，你可以改变这些值试试效果）。
                    context.drawImage(localVideoElm, 0, 0, localVideoElm.clientWidth, localVideoElm.clientHeight);
                    //根据canvas内容进行编码，并赋值到图片上
                    var data = canvas[0].toDataURL('image/png');
                    img.attr('src', data);
                    //插入到id为capture-list的有序列表里
                    $('#capture-list').append($('<li></li>').html(img));
                }
            })

        });

        //STUN,TURN服务器配置参数
        const iceServer = {
            iceServers: [{ urls: ["stun:ss-turn1.xirsys.com"] }, { username: "CEqIDkX5f51sbm7-pXxJVXePoMk_WB7w2J5eu0Bd00YpiONHlLHrwSb7hRMDDrqGAAAAAF_OT9V0dWR1d2Vi", credential: "446118be-38a4-11eb-9ece-0242ac140004", urls: ["turn:ss-turn1.xirsys.com:80?transport=udp", "turn:ss-turn1.xirsys.com:3478?transport=udp"] }]
        };
        //自己的coturn服务器
        const iceServer = {
        iceServers: [{
            'urls':'turn:106.54.87.109:3478',
            'credential':'123456',
            'username':'test'
        }]
    };

        //PeerConnection
        var pc = [];
        var localStream = null;

        function InitCamera() {

            if (canGetUserMediaUse()) {
                getUserMedia({
                    video: true,
                    audio: false
                }, (stream) => {
                    localStream = stream;
                    localVideoElm.srcObject = stream;
                    $(localVideoElm).width(800);
                }, (err) => {
                    console.log('访问用户媒体失败: ', err.name, err.message);
                });
            } else {
                alert('您的浏览器不兼容');
            }

        }

        function StartCall(parterName, createOffer) {

            pc[parterName] = new RTCPeerConnection(iceServer);

            //如果已经有本地流，那么直接获取Tracks并调用addTrack添加到RTC对象中。
            if (localStream) {

                localStream.getTracks().forEach((track) => {
                    pc[parterName].addTrack(track, localStream);//should trigger negotiationneeded event
                });

            }else{
                //否则需要重新启动摄像头并获取
                if (canGetUserMediaUse()) {
                    getUserMedia({
                        video: true,
                        audio: false
                    }, function (stream) {
                        localStream = stream;

                        localVideoElm.srcObject = stream;
                        $(localVideoElm).width(800);

                    }, function (error) {
                        console.log("访问用户媒体设备失败：", error.name, error.message);
                    })
                } else { alert('您的浏览器不兼容'); }

            }

            //如果是呼叫方,那么需要createOffer请求
            if (createOffer) {
                //每当WebRTC基础结构需要你重新启动会话协商过程时，都会调用此函数。它的工作是创建和发送一个请求，给被叫方，要求它与我们联系。
                pc[parterName].onnegotiationneeded = () => {
                    //https://developer.mozilla.org/zh-CN/docs/Web/API/RTCPeerConnection/createOffer

                    pc[parterName].createOffer().then((offer) => {
                        return pc[parterName].setLocalDescription(offer);
                    }).then(() => {
                        //把发起者的描述信息通过Signal Server发送到接收者
                        socket.emit('sdp', {
                            type: 'video-offer',
                            description: pc[parterName].localDescription,
                            to: parterName,
                            sender: socket.id
                        });
                    })
                };
            }

            //当需要你通过信令服务器将一个ICE候选发送给另一个对等端时，本地ICE层将会调用你的 icecandidate 事件处理程序。有关更多信息，请参阅Sending ICE candidates 以查看此示例的代码。
            pc[parterName].onicecandidate = ({ candidate }) => {
                socket.emit('ice candidates', {
                    candidate: candidate,
                    to: parterName,
                    sender: socket.id
                });
            };

            //当向连接中添加磁道时，track 事件的此处理程序由本地WebRTC层调用。例如，可以将传入媒体连接到元素以显示它。详见 Receiving new streams 。
            pc[parterName].ontrack = (ev) => {
                let str = ev.streams[0];

                if (document.getElementById(`${parterName}-video`)) {
                    document.getElementById(`${parterName}-video`).srcObject = str;
                } else {
                    let newVideo = document.createElement('video');
                    newVideo.id = `${parterName}-video`;
                    newVideo.autoplay = true;
                    newVideo.controls = true;
                    //newVideo.className = 'remote-video';
                    newVideo.srcObject = str;

                    document.getElementById('videos').appendChild(newVideo);
                }
            }



        }

        var socket = io();

        socket.on('connect', () => {
            InitCamera();

            //输出内容 其中 socket.id 是当前socket连接的唯一ID
            console.log('connect ' + socket.id);

            $('#user-id').text(socket.id);

            pc.push(socket.id);

            socket.emit('new user greet', {
                sender: socket.id,
                msg: 'hello world'
            });

            socket.on('need connect', (data) => {

                console.log(data);
                //创建新的li并添加到用户列表中
                let li = $('<li></li>').text(data.sender).attr('user-id', data.sender);
                $('#user-list').append(li);
                //同时创建一个按钮
                let button = $('<button class="call">通话</button>');
                button.appendTo(li);
                //监听按钮的点击事件, 这是个demo 需要添加很多东西，比如不能重复拨打已经连接的用户
                $(button).click(function () {
                    //$(this).parent().attr('user-id')
                    console.log($(this).parent().attr('user-id'));
                    //点击时，开启对该用户的通话
                    StartCall($(this).parent().attr('user-id'), true);
                });

                socket.emit('ok we connect', { receiver: data.sender, sender: socket.id });
            });
            //某个用户失去连接时，我们需要获取到这个信息
            socket.on('user disconnected', (socket_id) => {
                console.log('disconnect : ' + socket_id);

                $('#user-list li[user-id="' + socket_id + '"]').remove();
            })
            //链接吧..
            socket.on('ok we connect', (data) => {
                console.log(data);

                $('#user-list').append($('<li></li>').text(data.sender).attr('user-id', data.sender));
                //这里少了程序，比如之前的按钮啊，按钮的点击监听都没有。
            });

            //监听发送的sdp事件
            socket.on('sdp', (data) => {
                //如果时offer类型的sdp
                if (data.description.type === 'offer') {
                    //那么被呼叫者需要开启RTC的一套流程，同时不需要createOffer，所以第二个参数为false
                    StartCall(data.sender, false);
                    //把发送者(offer)的描述，存储在接收者的remoteDesc中。
                    let desc = new RTCSessionDescription(data.description);
                    //按1-13流程走的
                    pc[data.sender].setRemoteDescription(desc).then(() => {

                        pc[data.sender].createAnswer().then((answer) => {
                            return pc[data.sender].setLocalDescription(answer);
                        }).then(() => {
                            socket.emit('sdp', {
                                type: 'video-answer',
                                description: pc[data.sender].localDescription,
                                to: data.sender,
                                sender: socket.id
                            });

                        }).catch();//catch error function empty

                    })
                } else if (data.description.type === 'answer') {
                    //如果使应答类消息（那么接收到这个事件的是呼叫者）
                    let desc = new RTCSessionDescription(data.description);
                    pc[data.sender].setRemoteDescription(desc);
                }
            })

            //如果是ice candidates的协商信息
            socket.on('ice candidates', (data) => {
                console.log('ice candidate: ' + data.candidate);
                //{ candidate: candidate, to: partnerName, sender: socketID }
                //如果ice candidate非空（当candidate为空时，那么本次协商流程到此结束了）
                if (data.candidate) {
                    var candidate = new RTCIceCandidate(data.candidate);
                    //讲对方发来的协商信息保存
                    pc[data.sender].addIceCandidate(candidate).catch();//catch err function empty
                }
            })


        });


    </script>
</body>


</html>
```

