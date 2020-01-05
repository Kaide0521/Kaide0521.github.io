1. 安装next主题
git clone https://github.com/iissnan/hexo-theme-next themes/next
放在themes目录下
blog
----themes
--------next

2. 添加RSS源
 npm install --save hexo-generator-feed
 
编辑Blog/_config.yml文件，在文件末尾添加

# Extensions
## Plugins: http://hexo.io/plugins/
plugins: hexo-generate-feed

配置主题_config.yml文件，command+f搜索rss，在后面加上/atom.xml

# Set rss to false to disable feed link.
# Leave rss as empty to use site's feed link.
# Set rss to specific value if you have burned your feed already.
rss: /atom.xml //注意：有一个空格


3. 添加字数统计
https://github.com/theme-next/hexo-symbols-count-time


站点地图
npm install hexo-generator-sitemap --save
npm install hexo-generator-baidu-sitemap --save


标签

归档

搜索

留言