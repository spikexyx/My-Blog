---
title: 建站手记
date: 2024-12-25 19:35:08
tags: 
  - 建站手记
cover: "/imgs/screen-250330.png"
---

{% note success %} 

本站使用[Hexo](https://hexo.io/zh-cn/docs/)进行静态搭建 + [Netlify](https://app.netilify.com)进行动态部署，主题选用[Redefine](https://redefine-docs.ohevan.com/zh)

{% endnote %} 

## Hexo静态搭建



{% notel blue 什么是Hexo? %}

Hexo 是一个快速、简洁且高效的博客框架。 Hexo 使用 [Markdown](http://daringfireball.net/projects/markdown/)（或其他标记语言）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

{% endnotel %}

### 安装

安装 Hexo 前需要先安装 Git 和 Node.js

**安装 Git：**

- Windows：下载并安装 [git](https://git-scm.com/download/win)。

- Mac：使用 [Homebrew](http://mxcl.github.com/homebrew/), [MacPorts](http://www.macports.org/) 或者下载 [安装程序](http://sourceforge.net/projects/git-osx-installer/)。

- Linux (Ubuntu, Debian)：`sudo apt-get install git-core`

- Linux (Fedora, Red Hat, CentOS)：`sudo yum install git-core`

**安装 Node.js：**

- Windows：通过 [nvs](https://github.com/jasongin/nvs/)（推荐）或者 [nvm](https://github.com/nvm-sh/nvm) 安装。 

- Mac：使用 [Homebrew](https://brew.sh/) 或 [MacPorts](http://www.macports.org/) 安装。

- Linux（DEB/RPM-based）：从 [NodeSource](https://github.com/nodesource/distributions) 安装。

**安装 Hexo**

所有必备的应用程序安装完成后，即可使用 npm 安装 Hexo：

```bash
npm install -g hexo-cli
```

---

### 建站

安装 Hexo 完成后，执行下列命令，Hexo 将会在指定文件夹中新建所需要的文件

```bash
hexo init <folder>
cd <folder>
npm install
```

初始化后，项目文件夹将如下所示：

```yaml
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

_config.yml 是网站的 [配置](https://hexo.io/zh-cn/docs/configuration) 文件，可以在里面配置大部分的基础参数，具体参数含义可以参考官方文档。

{% notel default fa-info 参考文档 %}

https://hexo.io/zh-cn/docs/

{% endnotel %}

## Netlify部署网站

1. 创建 [Github](https://github.com/) 仓库（过程略）。
2. 将前一步中本地初始化的Hexo文件夹提交到这个github仓库。

```bash
# 在上一步创建的<folder>里进行git初始化
git init
git add .
git commit -m "first commit"
git remote add origin <仓库git链接>
git push -u origin main
```

3. 在 [Netlify](https://app.netlify.com/) 登录（用github账号）并选择新建网站，关联刚创建的网站仓库，点进deploy进行部署即可。

> netlify会给默认创建的免费网站赋予域名<网站名称.netlify.app>，也可以自己去买域名之后关联过来从而使用自己的域名

## 网站美化 - Redefine主题

直接参考Redefine网站安装配置即可：

https://redefine-docs.ohevan.com/zh

## 写博客 & 测试网站页面效果

参考Hexo 的官方文档即可：
https://hexo.io/zh-cn/docs/commands

常用命令包括：

- 建新页面

```bash
hexo new page <pageName>
```

- 建新博客

```bash
hexo new post <postName>
```

- 本地构建

```bash
hexo clean
hexo generate
```

- 本地测试网页效果

```bash
hexo server
```

- 部署到网站（Netlify版本）

```bash
# Netlify会根据关联的github仓库的diff自动部署，所以本地测试OK后直接push到远程，等一段时间就可以看见变化了
git add .
git commit -m "commit"
git push
```

## 附件

1. 我的 _config.redefine.yml

 ```yaml
 # >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # THEME REDEFINE CONFIGURATION FILE V2
 # BY EVANNOTFOUND
 # GITHUB: https://github.com/EvanNotFound/hexo-theme-redefine
 # DOCUMENTATION: https://redefine-docs.ohevan.com
 # DEMO: https://redefine.ohevan.com
 # <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # BASIC INFORMATION >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/basic/info
 info:
   # Site title
   title: Ethan's CyberSpace
   # Site subtitle
   subtitle:
   # Author name
   author: Ethan Xu
   # Site URL
   url: https://ethanxu.netlify.app
 # BASIC INFORMATION <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # IMAGE CONFIGURATION >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/basic/defaults
 defaults:
   # Favicon
   favicon: /imgs/code.png
   # Site logo
   logo: /imgs/icons8-deathly-hallows-96.png
   # Site avatar
   avatar: /imgs/jil-valentine-avatar.png
 # IMAGE CONFIGURATION <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # COLORS >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/basic/colors
 colors:
   #Primary color
   primary: "#5DAC81"
   #primary: "#2EA9DF"
   # Secondary color (TBD)
   secondary:
   # Default theme mode initial value (will be overwritten by prefer-color-scheme)
   default_mode: light # light, dark
 # COLORS <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # SITE CUSTOMIZATION >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/basic/global
 global:
   # Custom global fonts
   fonts:
     # Chinese fonts
     chinese:
       enable: false # Whether to enable custom chinese fonts
       family:  # Font family
       url:  # Font URL to CSS file
     # English fonts
     english:
       enable: false # Whether to enable custom english fonts
       family:  # Font family
       url:  # Font URL to CSS file
     # Custom title fonts (navbar, sidebar)
     title:
       enable: false # Whether to enable custom title fonts
       family:  # Font family
       url:  # Font URL to CSS file
   # Content max width
   content_max_width: 1000px
   # Sidebar width
   sidebar_width: 210px
   # Effects on mouse hover
   hover:
     shadow: true # shadow effect
     scale: false # scale effect
   # Scroll progress
   scroll_progress:
     bar: false # progress bar
     percentage: true # percentage
   # Website counter
   website_counter:
     url: https://cn.vercount.one/js # counter API URL (no need to change)
     enable: true # enable website counter or not
     site_pv: true # site page view
     site_uv: true # site unique visitor
     post_pv: false # post page view
   # Whether to enable single page experience (using swup). See https://swup.js.org/. similar to pjax
   single_page: true
   # Whether to enable Preloader.
   preloader:
     enable: false
     custom_message: # Custom message. If empty, the site title will be displayed
   # Whether to enable open graph
   open_graph:
     enable: true
     image: /images/redefine-og.webp  # default og:image
     description: Hexo Theme Redefine, Redefine Your Hexo Journey.
   # Google Analytics
   google_analytics:
     enable: false # Whether to enable Google Analytics
     id:  # Google Analytics Measurement ID
     # SITE CUSTOMIZATION <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # FONTAWESOME >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/basic/fontawesome
 fontawesome: # Pro v6.2.1
   # Thin version
   thin: false
   # Light version
   light: false
   # Duotone version
   duotone: false
   # Sharp Solid version
   sharp_solid: false
 # FONTAWESOME <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # HOME BANNER >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/home/home_banner
 home_banner:
   # Whether to enable home banner
   enable: true
   # style of home banner
   style: fixed # static or fixed
   # Home banner image
   image:
     light: /imgs/wallhaven-d6y12l.jpg # light mode
     dark: /imgs/wallhaven-d6y12l.jpg # dark mode
   # Home banner title
   title: 𝕰𝖙𝖍𝖆𝖓'𝖘 𝕮𝖞𝖇𝖊𝖗𝕾𝖕𝖆𝖈𝖊
   # Home banner subtitle
   subtitle:
     text: ["Welcome to My Personal Website."] # subtitle text, array
     hitokoto:  # 一言配置
       enable: false # Whether to enable hitokoto
       show_author: false # Whether to show author
       api: https://v1.hitokoto.cn # API URL, can add types, see https://developer.hitokoto.cn/sentence/#%E5%8F%A5%E5%AD%90%E7%B1%BB%E5%9E%8B-%E5%8F%82%E6%95%B0
     typing_speed: 100 # Typing speed (ms)
     backing_speed: 80 # Backing speed (ms)
     starting_delay: 500 # Start delay (ms)
     backing_delay: 1500 # Backing delay (ms)
     loop: true # Whether to loop
     smart_backspace: true # Whether to smart backspace
   # Color of home banner text
   text_color:
     light: "#fff" # light mode
     dark: "#d1d1b6" # dark mode
   # Specific style of the text
   text_style:
     # Title font size
     title_size: 2.8rem
     # Subtitle font size
     subtitle_size: 1.5rem
     # Line height between title and subtitle
     line_height: 1.2
   # Home banner custom font
   custom_font:
     # Whether to enable custom font
     enable: false
     # Font family
     family:
     # URL to font CSS file
     url:
   # Home banner social links
   social_links:
     # Whether to enable
     enable: false
     # Social links style
     style: default # default, reverse, center
     # Social links
     links:
       github:  # your GitHub URL
       instagram: # your Instagram URL
       zhihu:  # your ZhiHu URL
       twitter:  # your twitter URL
       email:  # your email
       # ...... # you can add more
     # Social links with QRcode drawers
     qrs:
       weixin:  # your Wechat QRcode image URL
       # ...... # you can add more
 # HOME BANNER <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # NAVIGATION BAR >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/home/navbar
 navbar:
   # Auto hide navbar
   auto_hide: false
   # Navbar background color
   color:
     left: "#f78736" #left side
     right: "#367df7"  #right side
     transparency: 35 #percent (10-99)
   # Navbar width (usually no need to modify)
   width:
     home: 1200px #home page
     pages: 1000px #other pages
   # Navbar links
   links:
     Home:
       path: /
       icon: fa-regular fa-house # can be empty
     Archives:
       path: /archives
       icon: fa-regular fa-archive # can be empty
 #    Tags:
 #      path: /tags
 #      icon: fa-regular fa-tags
     About:
       path: /about
       icon: fa-regular fa-user
     # Status:
     #   path: https://status.ohevan.com/
     #   icon: fa-regular fa-chart-bar
     # About:
     #   icon: fa-regular fa-user
     #   submenus:
     #     Me: /about
     #     Github: https://github.com/EvanNotFound/hexo-theme-redefine
     #     Blog: https://ohevan.com
     #     Friends: /friends
     # Links:
     #   icon: fa-regular fa-link
     #   submenus:
     #     Link1: /link1
     #     Link2: /link2
     #     Link3: /link3
     # ...... # you can add more
   # Navbar search (local search). Requires hexo-generator-searchdb (npm i hexo-generator-searchdb). See https://github.com/theme-next/hexo-generator-searchdb
   search:
     # Whether to enable
     enable: true
     # Preload search data when the page loads
     preload: true
 # NAVIGATION BAR <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # HOME PAGE ARTICLE SETTINGS >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/home/home
 home:
   # Sidebar settings
   sidebar:
     enable: true # Whether to enable sidebar
     position: left # Sidebar position. left, right
     first_item: menu # First item in sidebar. menu, info
     announcement: "Enjoy your time." # Announcement text
     show_on_mobile: true # Whether to show sidebar navigation on mobile sheet menu
     links:
       Home:
         path: /
         icon: fa-regular fa-house
       Archives:
         path: /archives
         icon: fa-regular fa-archive
       Tags:
         path: /tags
         icon: fa-regular fa-tags
       About:
         path: /about
         icon: fa-regular fa-user
     # Archives:
     #   path: /archives
     #   icon: fa-regular fa-archive # can be empty
     # Tags:
     #   path: /tags
     #   icon: fa-regular fa-tags # can be empty
     # Categories:
     #   path: /categories
     #   icon: fa-regular fa-folder # can be empty
     # ...... # you can add more
   # Article date format
   article_date_format: YYYY-MM-DD # auto, relative, YYYY-MM-DD, YYYY-MM-DD HH:mm:ss etc.
   # Article excerpt length
   excerpt_length: 200 # Max length of article excerpt
   # Article categories visibility
   categories:
     enable: true  # Whether to enable
     limit: 3 # Max number of categories to display
   # Article tags visibility
   tags:
     enable: true  # Whether to enable
     limit: 3  # Max number of tags to display
 # HOME PAGE ARTICLE SETTINGS <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # ARTICLE >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/posts/articles
 articles:
   # Set the styles of the article
   style:
     font_size: 16px # Font size
     line_height: 1.5 # Line height
     image_border_radius: 14px # image border radius
     image_alignment: center # image alignment. left, center
     image_caption: false # Whether to display image caption
     link_icon: true # Whether to display link icon
     delete_mask: false # Add mask effect to <del> tags, hiding content by default and revealing on hover
     title_alignment: left # Title alignment. left, center
     headings_top_spacing: # Top spacing for headings from h1-h6
       h1: 3.2rem
       h2: 2.4rem
       h3: 1.9rem
       h4: 1.6rem
       h5: 1.4rem
       h6: 1.3rem
   # Word count. Requires hexo-wordcount (npm install hexo-wordcount). See https://github.com/willin/hexo-wordcount
   word_count:
     enable: true # Whether to enable
     count: true # Whether to display word count
     min2read: true # Whether to display reading time
   # Author label
   author_label:
     enable: false # Whether to enable
     auto: false # Whether to automatically add author label, e.g. Lv1, Lv2, Lv3...
     list: []
   # Code block settings
   code_block:
     copy: true # Whether to enable code block copy button
     style: mac # mac | simple
     highlight_theme: # Color scheme for highlightjs code highlighting. For preview, see https://highlightjs.org/examples
       light: github # light mode theme, support: github, atom-one-light, default
       dark: vs2015 # dark mode theme, support: github-dark, monokai-sublime, vs2015, night-owl, atom-one-dark, nord, tokyo-night-dark, a11y-dark, agate
     font: # Custom font
       enable: false # Whether to enable
       family: # Font family
       url: # Font URL to CSS file
   # Table of contents settings
   toc:
     enable: true # Whether to enable TOC
     max_depth: 3 # TOC depth
     number: false # Whether to add number to TOC automatically
     expand: true # Whether to expand TOC
     init_open: true # Open toc by default
   # Whether to enable copyright notice
   copyright:
     enable: true # Whether to enable
     default: cc_by_nc_sa # Default license, can be cc_by_nc_sa, cc_by_nd, cc_by_nc, cc_by_sa, cc_by, all_rights_reserved, public_domain
   # Whether to enable lazyload for images
   lazyload: true
   # Pangu.js (automatically add space between Chinese and English). See https://github.com/vinta/pangu.js
   pangu_js: false
   # Article recommendation. Requires nodejieba (npm install nodejieba). Transplanted from hexo-theme-volantis.
   recommendation:
     # Whether to enable article recommendation
     enable: false
     # Article recommendation title
     title: 推荐阅读
     # Max number of articles to display
     limit: 3
     # Max number of articles to display mobile
     mobile_limit: 2
     # Placeholder image
     placeholder: /images/wallhaven-wqery6-light.webp
     # Skip directory
     skip_dirs: []
 # ARTICLE <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # COMMENT >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/posts/comment
 comment:
   # Whether to enable comment
   enable: false
   # Comment system
   system: waline # waline, gitalk, twikoo, giscus
   # System configuration
   config:
     # Waline comment system. See https://waline.js.org/
     waline:
       serverUrl: https://example.example.com # Waline server URL. e.g. https://example.example.com
       lang: zh-CN # Waline language. e.g. zh-CN, en-US. See https://waline.js.org/guide/client/i18n.html
       emoji: [] # Waline emojis, see https://waline.js.org/guide/features/emoji.html
       recaptchaV3Key: # Google reCAPTCHA v3 key. See https://waline.js.org/reference/client/props.html#recaptchav3key
       turnstileKey: # Turnstile key. See https://waline.js.org/reference/client/props.html#turnstilekey
       reaction: false # Waline reaction. See https://waline.js.org/reference/client/props.html#reaction
     # Gitalk comment system. See https://github.com/gitalk/gitalk
     gitalk:
       clientID: # GitHub Application Client ID
       clientSecret: # GitHub Application Client Secret
       repo: # GitHub repository
       owner: # GitHub repository owner
       proxy: # GitHub repository proxy
     # Twikoo comment system. See https://twikoo.js.org/
     twikoo:
       version: 1.6.10 # Twikoo version, do not modify if you dont know what it is
       server_url: # Twikoo server URL. e.g. https://example.example.com
       region: # Twikoo region. can be empty
     # Giscus comment system. See https://giscus.app/
     giscus:
       repo: # Github repository name e.g. EvanNotFound/hexo-theme-redefine
       repo_id: # Github repository id
       category: # Github discussion category
       category_id: # Github discussion category id
       mapping: pathname # Which value to use as the unique identifier for the page. e.g. pathname, url, title, og:title. DO NOT USE og:title WITH PJAX ENABLED since pjax will not update og:title when the page changes
       strict: 0 # Whether to enable strict mode. e.g. 0, 1
       reactions_enabled: 1 # Whether to enable reactions. e.g. 0, 1
       emit_metadata: 0 # Whether to emit metadata. e.g. 0, 1
       lang: en # Giscus language. e.g. en, zh-CN, zh-TW
       input_position: bottom # Place the comment box above/below the comments. e.g. top, bottom
       loading: lazy # Load the comments lazily
 # COMMENT <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # FOOTER >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/footer
 footer:
   # Show website running time
   runtime: false # show website running time or not
   # Icon in footer, write fontawesome icon code here
   # icon: '<i class="fa-solid fa-heart fa-beat" style="--fa-animation-duration: 0.5s; color: #f54545"></i>'
   icon: '<i class="fa-regular fa-rectangle-terminal" style="color: #2EA9DF"></i>'
   # The start time of the website, format: YYYY/MM/DD HH:mm:ss
   start: 2024/12/25 12:00:00
   # Site statistics
   statistics: true # show site statistics or not (total articles, total words)
   # Footer message
   customize: "𝒴𝑜𝓊'𝓇𝑒 𝓃𝒶𝓃𝑜 𝒷𝑜𝑜𝓈𝓉𝑒𝒹 !  𝒢𝑒𝓉 𝒾𝓃 𝓉𝒽𝑒𝓇𝑒 !"
   # ICP record number. See https://beian.miit.gov.cn/
   icp:
     enable: false # Whether to enable
     number: # ICP record number
     url: # ICP record url
 # FOOTER <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # INJECT >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/inject
 inject:
   # Whether to enable inject
   enable: false
   # Inject custom head html code
   head:
     -
     -
     # Inject custom footer html code
   footer:
     -
     -
       # INJECT <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
       # PLUGINS >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
     # Docs: https://redefine-docs.ohevan.com/plugins
 plugins:
   # RSS feed. Requires hexo-generator-feed (npm i hexo-generator-feed). See https://github.com/hexojs/hexo-generator-feed
   feed:
     enable: false # Whether to enable
   # Aplayer. See https://github.com/DIYgod/APlayer
   aplayer:
     enable: false # Whether to enable
     type: fixed # fixed, mini
     audios:
       - name: # audio name
         artist: # audio artist
         url: # audio url
         cover: # audio cover url
         lrc: # audio cover lrc
       #      - name: # audio name
       #        artist: # audio artist
       #        url: # audio url
       #        cover: # audio cover url
       #        lrc: # audio cover lrc
       # .... you can add more audios here
   # Mermaid JS. Requires hexo-filter-mermaid-diagrams (npm i hexo-filter-mermaid-diagrams). See https://mermaid.js.org/
   mermaid:
     enable: false # enable mermaid or not
     version: "11.4.1" # default v11.4.1
 # PLUGINS <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # PAGE TEMPLATES >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/page_templates
 page_templates:
   # Friend Links page column number
   friends_column: 2
   # Tags page style
   tags_style: cloud # blur, cloud
 # PAGE TEMPLATES <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 
 # CDN >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/cdn
 cdn:
   # Whether to enable CDN
   enable: false
   # CDN Provider
   provider: npmmirror # npmmirror, zstatic, cdnjs, jsdelivr, unpkg, custom
   # Custom CDN URL
   # format example: https://cdn.custom.com/hexo-theme-redefine/${version}/source/${path}
   # The ${path} must leads to the root of the "source" folder of the theme
   custom_url:
 # CDN <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 
 # DEVELOPER MODE >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> start
 # Docs: https://redefine-docs.ohevan.com/developer
 developer:
   # Whether to enable developer mode (only for developers who want to modify the theme source code, not for ordinary users)
   enable: false
 # DEVELOPER MODE <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< end
 ```

