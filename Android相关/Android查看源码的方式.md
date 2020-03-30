# Android查看源码的方式

1. 在Android Studio中，选中要看的版本，点击OK，Android Studio会自动下载我们需要的源码。

![image-20191111172159482](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-11-092159.png)

2. 下载Android Studio不存在的源码

   从https://android.googlesource.com/platform/libcore/,克隆需要的源码

   ![image-20191111172442753](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-11-092443.png)

   默认是master分支，git切换与build.gradle compileSdkVersion相同的tag。

   然后将文件夹复制到sdk-source的文件夹下。

   例如：dalvik.system.PathClassLoader  看不到源码![image-20191111172746656](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-11-092747.png)

   进入到git clone的文件夹内，进入到对应的文件夹下，复制的时候不要复制最外层的dalvik文件夹，要复制最里层的dalvik文件夹，然后复制到sources，对应的compileSdkVersion中

   ![image-20191111173208697](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-11-093209.png)

   ![image-20191111173052407](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/blog/2019-11-11-093053.png)