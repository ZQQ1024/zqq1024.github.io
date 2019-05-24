---
title: "CSS Margin Collapse"
categories:
  - others
tags:
  - CSS
  - margin
classes: wide

excerpt: "理解CSS Margin Collapse"
---

# 1 Introduction

为什么`margin: auto`可以让块级元素水平居中，而不能垂直居中。我理解，主要和Margin Collapse相关。本文介绍Margin Collapse，Margin Collapse主要和Top-Bottom方向相关，和Right-Left方向无关

# 2 Sibling

<p class="codepen" data-height="265" data-theme-id="0" data-default-tab="css,result" data-user="zqq1024" data-slug-hash="dEmMNZ" style="height: 265px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="Margin Collapse ex 2">
  <span>See the Pen <a href="https://codepen.io/zqq1024/pen/dEmMNZ/">
  Margin Collapse ex 2</a> by ZQQ1024 (<a href="https://codepen.io/zqq1024">@zqq1024</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>


![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190524141214.png)

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190524141355.png)

如上，设置了两个连续的div元素，可以看到`div#top`的margin-bottom为15px，而`div#bottom`的margin-top为35px，而最终的`div#top`和`div#bottom`的距离不是35+15=50px而是35px，选择的最长的（负数的同理，-35和-15选-35）

# 3 Child

<p class="codepen" data-height="265" data-theme-id="0" data-default-tab="css,result" data-user="zqq1024" data-slug-hash="ZNxGKQ" style="height: 265px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="Margin Collapse ex 1">
  <span>See the Pen <a href="https://codepen.io/zqq1024/pen/ZNxGKQ/">
  Margin Collapse ex 1</a> by ZQQ1024 (<a href="https://codepen.io/zqq1024">@zqq1024</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

如上我们嵌套的设置了多个div元素。可是元素之间并没有预想的相互之间堆叠，而是完全重叠的
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190524142150.png)

这也解释了为什么`margin: auto`仅仅是水平居中，而不是垂直居中

# 4 Disable

上面介绍了连续/嵌套时Margin Collapse的效果，有时候我们不想有这样的效果，这一部分谈谈如何禁用它，主要有下面两种方式：
- 修改父级元素的overflow、display、float
- 指定border或者padding之类的，不要让margin直接接触

# 5 References

> [https://www.w3schools.com/css/css_margin.asp](https://www.w3schools.com/css/css_margin.asp)  
[https://www.jonathan-harrell.com/whats-the-deal-with-margin-collapse/](https://www.jonathan-harrell.com/whats-the-deal-with-margin-collapse/)  
[https://css-tricks.com/what-you-should-know-about-collapsing-margins/](https://css-tricks.com/what-you-should-know-about-collapsing-margins/)
