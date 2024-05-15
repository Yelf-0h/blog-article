# 部署Stable diffusion（生图）教程

## 1、背景

[Stable Diffusion 万字长文详解稳定扩散模型](https://zhuanlan.zhihu.com/p/669570827) <===对模型的介绍

部署Stable Diffusion主要是用来靠模型生成高质量图片

当前比较出名的还有一个AI生图工具就是Midjourney

## 2、对比

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/5b0235bef375e70a9e8d9b6e766e0d6c.png)

相比之下就是免费，缺点是各种生成模型需要自身去找或者训练

## 3、实践

这边可以选择各种云主机，但是我的这个是基于AutoDL AI算力云

### 3.1、注册登录

[AutoDL AI算力云](https://www.autodl.com/market/list)  <== 官网链接

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/8dec7feacfb102e55598ee6d199fece8.png)

### 3.2、购买实例

首先先充值金额大于10元

然后去抢机子的时候到了，机子现在基本没有，需要一直刷新才能抢到，而且为什么需要先充值呢，因为到时候机子在选择过程中需要付费，就要去充值，等充值完回来，机子又没了

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/e95ce4fc640b8a3b59ae2ceadcd5fc2b.png)
选择按量付费，社区镜像选择

搜索stable-diffusion-webui 有界面的镜像，版本选择最高就行，不过新版本没有官方教程，但是最新版本一键启动
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/05662494e3c77c04a90eee38e36a2ca0.png)

然后点击自己的实例，选择JupyterLab
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/0d48a5b45e4e8c9033f634f3553debc5.png)
进入到界面后，选择第一段代码，然后点击小三角形运行
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/333f0bbc4b7351f166b6ccb21517fec5.png)
等运行完毕后，会出现以下界面
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/023eff53535a15fc9da8124ae16a69cf.png)
这个界面的设置可以选择下载模型
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/168e33ea2ff8a1610010b6d6581fdcb3.png)
这时候点击启动
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/a49a20064c7e7c33e35a152aaf4cb5a1.png)
等运行完毕
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/665a3b117c8eb3f3ffdff594e0aebbbb.png)
运行完毕后点击自定义服务 就能进行使用了
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/6daf171b3bcc3ed0b52dd0f4ecf3d919.png)
复制来一些关键词来测试
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/9b2a9634dd1c9a1e1f7d20198af826e4.png)
