---
layout: post
title: 用D3实现有数值提示线的线图
---

线图常用来展示时间序列数据。本文将介绍如何用D3（v4）实现有色阶过渡效果的线图，同时为它添加数值指示线。最终的成品效果如下：

<iframe width="520" height="385" src="https://scanthonie.github.io/d3-data-visualization-snippets/a002/viz.html" frameborder="0" allowfullscreen></iframe>

<a href="https://scanthonie.github.io/d3-data-visualization-snippets/a002/index.html" target="_blank">点击此处，在新的页面中查看。</a>

## 准备工作

首先，参考[用D3实现有动画效果的饼状图](https://scanthonie.github.io/Animated-Pie-Chart/)这篇文章，准备好我们的HTML模版，[第4版的d3.js](https://cdnjs.com/libraries/d3)，和我们要添加`show()`函数的`draw_canvas1.js`文件。

然后准备好存储数据的csv文档，内容如下：

{% highlight csv %}
month,visits
Jan.,26451
Feb.,25465
Mar.,25648
Apr.,35646
May,34654
June,38015
Jul.,52054
Aug.,72507
Sept.,78191
Oct.,45258
Nov.,65241
Dec.,30214
{% endhighlight %}

## 编写`draw_canvas1.js`

`draw_canvas1.js`中的全部代码，都包含在一个叫`show()`的函数之中。首先在里面添加一段读取csv文档的代码：

{% highlight javascript %}
var data1;
d3.csv(
  'data/data1.csv',
  function(d) {
    d.visits = +d.visits;
    return d;
  },
  function(data) {
    data1 = data;
    exec(); // 调用“exec()”函数执行图形绘制
  }
);
{% endhighlight %}

下一步，设置控制动画的`transition()`，以及图形尺寸等全局变量：

{% highlight javascript %}
var t = d3
  .transition()
  .ease(d3.easeQuad)
  .duration(2000);
var margin = { top: 20, bottom: 20, right: 20, left: 45 },
  width = 580 - margin.left - margin.right,
  height = 380 - margin.top - margin.bottom;
var canvas1 = d3
  .select('#canvas1')
  .attr('width', width + margin.left + margin.right)
  .attr('height', height + margin.top + margin.bottom)
  .append('g')
  .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')');
{% endhighlight %}

下一步，编写`exec()`函数，先设置`xScale`和`yScale`两个对象。由于横坐标轴上的是离散数据，使用了`d3.scalePoint()`，纵坐标是连续的数值数据，使用了`d3.scaleLinear()`。

{% highlight javascript %}
function exec() {
  var xScale = d3
    .scalePoint()
    .range([0, width])
    .domain(
      data1.map(function(d) {
        return d.month;
      })
    );

  var maxVisits = d3.max(data1, function(d) {
    return +d.visits;
  });
  maxVisitsCeil = Math.ceil(maxVisits / 1000) * 1000;
  var yScale = d3
    .scaleLinear()
    .range([height, 0])
    .domain([0, maxVisitsCeil]);
}
{% endhighlight %}

在`exec()`函数中添加下面的函数调用代码，分别实现添加过渡色阶、线条下的着色区域、线条本身、坐标轴和动态数值指示器的效果。

{% highlight javascript %}
execGradients(yScale);
execArea(xScale, yScale, data1);
execLine(xScale, yScale, data1);
execAxis(xScale, yScale);
execMouseTracker(xScale, yScale, data1);
{% endhighlight %}

首先用下面的代码完成色阶过渡的效果。需要分别设置两个过渡的色阶，`#area-gradient`和`#line-gradient`，分别对应线条下的着色区域和线条。[在这个页面查看详细的SVG色阶介绍。](https://developer.mozilla.org/en-US/docs/Web/SVG/Tutorial/Gradients)

{% highlight javascript %}
function execGradients(yScale) {
  var rangeMax = yScale.invert(0);
  var rangeMin = yScale.invert(height);

  canvas1
    .append('linearGradient')
    .attr('id', 'area-gradient')
    .attr('gradientUnits', 'userSpaceOnUse')
    .attr('x1', 0)
    .attr('y1', yScale(rangeMax))
    .attr('x2', 0)
    .attr('y2', yScale(rangeMin))
    .selectAll('stop')
    .data([
      { offset: '0%', color: '#E5F2D7' },
      { offset: '50%', color: '#EEEEEE88' },
      { offset: '100%', color: '#F3DBE222' }
    ])
    .enter()
    .append('stop')
    .attr('offset', function(d) {
      return d.offset;
    })
    .attr('stop-color', function(d) {
      return d.color;
    });

  canvas1
    .append('linearGradient')
    .attr('id', 'line-gradient')
    .attr('gradientUnits', 'userSpaceOnUse')
    .attr('x1', 0)
    .attr('y1', yScale(rangeMax))
    .attr('x2', 0)
    .attr('y2', yScale(rangeMin))
    .selectAll('stop')
    .data([{ offset: '0%', color: '#97D755' }, { offset: '100%', color: '#D8949B' }])
    .enter()
    .append('stop')
    .attr('offset', function(d) {
      return d.offset;
    })
    .attr('stop-color', function(d) {
      return d.color;
    });
}
{% endhighlight %}

接着，添加`execArea()`函数。`area0`和`area`两个变量，分别用来计算动画初始阶段和结束时的`path`元素的`d`属性，实现整个线图从横坐标逐渐升起的效果。用`style('fill', 'url(#area-gradient)')`方法，设置这个区域的色阶过渡。

{% highlight javascript %}
function execArea(xScale, yScale, data1) {
  var area0 = d3
    .area()
    .x0(function(d) {
      return xScale(d.month);
    })
    .x1(function(d) {
      return xScale(d.month);
    })
    .y0(function() {
      return yScale(0);
    })
    .y1(function(d) {
      return yScale(0);
    })
    .curve(d3.curveCatmullRom.alpha(0.5));
  var area = d3
    .area()
    .x0(function(d) {
      return xScale(d.month);
    })
    .x1(function(d) {
      return xScale(d.month);
    })
    .y0(function() {
      return yScale(0);
    })
    .y1(function(d) {
      return yScale(d.visits);
    })
    .curve(d3.curveCatmullRom.alpha(0.5));

  canvas1
    .append('path')
    .attr('d', area0(data1))
    .style('fill', 'url(#area-gradient)')
    .transition(t)
    .attr('d', area(data1));
}
{% endhighlight %}

添加`execLine()`函数，类似的，它也包含了`line0`和`line`两个变量，对应一头一尾两个状态。

{% highlight javascript %}
function execLine(xScale, yScale, data1) {
  var line0 = d3
    .line()
    .x(function(d) {
      return xScale(d.month);
    })
    .y(function(d) {
      return yScale(0);
    })
    .curve(d3.curveCatmullRom.alpha(0.5));
  var line = d3
    .line()
    .x(function(d) {
      return xScale(d.month);
    })
    .y(function(d) {
      return yScale(d.visits);
    })
    .curve(d3.curveCatmullRom.alpha(0.5));

  canvas1
    .append('path')
    .attr('d', line0(data1))
    .style('fill', 'none')
    .style('stroke', 'url(#line-gradient)')
    .style('stroke-width', '3')
    .transition(t)
    .attr('d', line(data1));
}
{% endhighlight %}

添加坐标轴的代码也非常简单明了。

{% highlight javascript %}
function execAxis(xScale, yScale) {
  var bottomAxis = d3
    .axisBottom()
    .scale(xScale)
    .tickSizeOuter(0)
    .tickSize(0);
  var bottomAxisChart = canvas1
    .append('g')
    .attr('transform', 'translate( 0 ' + yScale(0) + ')')
    .call(bottomAxis);

  bottomAxisChart.selectAll('text').attr('font-size', '1.5em');
}
{% endhighlight %}

再添加`execMouseTracker()`函数，它先添加曲线上指示数值的圆点和数值，然后再添加一根竖直的直线。接着，绑定`mouseover`、`mouseout`和`mousemove`这三个时间的回调函数，分别起到显示、隐藏和移动修改数值指示器的效果。

{% highlight javascript %}
function execMouseTracker(xScale, yScale, data1) {
  var focus = canvas1
    .append('g')
    .attr('class', 'focus')
    .style('display', 'none');
  focus
    .append('circle')
    .attr('id', 'visitsCircle')
    .attr('r', 4.5);
  focus
    .append('text')
    .attr('id', 'visitsText')
    .attr('x', 9)
    .attr('dy', '.35em');

  var verticalPath = d3.line()([[0, -10], [0, height + 10]]);
  focus
    .append('path')
    .attr('d', verticalPath)
    .attr('class', 'verPath')
    .attr('stroke', 'grey')
    .attr('stroke-width', '1');

  canvas1
    .append('rect')
    .attr('class', 'overlay')
    .attr('width', width)
    .attr('height', height)
    .attr('fill', 'rgba(0,0,0,0)')
    .on('mouseover', function() {
      focus.style('display', null);
    })
    .on('mouseout', function() {
      focus.style('display', 'none');
    })
    .on('mousemove', mousemove);
}
{% endhighlight %}

接下来，在`execMouseTracker()`函数中添加`mousemove()`回调函数。函数的第一部分，用于是`invertXScale()`函数，用于将D3捕捉到的鼠标位置，转换为最接近的月份。然后我们计算出该月份对应的纵坐标值。修改指示线和圆圈的`transform`属性，将它们调整到合适的位置。最后修改显示的数值。但月份为12月的时候，为避免数值显示不完整，将其显示在圆圈的左侧，其它月份的数值显示在圆圈的右侧。

{% highlight javascript %}
function mousemove() {
  var invertXScale = function(x) {
    var domain = xScale.domain();
    var range = xScale.range();
    var scale = d3
      .scaleQuantize()
      .domain(range)
      .range(domain);
    return scale(x);
  };
  var xMonth = invertXScale(d3.mouse(this)[0]);
  var xPos = xScale(xMonth);

  var d = monthToNum(xMonth);
  d = data1[d - 1].visits;
  var yPos = yScale(d);

  focus.select('#visitsCircle').attr('transform', 'translate(' + xPos + ',' + yPos + ')');
  focus.select('.verPath').attr('transform', 'translate(' + xPos + ',' + 0 + ')');

  var textOffset = 5;
  if (xMonth == 'Dec.') {
    focus
      .select('#visitsText')
      .attr('transform', 'translate(' + (xPos - 5 * textOffset) + ',' + yPos + ')')
      .attr('text-anchor', 'end')
      .text(d);
  } else {
    focus
      .select('#visitsText')
      .attr('transform', 'translate(' + (xPos + textOffset) + ',' + yPos + ')')
      .attr('text-anchor', 'start')
      .text(d);
  }
}
{% endhighlight %}

最后在HTML页面中，用Javascript代码调用`show()`函数，就大功告成了。

[本示例的完整代码，请在这里查看。](https://github.com/scanthonie/d3-data-visualization-snippets/tree/master/a002)
