# Typora + Github + PicGo 的 markdown 免费图床配置

[toc]

## 前言

之前用的新浪的图床，挂了。考虑了下还是用 github 靠谱点。结果配置后一直在 Typora 中提示 load image failed ，遂简单记录一下。
Typora 一个 markdown 编辑神 App 。
PicGo 是一个用于快速上传图片并获取图片 URL 链接的工具。支持七牛云、腾讯云GitHub、阿里云OSS 等。通过 PicGo 将图片上传到这些图床，获得一个在线 URL 从而在网页、博客或 markdown 文档中轻松插入图片。

## 步骤

	### [Typora下载，支持正版](https://Typora.whsykji.com/index.html?bd_vid=11302008112874368681)

安装完，打开进入设置页面。选择图像。

![image 1](https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master/image-20240325164305110.png)

### [Github 创建仓库](https://github.com/)

1. ![image-20240325163314230](https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master/image-20240325163314230.png)

2. 如果之前有 token，有 repo 的访问权限，可以跳过此步。若无，则在 [settting](https://github.com/settings/profile) 里选择 [Developer Settings](https://github.com/settings/apps).

   ![image-20240325163857581](https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master/image-20240325163857581.png)

   ![image-20240325164028910](https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master/image-20240325164028910.png)

   复制这个 token，并且找个安全地方保存好。

## 安装和设置 PicGo

#### 安装

点击 [PicGo Github](https://github.com/Molunerfinn/PicGo) 去下载，安装（建议下载 release lastest 版本）

#### 配置

1. ![image-20240325165017267](https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master/image-20240325165017267.png) 

   然后安装。安装时候可能提醒你需要安装 node.js，然后重启应用。

   如果你安装了 brew，终端执行 ``brew search node`` 。 然后安装一个目前较多用的@12 以上的版本。``brew install node @20``。在此不多赘述。然后重启 PicGo。

2. ![image-20240325170044594](https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master/image-20240325170044594.png)

   customUrl 设定自定义域名。这里改为用 CDN 的，速度快。如果不改，我之前是即便上传成功也不会在 PicGo 和 Typora 中显示。
   ``https://cdn.jsdelivr.net/gh/github 用户名/仓库名@分支名``。点击确定，并设置为默认的图库。

3. 然后在 Typora 的设置页面中测试下。

   ![image-20240325170432030](https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master/image-20240325170432030.png)

   它会默认 upload 两张图片。提示验证成功就 ok 了。如果提示失败可以打开它提示的路径查看 log 。

## 可能的错误

1. 上传成功了，但是在 PicGo 和 Typora 中不现实。提示 load image failed 。

   * 关于这个错误的提示，我之前遇到过是的 PicGo 中直接使用的 GitHub 而不是 GithubPlus。在使用 GithubPlus 而且使用自定义域名使用 CDN 后正常
   * 之前也遇到过因为 Typora 的版本问题，升级后 ok 。
   * 还有一种情况是 github 仓库设置的不是 public。

2. PicGo 上传之后返回图片链接出现乱码，不能正确显示。

   试试用 PicGo 的稳定版本，不要用 beta 版。

### 补充

1. github 的单个仓库最大 1G 限制，单个文件最大 100MB 限制。
2. Typora 里点击验证图片上传，成功一次后。如果是 github 做图床，设置不变，再点就是 fail了，同名文件了。

