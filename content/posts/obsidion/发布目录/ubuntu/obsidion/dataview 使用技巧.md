---
tags:
  - obsidian
  - blog
categories: ubuntu
title: dataview 使用技巧
date: 2024-03-31T02:00:36.531Z
lastmod: 2024-09-15T14:41:08.145Z
---
\#obsidian

```table-of-contents
style: nestedList # TOC style (nestedList|inlineFirstLevel)
maxLevel: 0 # Include headings up to the speficied level
includeLinks: true # Make headings clickable
debugInConsole: false # Print debug info in Obsidian console
```

### dateview 可以用来生成目录

节选：[快速聚合 Obsidian 笔记，试试用 Dataview 生成目录](https://new.qq.com/rain/a/20210818A05T0100)

以下是一些场景：

生成包含同样关键字的笔记的目录

生成同一个标签的笔记的目录

生成同一个作者的书目的目录

这时候，Dataview 就派上用场了。

Dataview 可以生成 MOC，或者你也可以跟我一样，不去管 MOC 是什么，就把它当作一个目录。

我们知道，目录是一篇文章的概览。Dataview 其实生成了你的多篇文章目录。

如果你有这种需要的话，就接着看下去吧。

安装插件

#### 从文件名

场景：假设，现在你有多篇关于【习惯】的笔记，并且这些笔记的名称中都有 「习惯」 两个字。记录了好几个习惯养成的方法，今天你忽然意识到，关于习惯的方法论已经看了好几个了，想把它们列出来放到一起看看能不能产生什么化学效果。

老规矩，先贴一个语法和效果图：

![inews.gtimg.com/newsapp\_bt/0/13894494258/641](http://inews.gtimg.com/newsapp_bt/0/13894494258/641)

> ```dataview
> list
> from "工具"
> where
> contains(file.name,"Dataview")
> sort file.ctime
> ```

> list：你创建了一个列表 / 清单。\
> from：留空就是不筛选文件夹和标签，从所有笔记文件去找。\
> where 条件：匹配（contains）了文件名（file.name）中包含「习惯」两个字的笔记\
> 如果你需要排序，就写 sort，不需要，留空就可以。

![inews.gtimg.com/newsapp\_bt/0/13894494259/641](http://inews.gtimg.com/newsapp_bt/0/13894494259/641)

Dataview 使用 yml 的元素

where 和 sort 就可以直接使用你在 yml 中设置的 key 了。看看下面的写法：\
![inews.gtimg.com/newsapp\_bt/0/13894494260/641](http://inews.gtimg.com/newsapp_bt/0/13894494260/641)

上面依次是：包含作者为「鲁迅」的原笔记、聚合作者为「鲁迅」的目录编辑模式、聚合作者为「鲁迅」的目录预览模式。

嗯哼，鲁迅合集就做好了。

扩展：创建一个书目列表吧！

上面你已经学会了限制检索范围、模糊搜索文件名来创建列表，学会了从标签创建列表，还学会了用 yml 定义的 key 当作字段做条件，基本的检索语法你已经学会啦。

现在我们想在展示形式上有所改变，比如书目列表。我不想只展示作品的名字，我还想展示作者、阅读日期、标签。

还记得上面列表中的展示形式吗？

对，就是 Dataview 语法的第一行那个 ，现在我们来变一下，写个 吧。

不过我们的语法有一点小小的改动。既然是 table，那么列名就必不可少。

先贴一下代码看看吧:\
![inews.gtimg.com/newsapp\_bt/0/13894494261/1000](http://inews.gtimg.com/newsapp_bt/0/13894494261/1000)

![](https://picgo.myjojo.fun:666/i/2024/01/02/6593c9699601b.webp)

![](https://picgo.myjojo.fun:666/i/2024/01/02/6593c9b45231d.webp)\
![image.png](https://picgo.myjojo.fun:666/i/2024/01/02/6593cac21f27a.png)\
![03032201kqorj7j6vx6010xpnn940a0hrf!nd\_dft\_wlteh\_webp\_3.webp](https://picgo.myjojo.fun:666/i/2024/01/02/6593cae7763e1.webp)\
![03032201kqorj7j6vx6010xpnn95cww3rf!nd\_dft\_wlteh\_webp\_3.webp](https://picgo.myjojo.fun:666/i/2024/01/02/6593cb042dcd9.webp)\
![03032201kqorj7j6vx6010xpnn96dffzee!nd\_dft\_wlteh\_webp\_3.webp](https://picgo.myjojo.fun:666/i/2024/01/02/6593cb2145851.webp)
