---
title: "在 CentOS7 上使用 rvm 安装 ruby 并配置 Jeykll 使用 minimal-mistakes 主题"
excerpt: 在 CentOS7 上使用 rvm 安装 ruby 并配置 Jeykll 使用 minimal-mistakes 主题
permalink: /systems/install-jekyll-minimal-mistakes-using-ruby-managed-by-rvm
last_modified_at: 2018-08-02
layout: single
#header:
#  image: /assets/images/openstack.jpg
# author_profile: true
tags:
 - centos 7.x
 - rvm
 - ruby
 - jekyll
 - minimal-mistakes

---

本次的主要目的是使用`minimal-mistakes`这个`Jekyll`主题来构建一个博客，引发一系列劳神费力的事情。

### 安装`rvm`
参考这个帖子： https://tecadmin.net/install-ruby-latest-stable-centos/

- 设置yum源  

  由于国内网络问题，需要事先设置好`yum`的base和epel源，我使用了中国科技大学的yum源，配置不赘述。
  
  可以参考这里：[CentOS 7.x 系统初始化配置](/systems/centos7-system-init)
  
  由于已经指定了yum镜像，我禁用了yum的`fastestmirror`插件：
  ```
  vi /etc/yum/pluginconf.d/fastestmirror.conf
  
  [main]
  enabled=0
  ```
- 安装依赖
  ```
  yum -y install gcc-c++ patch readline readline-devel zlib zlib-devel \
   libyaml-devel libffi-devel openssl-devel make \
   bzip2 autoconf automake libtool bison iconv-devel sqlite-devel
  ```

- 安装rvm
  使用curl安装，下载很慢，先设置代理：
  ```
  export http_proxy="http:st.xxxx.me:44443"
  export https_proxy="http:st.xxxx.me:44443"
  
  curl -sSL https://rvm.io/mpapis.asc | gpg --import -
  curl -L get.rvm.io | bash -s stable

  source /etc/profile.d/rvm.sh
  rvm reload
  
  rvm requirements run
  ```



### 安装ruby

我们使用rvm安装ruby，同理需要设置rvm使用国内的镜像源，我这里替换成了淘宝的源：

```bash
sed -i -E 's!https?://cache.ruby-lang.org/pub/ruby!https://ruby.taobao.org/mirrors/ruby!' $rvm_path/config/db
```

由于Jekyll依赖`ffi`这个模块，它支持Ruby & RubyGems的最新版本是 `ffi 1.9.18`, 并且支持Ruby的版本要求 < 2.5, >= 2.0，因此我选择安装ruby 2.4:

```bash
rvm install 2.4

#安装完成检查已安装的版本
rvm list

#设置默认ruby
rvm use 2.4 --default
ruby --version
```

### 安装Jekyll

- 设置`gem mirrors`

  由于安装过程中gem会下载一堆依赖，我们还需要设置gem的镜像地址，这里还是选择了淘宝的镜像：

  - 全局设置
    ```bash
    gem sources -l
    gem sources --remove https://rubygems.org
    gem sources -a https://ruby.taobao.org
    ```
    
  - 当前项目
    ```bash
    bundle config mirror.https://rubygems.org https://ruby.taobao.org
    ```
    
    或者修改项目目录下的`Gemfile`文件：
    ```bash
    vim Gemfile
    
    #source 'https://rubygems.org/'
    source 'https://ruby.taobao.org/'
    ```

- 安装Jekyll
  ```bash
  gem install bundler jekyll
  jekyll new xxxx.me
  cd my-awesome-site
  ```

### 使用`minimal-mistakes`主题
文档：
> https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide/



- `vim Gemfile`
  ```bash
  gem "minimal-mistakes-jekyll"
  ```

- `vim _config.yml`
  ```bash
  theme: minimal-mistakes-jekyll
  ```

- 安装`minimal-mistakes-jekyll`的依赖
  ```bash
  bundle install
  ```

- 启动服务
  ```bash
  bundle exec jekyll serve --host 192.168.100.100 -w
  ```
  
- 访问服务

  > http://192.168.100.100:4000
  
### 美化定制
- 修改页面宽度
  
  `vim ./_sass/minimal-mistakes/_variables.scss`

  ```scss
  /*
     Breakpoints
     ========================================================================== */
  $small: 600px !default;
  $medium: 768px !default;
  $medium-wide: 1024px !default;
  $large: 1200px !default;
  $x-large: 1366px !default;
  
  /*
     Grid
     ========================================================================== */
  $right-sidebar-width-narrow: 220px !default;
  $right-sidebar-width: 260px !default;
  $right-sidebar-width-wide: 300px !default;
  ```
- 修改页面字体大小

  `vim ./_sass/minimal-mistakes/_reset.scss`  

  ```scss
  html {
    /* apply a natural box layout model to all elements */
    box-sizing: border-box;
    background-color: $background-color;
    font-size: 16px;
  
    @include breakpoint($medium) {
      font-size: 15px;
    }
  
    @include breakpoint($large) {
      font-size: 16px;
    }
  
    @include breakpoint($x-large) {
      font-size: 18px;
    }
  
    -webkit-text-size-adjust: 100%;
    -ms-text-size-adjust: 100%;
  }
  ```

- 给 home 页面文章列表添加更新日期
  
  参考: https://github.com/dvhart/dvhart.github.io/blob/master/_includes/archive-single.html  

  `vim _includes/archive-single.html`  

  ```html
  {% raw %}
  # 删除第 33-35 行，并在第32行后面插入如下部分
    <!-- dvhart: date read-time meta line -->
    <p class="page__meta">
      {% if post.last_modified_at %}
        <i class="fa fa-fw fa-calendar" aria-hidden="true"></i> <time datetime="{{ post.last_modified_at | date: "%Y-%m-%d" }}">{{ post.last_modified_at | date: "%B %d, %Y" }}</time>&emsp;
      {% elsif post.date %}
        <i class="fa fa-fw fa-calendar" aria-hidden="true"></i> <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %d, %Y " }}</time>&emsp;
      {% endif %}
      {% if post.read_time %}<i class="fa fa-clock-o" aria-hidden="true"></i>&nbsp;{% include read-time.html %}{% endif %}
    </p>
    <!-- end dvhart date read-time meta line -->
  {% endraw %}
  ```


- 布局设置

> https://mmistakes.github.io/minimal-mistakes/docs/layouts/#headers  
> https://github.com/mmistakes/minimal-mistakes/issues/892  
> https://github.com/mmistakes/minimal-mistakes/issues/1303