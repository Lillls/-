# Mac配置oh my zsh,并使用ITerm2

先看看最终效果

![image-20191115151234824](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-11-15-071235.png)

1. 第一步安装on my zsh

   ```shell
   sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
   ```

2. 切换为zsh

   ```shell
    sudo chsh -s /bin/zsh
   ```

3. 更换主题

   在命令行用vim打开用户根目录下.zshrc，我的修改为agnoster：

   ```
   ZSH_THEME="agnoster"
   ```

4. 安装power-line字体

   ```shell
   # clone
   git clone https://github.com/powerline/fonts.git --depth=1
   # install
   cd fonts
   ./install.sh
   # clean-up a bit
   cd ..
   rm -rf fonts
   ```

5. 设置字体高亮

   ```shell
   brew install zsh-syntax-highlighting
   ```

   打开zshrc配置文件追加

   ```shell
   open ~/.zshrc
   ```

   ```bash
   ##关键字高亮
   source $ZSH/oh-my-zsh.sh
   source /usr/local/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
   #很多教程只有上边那两行，但是我们设置好编译后发现关键字还是没有高亮，这是因为iTerm终端自身的原因，加上后面两行代码就可以将zsh中主题的颜色加载出来了
   export CLICOLOR=1
   export TERM=xterm-256color
   ```

6. 下载ITeam2

   自己去官网下载

7. 更换颜色

   打开ITerm2的设置页面-Profiles-colors

   选择**Solarized-Dark**

   ![image-20191115152142935](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-11-15-072143.png)

8. 更换字体

   打开ITerm2的设置页面-Profiles-Text

   选择**Meslo Lg L for PowerLine**

   ![image-20191115152304240](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-11-15-072304.png)

   

### 去掉终端前的用户名

![截屏2019-11-15下午3.59.39](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-11-15-080032.png)

```shell
open ~/.zshrc
```

```
# redefine prompt_context for hiding user@hostname
prompt_context () { }
```

```
source ~/.zshrc
```

![截屏2019-11-15下午4.01.49](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-11-15-080201.png)

​	

