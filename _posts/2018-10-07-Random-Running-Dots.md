---
layout: post
title: 用D3实现鼠标滑动时的圆点随机跳动效果
---

本教程采用D3.js实现在鼠标滚轮滑动时，屏幕上出现的彩色圆点随机跳动效果。这个项目模仿了[Shirley Wu](http://sxywu.com/)[为 The Pudding 制作的一个数据可视化项目](https://pudding.cool/2017/03/hamilton/index.html)的部分效果。

<iframe width="520" height="390" style="overflow:hidden;" src="https://scanthonie.github.io/d3-data-visualization-snippets/a003/index.html" frameborder="0" allowfullscreen></iframe>

<a href="https://scanthonie.github.io/d3-data-visualization-snippets/a003/index.html" target="_blank">点击此处，在新的页面中查看。</a>

## 准备工作

准备好第四版的D3和HTML文档。今天的项目里，我们绘图的`show()`函数被放置在`draw_svg1.js`文件中。准备就绪之后，就可以开始编写我们的Javascript代码了。

## 编写`draw_svg1.js`

`draw_svg1.js`中的全部代码，都包含在一个叫`show()`的函数之中。这个项目的实现，主要依靠D3的`force`布局来实现。首先，参考[Mike Bostock 的这篇文章](https://bl.ocks.org/mbostock/b1f0ee970299756bc12d60aedf53c13b)，创建一个用来为各个节点 (node) 独立进行力模拟的`isolate()`函数，稍后将用它来包裹D3的`force.initialize()`函数：

{% highlight javascript %}
function isolate(force, filter) {
  var initialize = force.initialize;
  force.initialize = function() {
    initialize.call(force, nodes.filter(filter));
  };
  return force;
}
{% endhighlight %}

下一步，配置一些要用到的常量，包括色阶、窗口尺寸和圆点数量等等：

{% highlight javascript %}
var n = 600;
var colorScale = d3.scaleSequential(d3.interpolateRainbow).domain([0, 1]);
var w = window.innerWidth || 580;
var heightWin = window.innerHeight;

var margin = { top: 20, bottom: 20, right: 0, left: 0 },
  width = w - margin.left - margin.right,
  height = 6000 - margin.top - margin.bottom;
{% endhighlight %}

用`nodes`变量来存储我们随机生成的圆点。这里用到了`map()`函数，是函数式编程 (functional programming) 中经常用到的方式。如果需要详细了解，网络上有很多函数式编程的教程和例子。

{% highlight javascript %}
var nodes = d3.range(n).map(function(i) {
  return {
    index: i,
    identifier: i.toString(),
    x: Math.random() * w,
    y: Math.random() * heightWin
  };
});
{% endhighlight %}

往`svg`元素中添加`g`元素，准备绘制圆点的空间。

{% highlight javascript %}
var canvas1 = d3
  .select('#canvas1')
  .attr('width', width + margin.left + margin.right)
  .attr('height', height + margin.top + margin.bottom)
  .append('g')
  .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')');
{% endhighlight %}

创建一个用于力模拟的对象`simulation`。`alphaDecay(0)`确保了圆点能始终保持跳动效果，而不会在等待一段事件后便失去跳动效果。`force('collision', d3.forceCollide(10).strength(1))`让圆点之间产生碰撞的效果，不会重叠在一起。

{% highlight javascript %}
var simulation = d3
  .forceSimulation(nodes)
  .velocityDecay(0.75)
  .alphaDecay(0)
  .force('collision', d3.forceCollide(10).strength(1));
{% endhighlight %}

用`forEach()`函数遍历`nodes`，往svg画布上添加颜色随机，位置随机的圆点。

{% highlight javascript %}
nodes.forEach(function(node) {
    canvas1
      .append('circle')
      .data([node])
      .attr('cx', function(d) {
        return d.x;
      })
      .attr('cy', function(d) {
        return d.y;
      })
      .attr('r', 3.5 + Math.random() * 2)
      .attr('fill', colorScale(Math.random()));
  });
{% endhighlight %}

力仿真过程中，每次节点位置的刷新，都会触发`tick`事件。将`tick`事件与`ticked()`回调函数绑定，然后编写该回调函数，以刷新圆点的位置。

{% highlight javascript %}
simulation.on('tick', ticked);

function ticked() {
  canvas1
    .selectAll('circle')
    .attr('cx', function(d) {
      return d.x;
    })
    .attr('cy', function(d) {
      return d.y;
    });
}
{% endhighlight %}

将鼠标滚轮滑动的事件与`scrollFunc()`函数绑定。`scrollFunc()`函数中，用教程最开头部分的`isolate`函数，将各个节点的`forceX`和`forceY`力度进行独立的设置。x坐标只在水平方向上做一定范围的随机摆动，y坐标则在窗口可见范围上下完全随机变化，以便产生上下跳动的效果。用`simulation.restart()`重启力仿真。代码的最后，主动调用一次`scrollFunc()`函数，使得页面在加载、刷新时也会有一个初始的跳动效果。

{% highlight javascript %}
window.addEventListener('scroll', scrollFunc);

function scrollFunc() {
  var scrollY = window.scrollY || 0;
  nodes.forEach(function(node) {
    simulation.force(
      'y' + node.identifier,
      isolate(d3.forceY(Math.random() * (heightWin + 1200) - 600 + scrollY).strength(0.4), function(d) {
        return d.identifier == node.identifier;
      })
    );
    simulation.force(
      'x' + node.identifier,
      isolate(d3.forceX(((Math.random() - 0.5) * width) / 3 + node.x).strength(0.1), function(d) {
        return d.identifier == node.identifier;
      })
    );
  });
  simulation.restart();
}

scrollFunc();
{% endhighlight %}

大功告成，制作完毕！在HTML文档中添加一些解释文字和CSS效果即可。

[本示例的完整代码，请在这里查看。](https://github.com/scanthonie/d3-data-visualization-snippets/tree/master/a003)
