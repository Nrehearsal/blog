---
title: "Nginx学习笔记之简介与编译指南"
date: 2018-11-18T19:08:23+08:00
draft: false
---

{{% summary %}}
> 身为一个中国人，最大的痛苦时忍受别人“推己及人”的次数，比世界上任何地方的人都要多。--王小波
{{% /summary %}}

### **nginx主要应用场景**

	nginx一般作为网络的边缘节点，主要有以下三个应用场景
    
- **静态资源服务器**

    使用nginx提供本地文件的网络访问服务
- **反向代理服务器**

    缓存：缓存上游服务的不常改变的内容，直接由nginx提供，减少用户访问时延。
    
	负载均衡服务器：方便动态扩容，和容灾，提高服务的可用和可靠性。
- **高性能API服务器**：当应用服务的性能达到瓶颈，可使用nginx和nginx支持动态语言(如JavaScript，Lua)，由nginx直接访问缓存服务和数据库服务，直接提供API功能，或者作为web防火墙，如OpenResty等。

### **nginx的四个主要的组成部分**

- **nginx二进制可执行文件**：nginx与其模块编译生成的文件

- **nginx.conf配置文件**：控制nginx的行为

- **access.log访问日志**：记录每一条http请求信息

- **error.log错误日志**：定位错误发生位置

### **版本选择**

- [官方开源版本](http://nginx.org/)：推荐使用的版本，源代码开放，可根据业务需求灵活配置。

- [官方商业版本](https://www.nginx.com/)：在整合第三方模块，运营监控，技术支持上有很多优点，缺点就是不开源。

- [Tengine](http://tengine.taobao.org/)：阿里巴巴出品，虽然很多特性领先于官方版本，但无法与nginx同步升级，如无特定需求，不推荐使用。

- [OpenResty开源版](https://openresty.org/)：使用lua语言开发第三方模块，兼具高性能和高开发效率。

- [OpenResty商业版](https://openresty.com/)：OpenResty的商业版本，开箱即用。

### **编译指南**
为了方便添加第三方模块，编译安装的方式比从直接软件仓库安装更常用。

- **源代码下载**：从nginx.org下载nginx源代码

- **源代码目录结构**

    ```sh
    total 764
    drwxr-xr-x 8 ubuntu ubuntu   4096 Nov  6 21:52 ./
    drwxrwxr-x 3 ubuntu ubuntu   4096 Nov 18 18:13 ../
    drwxr-xr-x 6 ubuntu ubuntu   4096 Nov 18 18:13 auto/
    -rw-r--r-- 1 ubuntu ubuntu 287441 Nov  6 21:52 CHANGES
    -rw-r--r-- 1 ubuntu ubuntu 438114 Nov  6 21:52 CHANGES.ru
    drwxr-xr-x 2 ubuntu ubuntu   4096 Nov 18 18:13 conf/
    -rwxr-xr-x 1 ubuntu ubuntu   2502 Nov  6 21:52 configure*
    drwxr-xr-x 4 ubuntu ubuntu   4096 Nov 18 18:13 contrib/
    drwxr-xr-x 2 ubuntu ubuntu   4096 Nov 18 18:13 html/
    -rw-r--r-- 1 ubuntu ubuntu   1397 Nov  6 21:52 LICENSE
    drwxr-xr-x 2 ubuntu ubuntu   4096 Nov 18 18:13 man/
    -rw-r--r-- 1 ubuntu ubuntu     49 Nov  6 21:52 README
    drwxr-xr-x 9 ubuntu ubuntu   4096 Nov 18 18:13 src/
    ```
    auto：cc目录(编译相关)，lib目录(相关库)，os目录(判断操作系统类型)，其他文件辅助configure脚本判定nginx支持模块，以及操作系统特性。
    
    CHANGES：各个版本的特性和bugfix。
    
    conf：配置实例文件，方便配置nginx。
    
    configure：根据参数生产编译所需的中间文件，和编译环境准备。
    
    contrib：存放一些工具脚本，如vim配置，使vim支持nginx配置文件语法高亮。
    
    html：提供两个标准html文件，50x.html错误页面，index.html默认欢迎首页。
    
    man：linux的nginx帮助信息文件。
    
    src：nginx源代码。
    
- **configure**

    ```sh
    $ ./configure --help | more
    
    --prefix=PATH                      set installation prefix
    --sbin-path=PATH                   set nginx binary pathname
    --modules-path=PATH                set modules path
    --conf-path=PATH                   set nginx.conf pathname
    --error-log-path=PATH              set error log pathname
    --pid-path=PATH                    set nginx.pid pathname
    --lock-path=PATH                   set nginx.lock pathname
    ```
    路径相关: 如安装路径，二进制文件存放路径，动态模块路径，运行时文件存放路径。如无特别变动，只需配置prefix(安装路径)即可。
    
    ```sh
    --with-http_degradation_module     enable ngx_http_degradation_module
    --with-http_slice_module           enable ngx_http_slice_module
    --with-http_stub_status_module     enable ngx_http_stub_status_module
    
    --without-http_charset_module      disable ngx_http_charset_module
    --without-http_gzip_module         disable ngx_http_gzip_module
    ```
    模块相关:
    --with-xxx(xxx模块默认不启动)主动添加该模块;
    --with-out-xxx(xxx模块默认启动)主动去掉该模块;
    
    ```
    --with-cc=PATH                     set C compiler pathname
    --with-cpp=PATH                    set C preprocessor pathname
    --with-cc-opt=OPTIONS              set additional C compiler options
    --with-ld-opt=OPTIONS              set additional linker options
    --with-cpu-opt=CPU                 build for the specified CPU, ...
    ```
    编译相关：编译器优化参数，cpu平台架构等相关配置等。
    
- **中间文件**
	
	执行configure和make后，会在**objs**目录下生成一些中间文件
    ```sh
    /home/ubuntu/nginx_learn/nginx-1.14.1/objs
    
    total 4.0M
    -rw-rw-r-- 1 ubuntu ubuntu  17K Nov 18 18:51 autoconf.err
    -rw-rw-r-- 1 ubuntu ubuntu  39K Nov 18 18:51 Makefile
    -rwxrwxr-x 1 ubuntu ubuntu 3.8M Nov 18 18:57 nginx
    -rw-rw-r-- 1 ubuntu ubuntu 5.3K Nov 18 18:57 nginx.8
    -rw-rw-r-- 1 ubuntu ubuntu 6.8K Nov 18 18:51 ngx_auto_config.h
    -rw-rw-r-- 1 ubuntu ubuntu  657 Nov 18 18:51 ngx_auto_headers.h
    -rw-rw-r-- 1 ubuntu ubuntu 5.6K Nov 18 18:51 ngx_modules.c
    -rw-rw-r-- 1 ubuntu ubuntu  34K Nov 18 18:57 ngx_modules.o
    drwxrwxr-x 9 ubuntu ubuntu 4.0K Nov 18 18:51 src
    ```
    其中ngx_modeles.c显示了那些模块会被编译进nginx

- 编译

    ```sh
    $ make
    ```
    
    同时会在**objs**目录下生成nginx二进制文件，和相关so文件。

- 安装

    ```sh
    $ make install
    ```
    
    会将相关文件拷贝到prefix执行的目录中:
    ```
    total 16K
    drwxrwxr-x 2 ubuntu ubuntu 4.0K Nov 18 19:01 conf
    drwxr-xr-x 2 ubuntu ubuntu 4.0K Nov 18 19:01 html
    drwxrwxr-x 2 ubuntu ubuntu 4.0K Nov 18 19:01 logs
    drwxrwxr-x 2 ubuntu ubuntu 4.0K Nov 18 19:01 sbin
    ```
    
    分别对应上文中指出的nginx的四个主要组成部分。
