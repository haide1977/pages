## 基于Jekyii的Ryan Wang's github blog
**[https://haide1977.me](https://haide1977.me)**  
  
感谢[Qi's blog](https://tianqi.name/blog/) 制作的极简jekyll主题

## 折腾上线的主要内容
- 在ubuntu vm 中搭建 Jekyll 本地开发和调试环境  
- 修改_config.yml 进行个性化设置  
- 解决about没有自动编译生成页面问题。【涉及到config 配置】 
- favicon.ico 和logo 修改 【手动生成了ico和不同规格的logo，不折腾自动生成了】
- 目前的skin 中没有assets 目录【东西都存在 /static 目录下，在编译时候会把图片等编译过去 】
- 对Latex 的支持     
  【修改了js（mathjax） 插件的源，改bootcss 的2.7.2版本，为mathjax.org 的源 。在/_include/utils 目录下设置。】       
- 图片统一到七牛云存储  
 【加自动样式和水印| 采用七牛客户端管理工具（QBOX ）】   
- 购买独立域名和设置域名指向  
- 利用cloudflare.net 服务，快速加上域名https 小绿锁   


## 部分折腾过程参考的网址
- 七牛图片存储策略[开启原图保护 | 图片样式访问]  
  + 官方：[什么是原图保护]( https://developer.qiniu.com/kodo/kb/1359/what-is-the-original-protection) 
  + 官方：[图片样式分割符]( https://developer.qiniu.com/kodo/kb/1327/what-is-the-style-and-the-style-separators)  
- 域名指向设置  
  + github 官方：[github pages 使用独立域名]( https://help.github.com/articles/using-a-custom-domain-with-github-pages/)  
  + [基于godaddy购买域名的github指向设置经验]( https://medium.com/@supriyakankure/how-to-add-a-custom-domain-to-your-github-page-with-godaddy-84495781143e)   
- https的小绿锁
  + [利用cloudflare给github页面上https小绿锁]( https://blog.cloudflare.com/secure-and-fast-github-pages-with-cloudflare/)  

## 原主题风格readme中的部分链接 
- 分页（[jekyll-paginate](https://github.com/jekyll/jekyll-paginate)）
- 文章目录（[TOC](http://projects.jga.me/toc/)）
- 阅读次数统计（[LeanCloud](https://leancloud.cn/)）
- Emoji（[Jemoji](https://github.com/jekyll/jemoji)）
- 评论（[Disqus](https://disqus.com/)）
- 网站图标的自动化工具（[gulp-svg2png](https://www.npmjs.com/package/gulp-svg2png), [gulp-to-ico](https://www.npmjs.com/package/gulp-to-ico)）
- 数学公式([MathJax](https://www.mathjax.org/))
- RSS（[jekyll-feed](https://github.com/jekyll/jekyll-feed)）


## 原主题风格的作者信息

在线访问：[Qi's blog](https://tianqi.name/blog/)

示例源码：[https://github.com/kitian616/kitian616.github.io/](https://github.com/kitian616/kitian616.github.io/)