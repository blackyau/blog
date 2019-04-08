---
layout: post
title: Typecho部署“一言”
date: 2016-09-11 14:38:00
updated: 2016-09-11 15:26:00
categories: 教程
tags: 
    - WEB
    - PHP
    - Typecho
urlname: 5
comment: true
---
在Telegram闲逛的时候，看到一名大佬发了一个链接。戳进去看了看，原来是一个一言API复刻版。试了试速度还不错，正好也从来没弄过，所以今天也就把它鼓捣上了。引用方法也很简单。

<!-- more -->

对于Typecho来说，要将header.php中的head区加入以下代码

```php
    <script type="text/javascript" src="//api.i-meto.com/hitokoto?encode=js"></script>
```

然后再将footer.php以下代码标为注释

```php
    &copy;
      <span itemprop="copyrightYear"><?php echo date('Y'); ?></span>
      <span class="with-love">
       <i class="icon-next-heart fa fa-heart"></i>
      </span>
      <span class="author" itemprop="copyrightHolder">
      <?php if ($this->options->next_name) {
          $this->options->next_name();
      } else {
          $this->options->title();
      }
      ?>
      </span>
```

然后就在注释的外面加上

```php
    <script>hitokoto()</script>
```

就这样我的blog底端就有高逼格的、会自动刷新的一言了

[点我](https://st.blackyau.net/blog/5/Typecho.zip)可以直接下载已修改的footer.php、header.php。直接替换即可生效