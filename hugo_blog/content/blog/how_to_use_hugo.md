---
title: "如何使用Hugo和GithubPages搭建和管理个人博客"
date: 2018-10-14T15:49:58+08:00
draft: false
---

{{% summary %}}
> 死会引人哭泣。虽则如此，人生的三分之一却在睡眠中打发掉了。——拜伦
{{% /summary %}}

<br/>[**Hugo**](https://github.com/gohugoio/hugo)是由Go语言实现的静态网站生成。
具有简单、易用、高效、易扩展、快速部署等优点。可将markdown格式的文本结合自定义的主题生成静态的html。

[**GithubPages**](https://help.github.com/articles/what-is-github-pages/)是一个静态站点托管服务，旨在直接从GitHub存储库托管您的个人，组织或项目页面。

Hugo+GithubPages的方案相比于传统的博客系统有以下优点：

- 低成本:&nbsp; 无需单独购买VSP服务器或者域名空间，并且不限流量。

- 高度定制化：可以方便的定义网站功能模块，增加个性化功能。

- 方便管理:&nbsp; 由于不需要使用数据库，所有数据均以md形式保存，可直接使用git
行管理，对于博客系统尤其适用。

- 更专业:&nbsp; 对于开源项目的文档，可以使用git与项目版本同步。

<br/>
### **安装Hugo**:
- 使用yum, apt-get, brew安装
-  通过源码安装
    -  安装golang，并配置$GOPATH
    -  编译安装hugo
	
        ```sh
		$ git clone https://github.com/gohugoio/hugo.git 
		$ cd hugo
		$ go install
		$ which hugo //验证安装
		```
		
### **如何使用Hugo**:  
- 创建博客模板

    ```sh
        $ hugo new site blog    //创建一个新站点
        $ cd blog
		
        $ tree blog
        #blog
        #├── archetypes
        #│   └── default.md
        #├── config.toml  //配置文件，可用于配置，网站主题，CopyRight，标题，站点结构等信息
        #├── content    //网站结构目录，更具sitemap生成，对应每个页面的md文件
        #├── data
        #├── layouts
        #├── static
        #└── themes    //主题目录，可从下文中的网站下载hugo主题
    ```
- 网站主题
    - [挑选喜欢的主题](https://themes.gohugo.io/)
    - 安装主题(以crab主题为例)
	
        ```sh
        $ cd blog/themes
        $ git clone https://github.com/thomasheller/crab.git
        $ cd crab
        $ ls 
        # archetypes  exampleSite  images  layouts  LICENSE.md  README.md  static  theme.toml  //exampleSite中包含了一个Demo站点
        
		$ cp exampleSite/config.toml ../blog/   //将Demo网站的配置文件拷贝到blog/目录下
        $ cp -r exampleSite/content/* ../blog/content/  //将Demo网站的站点md文件拷贝到blog/content/下
        ```
- 运行Demo网站
    ```sh
    $ cd blog/
    $ hugo serve -w
    ```
    访问```http://localhost:1313/```， 即可查看的Demo网站
- 构建自己的网站
    - 根据主题帮助文档修改config.toml内容为自己网站的相关信息
    - 根据content结构创建对应的md文件，例如:
	
        ```sh
        $ cd blog/
        $ hugo new blog/first_blog.md
        ```
    - 编辑first_blog.md，添加自定义内容，并执行```hugo serve -w ```，即可在浏览器查看到新增的博客文章
	
### **部署到GitHubPages**: 
- 创建一个公开的仓库blog
- 本地构建站点配置，以及静态html文件
   - 构建站点
        ```sh
        $ mkdir blog
        $ cd blog
        $ hugo new site hugo_blog
        
        $tree blog
        #blog
        #└── hugo_blog 
        #blog文件夹内包含hugo_blog文件夹，hugo_blog存放配置以及md文件
        #blog文件夹存在，hugo自动生成的静态文件
        
        $ cd hugo_blog
        #配置站点，添加体定义内容md文件
        ```
    - 测试站点是否正常，并构建静态文件
        ```sh
        $ cd hugo_blog
        $ hugo serve -w //测试站点是否配置正确，自定义内容是否添加成功
        $ hugo -d ../   //上级目录为blog
        ```
    - push代码到仓库
        ```sh
        $ cd blo
        $ git init
        $ git add .
        $ git commit -m"first commit"
        $ gti remote add origin git@github.com:username/blog.git
        $ git push origin master
        ```
        访问GitHub，将blog仓库的master分支设置为GitHubPages，并绑定CNAME为自己的域名。访问```https://uesrname.github.io```测试站点是否生效。
        最后，为域名添加CNAME纪录指向```username.github.io```，即可通过自己的域名访问到GitHubPages，也就是自己的博客了。
- 更新博客内容
    - 新增或修改对应的md文件
    - 使用hugo从新构建网站静态文件
    - push代码到远程仓库即可