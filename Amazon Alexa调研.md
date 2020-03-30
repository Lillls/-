# Amazon Alexa

## Alexa是什么？

Alexa是亚马逊基于云的语音服务，在亚马逊和第三方设备制造商的超过1亿台设备上使用。借助Alexa，可以构建自然的语音体验.

**Alexa的设计宗旨是响应许多不同的命令，甚至可以与用户对话。**



## 能用Alexa来做什么？

- ##### 创建Alexa技能 Alexa Skills

  通过培养Alexa技能，吸引全球超过1亿个支持Alexa设备的客户。技能就像Alexa的应用程序一样，使客户能够执行日常任务或通过语音自然地与您的内容互动。

- ##### 构建Alexa设备  Alexa Devices

  通过将Alexa集成到设备中或使用Alexa控制设备，来构建语音转发产品。




简单来说 集成了Alexa的设备或者硬件就像一部手机。Alexa技能就像手机里的App。有了手机之后可以通过App来做不同事。不同的是：

- 手机控制App,是通过触摸来发送指令，通过视觉效果来获取反馈。
- Alexa, 是通过语音来分析要使用那些Skills,然后给用户一个反馈。

## 怎么做？

- Alexa Skills

  1. 通过填写模板   Fill in a Template

     ![](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-08-090827.png)

     Alexa开发者有一些默认的模板，可以通过填写模板来创建一个简单的Skill.

     例如这个 Whose Turn （轮到谁了）。

     1. 添加名字

     ![image-20191108171059495](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-08-091103.png)

     

     

     2. 最后设置唤醒这个Skill的名字

        ![image-20191108171429747](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-08-091433.png)

        然后就可以通过 Who's next来唤醒这个Skill了

  2. 完全自定义  `

     主要分为以下几个步骤：

     ![image-20191108172508858](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-08-092509.png)

     当1，2，3创建好之后需要配置好处理Skill意图的服务端，可以选择AWS和其他的服务器都可以。

     第5步，可以选择Skill内产品，跟GP和AppStore类型差不多

     ![image-20191108173502242](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-08-093502.png)

     最后认证和发布之后，用户就可以使用这个Skill了。

- Alexa Devices

  重要的SDK组件

  ![image-20191108174837261](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-11-055213.png)

参考

https://github.com/alexa/avs-device-sdk/wiki/macOS-Quick-Start-Guide

https://developer.amazon.com/en-US/alexa/alexa-skills-kit