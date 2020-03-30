# Interactive Canvas

## Interactive Canvas是什么

基于Google Assistant的一种框架,可以构建一个既有语音会有,又有视觉交互的全屏Web App

![](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-11-20-interactivecanvasgame.gif)

## 怎么运行的

![](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-11-20-091654.jpg)

1. 对Google Assistant设备说话
2. 会把请求发送到Dialogflow,然后根据Fulfillment匹配意图,然后返回一个HtmlResponse,唤醒这个web app,第一次HtmlResponse通常会携带一个URL参数.
3. 这个URL就是这个web app的URL,然后Google Assistant会加载这个URL,然后界面就显示我们开发web app.
4. 这个web app被加载的时候会通过Interactive Canvas API注册callback.这个callback用来处理HtmlResponse
5. web app会根据HtmlResponse携带的data,绘制相应的东西.

### 怎么实现Interactive Canvas Web App

1. 导入相关的js文件

   ```javascript
    <!-- Load Interactive Canvas JavaScript -->
       <script src="https://www.gstatic.com/assistant/interactivecanvas/api/interactive_canvas.min.js"></script>
       <!-- Load PixiJS for graphics rendering -->
       <script src="https://cdnjs.cloudflare.com/ajax/libs/pixi.js/4.8.7/pixi.min.js"></script>
       <!-- Load Stats.js for fps monitoring -->
       <script src="https://cdnjs.cloudflare.com/ajax/libs/stats.js/r16/Stats.min.js"></script>
   ```

2. 定义注册callback

3. 根据需求定义相应意图的方法

4. 在Html初始话的时候注册回调

   ```javascript
   'use strict';
   
   class Action {
     constructor(scene) {
       this.canvas = window.interactiveCanvas;
       this.scene = scene;
       const that = this;
       this.commands = {
         TINT: function(data) {
           that.scene.sprite.tint = data.tint;
         },
         SPIN: function(data) {
           that.scene.sprite.spin = data.spin;
         },
         RESTART_GAME: function(data) {
           that.scene.button.texture = that.scene.button.textureButton;
           that.scene.sprite.spin = true;
           that.scene.sprite.tint = 0x0000FF; // blue
           that.scene.sprite.rotation = 0;
         },
       };
     }
   
     /**
      * Register all callbacks used by Interactive Canvas
      * executed during scene creation time.
      *
      */
     setCallbacks() {
       const that = this;
       // declare interactive canvas callbacks
       const callbacks = {
         onUpdate(data) {
           try {
             that.commands[data.command.toUpperCase()](data);
           } catch (e) {
             // do nothing, when no command is sent or found
           }
         },
       };
       // called by the Interactive Canvas web app once web app has loaded to
       // register callbacks
       this.canvas.ready(callbacks);
     }
   }
   
   ```

   ```javascript
   window.onload = () => {
     this.scene = new Scene();
     // Set Google Assistant Canvas Action at scene level
     this.scene.action = new Action(scene);
     // Call setCallbacks to register interactive canvas
     this.scene.action.setCallbacks();
   };
   ```

   

结论

- Interactive Canvas  Web App和普通的Web App有什么不一样?

  Interactive Canvas  Web App就是集成了Interactive Canvas API的Web App,它的交互侧重于语音交互,当然也可以实现触摸点击的交互

- Interactive Canvas  Web APP是什么?

  既然于普通的Web App没什么区别,它的容器也是webview.

  拿到Interactive Canvas Web App的URL,在浏览器中也能打开它.