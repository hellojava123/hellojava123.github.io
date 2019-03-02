## 简介

样式设计来自 [https://www.yinwang.org/](http://www.yinwang.org/)。

## 演示

smallyu [https://www.smallyu.net](https://www.smallyu.net)

## 特性

- 完全还原yinwang.org的样式

- 首页适配移动端

- 首页新增分页显示

- 首页新增版权显示

- 文章末尾显示返回首页链接

- 自定义链接跳转是否新标签打开

## 配置示例

```yml
# html lang
language: en

# 导航栏
menu:
  语雀: https://www.yuque.com/smallyu
  关于: /about

# 图标
favicon: /favicon.ico

# 建议页数大于1开启
paginator: false

# Header
header:
  blank: true     # 头部链接是否新标签页打开

# Footer
footer:
  display: false               # 是否显示footer
  context: "© smallyu.net"     # footer内容

# 文章页
post:
  blank: true        # 文章是否新标签页打开
  backHome: false     # 文章末尾是否显示返回首页
  backHome_hr: false  # 返回首页前是否显示分割线
  backHome_prefix: "↶ "       # 返回首页的前缀内容
  backHome_content: "返回首页"  # 返回首页的主体内容
  backHome_suffix: ""          # 返回首页的后缀内容
```
