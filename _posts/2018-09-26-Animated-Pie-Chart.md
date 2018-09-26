---
layout: post
title: 用D3实现有动画效果的饼状图
---

饼状图是数据可视化中最为常见的一种图形了，尤其适合展示多个分类所占百分比。饼状图用D3很容易实现，但是如何给饼状图加入动画效果，则具有一定的难度。本文将介绍如何用D3（v4）实现饼状图的动画效果。<a href="https://scanthonie.github.io/d3-data-visualization-snippets/a001/index.html" target="_blank">请先点击此处，预览最终的效果。</a>

## First things first

首先，我们需要准备好[第4版的d3.js](https://cdnjs.com/libraries/d3)（这份教程中的代码未在第5版的d3中进行测试）。除了d3.js本身以外，因为用到了额外的d3配色，因此还需准备好[d3-scale-chromatic.js](https://github.com/d3/d3-scale-chromatic)。这两个Javascript框架既可以下载到本地引用，也可以使用CDN中的文档。

我们要可视化的数据存储在一个csv文档中，因此，受浏览器读取本地文件的权限限制，如果要在本地查看效果，需要用node或Python等工具搭建一个简单的本地服务器。好一些代码编辑器的插件也能实现本地服务器的效果，也可以使用。

在我们的`index.html`中，加入引用d3库和我们自己所写的Javascript代码的几行命令：

{% highlight html %}
<script src="../common/js/lib/d3.v4.min.js"></script>
<script src='../common/js/lib/d3-scale-chromatic.v1.min.js'></script>
<script src="js/draw_canvas1.js"></script>
{% endhighlight %}

你需要根据自己文档的存储位置，调整js文件的路径和文件名。下一步，我们在`index.html`中加入一个h1元素，以及一个供d3操纵的SVG元素：

{% highlight html %}
<h1>An Animated Pie Chart with D3.js</h1>
<div id="visualization">
    <svg id="canvas1"></svg>
</div>
{% endhighlight %}

装载有数据的csv文档内容如下，第一列为书籍的主题，第二列为该主题书籍的收藏量：

{% highlight csv %}
subject,item_no
History,25156
Social Sciences,12546
Political Science,15145
Law,54154
Education,41654
Music,50551
Fine Arts,51516
{% endhighlight %}

## 编辑绘制饼图的`draw_canvas1.js`

在`draw_canvas1.js`中的全部代码，都被包含在一个叫`show()`的函数之中。最终在html中，我们会加入代码，直接调用`show()`函数，然后就能绘制出饼图。我们先添加一段读取csv文档的代码：

{% highlight javascript %}
var data1;
d3.csv(
  'data/data1.csv',
  function(d) {
    return {
      subject: d.subject,
      item_no: +d.item_no //“+”将字符变量转化为数字变量
    };
  },
  function(data) {
    data1 = data;
    exec(); // 调用“exec()”函数执行我们的绘制
  }
);
{% endhighlight %}

下一步，是编写`exec()`函数，它同样被包含在`show()`函数里面。`exec()`函数的第一部分，是为svg元素设置尺寸和边距等数值，然后添加一个g元素。我们绘制的饼状图就会被包含在g元素中。

{% highlight javascript %}
var margin = { top: 20, bottom: 20, right: 20, left: 45 },
  width = 800 - margin.left - margin.right,
  height = 500 - margin.top - margin.bottom;
var canvas1 = d3
  .select('#canvas1')
  .attr('width', width + margin.left + margin.right)
  .attr('height', height + margin.top + margin.bottom)
  .append('g')
  .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')');
{% endhighlight %}

接着，计算出我们数组`data1`的长度，存储在`length_data1`变量里面，以便稍后利用d3的组件计算出饼状图各部分的颜色。

{% highlight javascript %}
var length_data1 = data1.reduce(function(p, el) {
  if (p) {
    return 1 + parseInt(p);
  } else {
    return 1;
  }
});
{% endhighlight %}

`colors1()`函数能计算出饼图各部分的颜色。将`d3.interpolatePuBuGn`替换为其它的d3预设，可以实现不同的颜色效果。

{% highlight javascript %}
function colors1(i) {
  return d3.interpolatePuBuGn(i / length_data1);
}
{% endhighlight %}

饼状图实际上是通过SVG的`path`元素来绘制，我们首先需要用到`d3.arc()`方法，来将饼状图中各个部分的角度转化为`path`元素的`d`属性。我们绘制的是中间空心的饼状图，因此需要设置`outerRadius`和`innerRadius`里、外两个半径。

{% highlight javascript %}
var arc = d3
  .arc()
  .outerRadius((height / 2) * 0.7)
  .innerRadius((height / 2) * 0.33);
{% endhighlight %}

`d3.pie()`能根据数据每一行的具体数字，计算出饼状图中每一牙圆弧对应的角度。使用`padAngle()`可以在每一牙之间设置一个间距。

{% highlight javascript %}
var pie = d3
  .pie()
  .sort(null)
  .padAngle(0.04)
  .value(function(d) {
    return d.item_no;
  });
{% endhighlight %}

接着用`d3.transition()`设置动画的时间间隔，和间隔中动画速度的变化。

{% highlight javascript %}
var t = d3
  .transition()
  .ease(d3.easeQuad) //动画的速度由慢变快
  .duration(2800);
{% endhighlight %}

使用定义好的`pie()`函数和`data1`中存储的数据，计算出饼状图各个部分的弧形的角度，存储在变量`arcs1`中。在`canvas1`中增加一个g元素，然后用设置`transform`属性，让其位置居中，同时加入左右翻转，已实现动画从左往右的效果。

{% highlight javascript %}
var arcs1 = pie(data1);

var pieContainer = canvas1
  .append('g')
  .attr('transform', 'translate(' + width / 2 + ' ' + height / 2 + ') scale(-1, 1)');
{% endhighlight %}

在 `exec()`函数中加入一个`plot_pie()`函数，绘制出饼图。其中`transition()`方法之后的`attrTween()`方法，为`path`元素的`d`属性设置了动画效果。具体计算`d`属性值的`tweenArcs()`函数，会在下一步设置。

{% highlight javascript %}
function plot_pie() {
  var arcElements = pieContainer.selectAll('.pieChart').data(arcs1, function(d, i) {
    return i;
  });
  arcElements.exit().remove();
  var enterArcElements = arcElements
    .enter()
    .append('path')
    .attr('class', 'pieChart')
    .merge(arcElements)
    .attr('fill', function(d, i) {
      return colors1(i);
    })
    .transition(t)
    .attrTween('d', tweenArcs);
}
{% endhighlight %}

`tweenArcs()`是一个二阶函数，它的输入值是和SVG绑定的夹角数据，然后返回一个根据动画过渡进度变量`t`计算出`d`属性值的函数。

{% highlight javascript %}
function tweenArcs(d) {
  var interpolator = getArcInterpolator(this, d);
  // this是当前d3在处理的圆弧对象
  return function(t) {
    return arc(interpolator(t));
    //arc()函数的输入值是圆弧的角度，因此interpolator()函数计算出的夹角需要随着t而变化
  };
}
{% endhighlight %}

用`d3.interpolate()`方法实现了`getArcInterpolator()`函数。

{% highlight javascript %}
function getArcInterpolator(el, d) {
  var oldValue = el._oldValue;
  var interpolator = d3.interpolate(
    {
      startAngle: oldValue ? oldValue.startAngle : 0,
      endAngle: oldValue ? oldValue.endAngle : 0
    },
    d
  );
  el._oldValue = interpolator(0);

  return interpolator;
}
{% endhighlight %}

最后，在`exec()`函数中调用`plot_pie()`函数，即可实现绘图。同时也别忘了在`index.html`中加入调用`show()`函数的Javascript代码哦。

[本示例的完整代码，请在这里查看。](https://github.com/scanthonie/d3-data-visualization-snippets/tree/master/a001)
