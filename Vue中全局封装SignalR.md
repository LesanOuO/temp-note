## SignalR 简介
ASP.NET SignalR 是一个供 ASP.NET 开发人员使用的库，它简化了向应用程序添加实时 Web 功能的过程。实时 Web 功能是让服务器代码在内容可用时立即将内容推送到连接的客户端的能力，而不是让服务器等待客户端请求新数据。

[官方网址](https://docs.microsoft.com/en-us/aspnet/signalr/)

## 实现步骤

### 1. 引入SignalR官方库

```bash
npm install @microsoft/signalr
```

### 2. 创建一个signalR.js文件

```javascript
const signalR = require("@microsoft/signalr")

export default {
    SR: {},
    //初始化连接
    initSR: function(domain) {
        let that = this;
        domain = domain === "localhost" ? "内网IP" : domain;
        let url = `http://${domain}:8000/chatHub`;
        that.SR = new signalR.HubConnectionBuilder()
            .withUrl(url)
            .configureLogging(signalR.LogLevel.Information)
            .build();
        async function start() {
            try {
                await that.SR.start();
                console.log("signaR连接成功");
            } catch (err) {
                console.log("err", err);
                setTimeout(start, 5000);
            }
        }
        that.SR.onclose(async() => {
            await start();
        });
        start();
    },
    syncPage: function(func) {
        this.SR.on("ReceiveMessageFromGroup", function(group, message) {
            func();
        });
    },
    addToGroup: function(group) {
        this.SR.invoke("AddToGroup", group).catch(function(err) {
            return console.error(err.toString());
        });
    },
    removeFromGroup: function(group) {
        this.SR.off("ReceiveMessageFromGroup")
        this.SR.invoke("RemoveFromGroup", group).catch(function(err) {
            return console.error(err.toString());
        });
    },
    // 停止连接
    stopSR: function() {
        let that = this;
        async function stop() {
            try {
                await that.SR.stop();
                console.log("signaR退出成功");
            } catch (err) {}
        }
        stop();
    },
};
```

### 3. 在main.js中引入并全局挂载

```javascript
import signalr from "signaR的路径";
Vue.prototype.signalr = signalr;
```

### 4. 初始化连接
- 可在登录后进行初始化`this.signalr.initSR(document.Domain);`
- 由于页面刷新后，全局挂载的signalR会消失，所以需要在App.vue中再初始化一遍
  `mounted() {this.signalr.initSR(document.Domain);},`

### 5. 页面中使用

```javascript
this.signalr.SR.on('方法', function (data) {
    // 接收后要做的事
    console.log('方法', data)
})
```

**特别提醒！！！**
当页面切换时，需要销毁注册的方法，不然会导致重复注册方法，会产生多次调用注册的方法，需要再离开页面时清空方法。

```javascript
destroyed() {
    // 我使用的方法
    this.signalr.SR.off("方法")

    // 网上的方法，但是我使用无效
    this.signalr.SR.methods.方法 = []
}
```
