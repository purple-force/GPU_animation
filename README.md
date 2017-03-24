# GPU_animation


> 最近看到一篇关于GPU动画的神文，原文地址：https://www.smashingmagazine.com/2016/12/gpu-animation-doing-it-right/。
特此翻译出来，供自己以及他人查看。


### 如今，绝大多数人知道现代浏览器采用GPU来渲染网页的部分内容，尤其是动画部分。比如，采用`transform`属性的CSS动画看起来比使用`left`和`top`属性的动画更流畅。但如果你要问：“我如何利用GPU实现流畅的动画？”绝大多数情况下，你会听到如下回答：“使用`transform: translateZ(0)`或者`will-change: transform`。” 

在开始GPU动画-或者**合成**（compositing，浏览器厂商喜欢这么称呼）之前，从某种意义上来说，这些个属性就有点像IE6中使用的`zoom:1`一样。（译者注：有点拗口，意思应该是许多人只知道这样设置就行了，但是并不知道具体的原因）

但有时候，简单demo中丝滑流畅的动画，在实际网站中运行非常慢，造成视觉假象，甚至让浏览器崩溃。为什么会这样？**如何修复？**让我们来了解一下。

### 免责声明

在深入研究GPU合成之前，我想告诉你们一件十分重要的事：这是一个**大大的hack**。至少到目前为止，合成的工作原理，如何明确地将元素置于合成层，或者合成本身，关于这些问题，你在[W3C](https://www.w3.org/)规范上找不到任何答案。它只是浏览器执行特定任务时的一种优化操作，每个浏览器厂商有自己的实现方式。

本篇文章中你学到的所有东西，并不是合成原理的官方解释，而是我实验的结果，加上一点对浏览器子系统差异的常识和理解。有些东西可能是错的，有些可能随着时间而变化——提醒过你了。

### 合成如何工作

在开始GPU动画页面之前，我们得知道浏览器是如何工作的，不要简单地听从网上的或者本篇文章中的一些随意的建议。

假设我们有一个页面，其中有`A`和`B`两个元素，每一个都设置了`position: absolute`和不同的`z-index`值。浏览器会用CPU绘制，之后把完成的图像发送给GPU，由它显示在屏幕上。

```html
<style>
#a, #b {
 position: absolute;
}

#a {
 left: 30px;
 top: 30px;
 z-index: 2;
}

#b {
 z-index: 1;
}
</style>
<div id="a">A</div>
<div id="b">B</div>
```
<iframe src="https://sergeche.github.io/gpu-article-assets/examples/example1.html" height="280" frameborder="no" allowtransparency="true" style="width: 100%;"></iframe>

我们决定采用`left`属性和CSS动画，让`A`元素动起来：

```html
<style>
#a, #b {
 position: absolute;
}

#a {
 left: 10px;
 top: 10px;
 z-index: 2;
 animation: move 1s linear;
}

#b {
 left: 50px;
 top: 50px;
 z-index: 1;
}

@keyframes move {
 from { left: 30px; }
 to { left: 100px; }
}
</style>
<div id="a">A</div>
<div id="b">B</div>
```
<iframe src="https://sergeche.github.io/gpu-article-assets/examples/example1.html#.a:anim-left" height="280" frameborder="no" allowtransparency="true" style="width: 100%;"></iframe>

此种情况下，对于每个动画帧，浏览器都必须重新计算元素的位置（即reflow），渲染页面新状态的图像（即repaint），之后再发送给GPU显示到屏幕上。我们知道，重绘是非常消耗性能的，但是每个现代浏览器都足够智能，只重绘页面中变化的部分，而不是整个页面。尽管绝大多数情况下，浏览器可以很快地重绘，但我们的动画仍然不是太流畅。

在动画的每个阶段回流、重绘整个页面（即便是增量绘制），听起来就很慢，尤其是又大又复杂的布局。仅绘制两个独立的图像可能更高效——一个为A元素，另一个为A元素以外的整个页面——之后简单地偏移两个图片的相对位置。也就是说，**合成**缓存元素的图像可能更快。这就是GPU的优势所在：它能够以**亚像素精度**快速合成图像，使得动画如丝般顺滑。

为了优化合成，浏览器必须确保添加动画的CSS属性：
+ 不会影响文档流，
+ 不依赖文档流，
+ 不会造成重绘。

有人可能会以为，`top`和`left`属性，辅之以`position`为`absolute`或`fixed`，不依赖元素的环境，但其实并不是这样。例如，`left`属性可能是个百分比值，其依赖于`.offsetParent`的尺寸；另外，`em`，`vh`和其它单位依赖于它们的环境。相反，`transform`和`opacity`是仅有的满足上述条件的CSS属性。

让我们用`transform`而不是`left`来实现动画：

```html
<style>
#a, #b {
 position: absolute;
}

#a {
 left: 10px;
 top: 10px;
 z-index: 2;
 animation: move 1s linear;
}

#b {
 left: 50px;
 top: 50px;
 z-index: 1;
}

@keyframes move {
 from { transform: translateX(0); }
 to { transform: translateX(70px); }
}
</style>
<div id="a">A</div>
<div id="b">B</div>
```

此处，我们以**声明**的方式描述动画：起始位置，结束位置，持续时间等。这等于提前告诉浏览器哪些CSS属性会更新。因为浏览器发现没有属性会造成回流或者重绘，它就会采用合成优化：画两幅图像作为**合成层**，之后发送到GPU。

这种优化的优点是什么呢？

+ 我们获得了一个亚像素精度的、如丝般顺滑的动画，运行在专门为图形任务优化的单元上。并且运行得非常快。
+ 动画再也不受限于CPU。即使运行繁重的JavaScript任务，动画依然很快。

一切听起来似乎简单明了，不是吗？我们会遇到哪些问题？让我们看看这种优化的工作原理。

GPU是一个**独立的计算机**，这可能让你觉得吃惊。确实如此：每个现代设备必不可缺的部分是一个独立的单元，它有自己的处理器和内存、数据处理模块。如同其它应用和游戏一样，浏览器必须与GPU进行通信，好像和外设一样。

为了更好地理解其工作原理，想像下Ajax。假设你想用填写的表单数据注册网站用户。你不能简单地告诉远端的服务器，“嗨，把这些表单数据和JavaScript变量保存到数据库中。”远端数据库无法访问用户浏览器的内存。反而，你必须把页面中的数据以易解析的格式（比如JSON），收集在一个payload中，然后发给远端服务器。

合成的过程也差不多。GPU就像个远端的服务器，浏览器必须先创建一个payload，之后再发送给GPU。显然，GPU不是远离CPU千里之外；它就在那。在很多情况下，对远端服务器的请求和响应间隔时间在2S内是可以接受的。而对于GPU，3到5毫秒的延迟却能导致动画卡顿。

GPU payload长什么样？一般由层图像组成，还有一些附加的说明，比如层的尺寸，偏移，动画参数等。以下是GPU的payload生成和传输的大概过程：
+ 将每个合成层绘制为独立的图像
+ 准备层数据（尺寸，偏移，不透明度等）
+ 为动画准备着色器（如果可用的话）
+ 发送数据给GPU

如你所见，每次给元素添加神奇的`transform: translateZ(0) `或者`will-change: transform`属性时，都开启了同样的过程。然而重绘是非常耗性能的，此时会变得更慢。多数情况下，浏览器无法增量重绘。它必须用新创建的合成层绘制之前覆盖的区域：
<iframe src="https://sergeche.github.io/gpu-article-assets/examples/before-after-compositing.html" height="270" frameborder="no" allowtransparency="true" style="width: 100%;"></iframe>

### 隐式合成

让我们回到之前`A` `B`元素的例子。早先，我们将`A`做成动画，它处在页面所有元素之上。这会生成两个合成层：`A`元素一个，`B`元素和页面背景一个。
现在，我们让`B`元素动起来:
<iframe src="https://sergeche.github.io/gpu-article-assets/examples/example3.html#.b:anim-translate" height="280" frameborder="no" allowtransparency="true" style="width: 100%;"></iframe>
我们遇到了一个逻辑问题。元素`B`应该在一个独立的合成层，屏幕最终呈现的图像应该在GPU中合成。但是`A`元素应该出现在`B`元素上面，并且我们没有指定`A`提升到自己的层。

记得之前的**免责声明**：GPU合成模式并不是CSS规范的一部分；它只是浏览器内部使用的一种优化策略。如`z-index`定义的那样，我们强制`A`出现在`B`的上面。那么，浏览器会怎么做呢？

猜对了！浏览器会强制为`A`创建新的合成层——当然，增加了一次繁重的重绘：
<iframe src="https://sergeche.github.io/gpu-article-assets/examples/example4.html#.b:anim-translate" height="280" frameborder="no" allowtransparency="true" style="width: 100%;"></iframe>

这称为**隐式合成**：按照栈顺序，一个或多个非合成元素出现在合成元素上面时，会被提升到合成层——即被绘制成独立的图像发送到GPU中。

我们遇到隐式合成的情况比你想象的要频繁的多。浏览器会因很多原因将一个元素提升为合成层，比如：
+ 3D 变换：`translate3d`，`translateZ `等等；
+ `<video>`、`<canvas>`和`<iframe>`元素；
+ 通过`Element.animate()`实现的`transform`和`opacity`动画；
+ 通过CSS transition animation实现的`transform`和`opacity`动画；
+ `position: fixed`；
+ `will-change`；
+ `filter`；

更多情况请参考Chromium项目的“[CompositingReasons.h](https://cs.chromium.org/chromium/src/third_party/WebKit/Source/platform/graphics/CompositingReasons.h?q=file:CompositingReasons.h)”文件。

似乎GPU动画的主要问题是意想不到的大量重绘。但并不是。最大的问题是。。。

### 内存消耗

再一次温馨提示：GPU是独立的计算机：它不仅需要发送渲染好的图片给GPU，而且需要对其进行存储，以便后续动画复用。

一个合成层需要消耗多少内存？让我们看个简单点的例子。猜猜存储一个320×240像素，填满`#FF0000`颜色的长方形需要多少内存。
<iframe src="https://sergeche.github.io/gpu-article-assets/examples/rect.html" height="270" frameborder="no" allowtransparency="true" style="width: 100%;"></iframe>

一个标准的web开发者这样想：“嗯，这是个纯色的图像，我会将其保存为PNG然后查看其大小。应该小于1KB”。没错，这个PNG图片大概104字节。

问题是，PNG以及JPEG，GIF等，用来存储和传输图像数据。为了将这样的图像绘制到屏幕上，计算机必须解压图像数据，然后表示成像素数组。因此，我们的样图会消耗`320 × 240 × 3 = 230,400 bytes`的内存。也就是，图片宽度乘以高度获得图片的像素数。之后再乘以3，因为每个像素由3个字节描述（RGB）。如果图片包含透明通道，就得乘以4，因为附加的一个字节用来描述透明度（RGBa）：`320 × 240 × 4 = 307,200 bytes`。

浏览器总是按照RGBa图像的形式绘制合成层。似乎没有行之有效的方法来确定图片是否包含了透明通道。

再看一个更常见的例子：一个有10张图的旋转盘，每张图800×600像素。我们希望用户交互，比如拖拽时，图片之间能够平滑过渡，因此，我们为每幅图添加`will-change: transform`。这会提前将图片提升到合成层，因此，用户一开始交互时，过渡就会开始。现在计算下仅仅展示这一旋转盘需要多少额外内存：800 × 600 × 4 × 10 ≈ **19 MB**。

仅仅一个控制点就需要额外19MB内存！如果你是一个单页应用的WEB开发者，页面中有多个动画控制点，视差效果，高分辨率图像和其它视觉增强效果，那么每个页面多增加100到200MB仅仅是个开始。再考虑上隐式合成的话（承认吧——你之前根本没想过这个），最终页面会耗尽设备的内存。

此外，多数情况下，这些内存会被浪费掉，用来显示同样的结果：
<iframe src="https://sergeche.github.io/gpu-article-assets/examples/example5.html" height="620" frameborder="no" allowtransparency="true" style="width: 100%;"></iframe>

对于桌面客户端来说，这可能不是个问题，但会深深刺痛移动用户的心。首先，绝大多数现代设备拥有高分辨率的屏幕：这就将合成层图片的体量乘以4到9。其次，移动设备不像桌面设备那样有那么大的内存。比如，不是太旧的iPhone6仅搭载1GB共享内存（即，内存同时用于RAM和VRAM）。考虑到至少三分之一的内存用于操作系统和后台进程，另外的三分之一用于浏览器和当前页面（最好的情况是高度优化的页面，没有太多的framework），我们至多剩下200到300MB供GPU渲染。并且iPhone6是个相当昂贵的高端设备，更多平价的手机所搭载的内存更少。

你也许会问：“有可能在GPU上存储PNG图片来减少内存占用吗？”技术上是可行的。唯一的问题是GPU在屏幕上是[逐像素绘制](http://www.html5rocks.com/en/tutorials/webgl/shaders/)的，这意味着它必须一次次地解码整个PNG图片来获取每个像素数据。我怀疑这种情况下的动画比每秒一帧快点。

GPU特定的[图像压缩格式](https://en.wikipedia.org/wiki/Texture_compression)确实存在，但毫无意义。从压缩比来看，根本比不上PNG或者JPEG，并且使用上也缺乏硬件支持。

### 优缺点

既然学了些GPU动画的基本原理，让我们总结下它的优缺点：

**优点**
+ 动画既快又流畅，达到每秒60帧。
+ 精心制作的动画在独立的线程中运行，不会被繁重的JavaScript计算阻塞
+ 3D变换很“廉价”

**缺点**
+ 需要附加的重绘来将元素提升到合成层。有时这个过程很慢（比如，进行全层重绘，而不是增量重绘）。
+ 绘制的层必须传到GPU中。依据层的大小和数量，传输可能很慢。这可能导致中低端设备上元素闪烁。
+ 每个合成层消耗额外的内存。在移动设备上，内存是宝贵的资源。内存超标使用会使**浏览器崩溃**。
+ 如果你不考虑隐式合成，重绘缓慢、额外内存使用和浏览器崩溃的可能性会很高。
+ 我们会看到视觉假象，比如Safari上文本渲染，在某些情况下，页面内容会消失或者混乱。

如你所见，尽管有些独特的优势，GPU动画仍然有些令人讨厌的问题。最重要的是重绘和大量的内存消耗；因此，以下所有的优化策略都是处理这些问题的。

### 浏览器设置

在开始优化之前，我们得学习一些工具，来帮助我们检查页面的合成层，并且提供合理的优化反馈。

**Safari**

Safari的web 监视器有个很棒的“Layers”边条，它显示所有的层以及内存消耗，以及合成的**原因**。来看看这个边条：

1. 在Safari中，按`⌘ + ⌥ + I`打开web监视器。如果不起作用，打开“Preferences” → “Advanced”，开启“Show Develop Menu in menu bar”选项，再试一次。
2. 当web监视器打开后，选择“Elements”面板，选择右边条的“Layers”。
3. 现在，当你在主“Elements”上点击一个DOM元素时，你会看到一个关于选择元素以及所有后代合成层的信息层（如果使用了合成的话）。
4. 点击一个后代层，查看其合成原因。浏览器会告诉你为什么决定把这个元素迁移至它自己的合成层。
![](https://www.smashingmagazine.com/wp-content/uploads/2016/11/safari-large-opt.png)


**Chrome**

Chrome的开发者工具栏有个类似的面板，但你必须首先激活它：
1. 在Chrome中，访问`chrome://flags/#enable-devtools-experiments`，之后启用“Developer Tools experiments”项。
2. 用`⌘ + ⌥ + I`(Mac)或者`Ctrl + Shift + I`(PC)打开工具栏，之后点击右上角的如下图标，选择“Setting”菜单项：
![](https://www.smashingmagazine.com/wp-content/uploads/2016/11/devtools-icon-opt.png)
3. 回到“Experiments”面板，启用“Layers”面板。
4. 重新打开开发者工具栏。现在，你就能看到“Layers”面板了。
![](https://www.smashingmagazine.com/wp-content/uploads/2016/11/chrome-large-opt.png)

这个面板以树的形式展示当前页面所有活动的合成层。当选择一个层的时候，你会看到诸如尺寸，内存消耗，重绘次数和合成原因。

### 优化建议

现在我们已经设置好环境，可以开始优化合成层了。我们已经确定了合成的两个主要问题：额外的重绘，也会造成数据数据传送到GPU，还有额外的内存消耗。因此，以下所有的优化建议都是针对这两个问题的。

**避免隐式合成**

这是最简单明了的建议，同样也十分重要。提醒你一下，处在一个显式合成层（比如`position: fixed`，视频，CSS动画等）之上的所有非合成DOM元素，会被强制提升到自己的层，仅仅为了最后的GPU图像合成。在移动设备上，可能会导致动画启动缓慢。

举个简单的例子：

<iframe height="305" scrolling="no" src="https://codepen.io/sergeche/embed/jrZZgL/?height=305&amp;theme-id=light&amp;default-tab=result&amp;embed-version=2" frameborder="no" allowtransparency="true" allowfullscreen="true" style="width: 100%;"></iframe>

`A`元素是个需要用户交互启动的动画。如果你在“Layers”面板中查看这个页面，你会发现没有多余的层。但当点击“Play”按钮后，你会看到更多的层，这些层在动画结束后立即被移除。如果看下“Timeline”面板，会发现动画的开始和结束位置充斥大片区域的重绘：
![](https://www.smashingmagazine.com/wp-content/uploads/2016/11/chrome-timeline-large-opt.png)

以下是浏览器所做的，一步接一步：
1. 页面加载完成后，浏览器找不到任何合成的理由，因此选择了最优的策略：在单个背景层上绘制整个页面内容。
2. 点击“Play”按钮，我们给元素`A`显式增加了合成层——`transfrom`属性的一个变换。但是浏览器发现按照栈顺序，元素`A`在元素`B`下面，因此也将`B`提升到自己的合成层（隐式合成）。
3. 提升到合成层总会造成一次重绘：浏览器必须为元素创建一个新的纹理，然后从前一个层中移除掉。
4. 新层图像必须发送到GPU中，用来合成用户最终看到图像。依层的数量、纹理尺寸和内容复杂度的不同，重绘和数据传输可能花许多时间。这就是许多动画在开始和结束时出现元素闪烁的原因。
5. 动画结束一刹那，我们从元素`A`上移除了合成的原因。此时，浏览器发现不需要浪费资源来进行合成，因此很快回到最优策略：将页面的整个内容绘制在一个背景层当中，这意味着必须把`A`和`B`重绘回背景层当中（另一次重绘），之后把更新的纹理发送给GPU。如上步骤，可能造成闪烁。

为了免受隐式合成问题的困扰，减少视觉假象，有如下建议：
+ 给予动画元素尽可能高的`z-index`。理想情况下，这些元素应该是`body`元素的直接子元素。当然，动画元素在DOM树中嵌入很深、并且依赖常规流时，这是不大可能的。在此种情况下，你可以克隆该元素，将其放置到body中仅作动画之用。
+ 你可以利用`will-change`CSS属性给浏览器一个提示，表明你要使用合成。将这个元素设置在元素上，浏览器会（并不总是）将其提前提升到一个合成层中，因此动画能够流畅地启动和停止。但别滥用这个属性，否则最终会导致内存的急剧消耗！

**仅将`tranform`和`opacity`属性动画化**

`tranform`和`opacity`属性能够确保既不影响也不会被常规流或者DOM环境影响（也就是说，它们不会造成回流或者重绘，因此动画完全交由GPU渲染）。基本上，这意味着你可以高效地实现移动、缩放、旋转、透明变换动画，并且只有仿射变换。有时，你可以用这些属性模拟其它动画类型。

举个非常常见的例子：背景色变换。基本方法是添加一个`transition`属性：
```html
<div id="bg-change"></div>
<style>
#bg-change {
 width: 100px;
 height: 100px;
 background: red;
 transition: background 0.4s;
}

#bg-change:hover {
 background: blue;
}
</style>
```
在这个例子中，动画完全运行在CPU中，动画的每个阶段都会重绘。但我们可以让动画运行在GPU上。我们可以在上面添加一层，将其不透明度动画化，而不是`background-color`属性：
```html
<div id="bg-change"></div>
<style>
#bg-change {
 width: 100px;
 height: 100px;
 background: red;
}

#bg-change::before {
 background: blue;
 opacity: 0;
 transition: opacity 0.4s;
}

#bg-change:hover::before {
 opacity: 1;
}
</style>

```
这个动画会更快、更流畅。但记住，可能会引起隐式合成和额外的内存消耗。然而此种情况下，可以极大减少内存消耗。

**减少合成层的大小**

看下面两张图，看到区别了吗？

<iframe src="https://sergeche.github.io/gpu-article-assets/examples/layer-size.html" height="130" frameborder="no" allowtransparency="true" style="width: 100%;"></iframe>

这两个合成层**从视觉上来看是一样的**，但第一个有40，000字节（30KB），第二个仅仅400字节——小了100倍。为什么？看下代码：
```html
<div id="a"></div>
<div id="b"></div>

<style>
#a, #b {
 will-change: transform;
}

#a {
 width: 100px;
 height: 100px;
}

#b {
 width: 10px;
 height: 10px;
 transform: scale(10);
}
</style>
```
差别在于物理尺寸，`#a`为100×100像素（100×100×4=40000bytes），而`#b`仅为10×10像素（10×10×4=400bytes），但通过`transform: scale(10)`缩放到100×100像素。因为`#b`是一个复合层，由于`will-change`属性，`transform`在最终的图像绘制过程中，将完全在GPU中进行。

手法很简单：通过`width`和`height`属性减少合成层的物理大小，之后通过`transform: scale(…)`放大纹理。当然，这种把戏只能减少非常简单的、纯色层的内存消耗。但是，举个例子，如果你想为一个大的照片创建动画，可以减少5%到10%的大小，之后放大；用户可能看不出任何差别，而你可以节省好几兆宝贵的内存。

**尽可能地使用CSS 变换和动画**

我们知道，通过CSS transform和animation的`transform`和`opacity`动画会自动创建合成层，并且运行在GPU上。我们也可以通过JavaScript实现动画，但为了元素获取自己的合成层，必选先添加`transform: translateZ(0)`或`will-change: transform, opacity`。
JavaScript动画的每一步是在`requestAnimationFrame`回调函数中手动计算的。通过`Element.animate()`实现的动画是声明式CSS动画的变体。

一方面，通过CSS transition和animation创建简单可复用的动画非常容易；另一方面，创建包含漂亮轨迹的复杂动画时，JavaScript动画又比CSS动画容易实现。另外，JavaScript是和用户输入交互的唯一方式。

哪一个更好？我们可以只用一个通用的JavaScript动画库来实现所有动画吗？

基于CSS的动画有个很重要的特性：**完全在GPU上运行**。因为你**声明了**动画如何开始和结束，浏览器可以赶在动画开始之前，准备好所需要的所有指令，之后发送给GPU。在**必须使用**JavaScript的情况下，浏览器所知的只有当前帧的状态。对一个流畅动画而言，我们必须以每秒60次的速度在浏览器主线程中计算好新帧，然后发送给GPU。这些计算和数据发送不仅比CSS动画慢，同时也依赖于主线程的工作负载：
<iframe src="https://sergeche.github.io/gpu-article-assets/examples/js-vs-css.html" height="180" frameborder="no" allowtransparency="true" style="width: 100%;"></iframe>

在上面的例子当中，当主线程被繁重的JavaScript任务阻塞的时候，你会看到发生了什么。CSS动画不受影响，因为新帧是在独立的线程上计算的，而JavaScript动画必须等到繁重的计算完成，之后才计算新帧。

因此，试着尽可能使用基于CSS的动画，尤其是加载和进度指示条。不仅更快，而且还不会被大量的JavaScript计算阻塞。

### 现实世界中优化的例子

本篇文章是我在为 [Chaos Fighters](https://ru.4game.com/chaos-fighters/)开发页面时的研究和实验结果。这是个响应式的手机游戏促销页面，有大量的动画。当开始开发的时候，我唯一所知的就是如何实现基于GPU的动画，但我并不知其工作原理。因此，在最初的里程碑页，就造成了iPhone5——当时最新的Apple手机——在页面加载完几秒钟后崩溃了。现在，这个页面运行良好，即使是在性能稍弱的设备上。

按照我的观点，让我们考虑下这个网站中最有趣的优化部分。

页面的最顶端是游戏的介绍，有个像太阳光线东西在背景上旋转。这是个无线循环、非交互的旋转盘——正适合用简单的CSS动画实现。首先想到的方案（错误尝试）是保存太阳光线的图片，将它放在`img`中，之后使用无限CSS动画：
<iframe width="350" height="402" scrolling="no" src="https://codepen.io/sergeche/embed/gwBjqG/?height=402&amp;theme-id=light&amp;default-tab=result&amp;embed-version=2" frameborder="no" allowtransparency="true" allowfullscreen="true"></iframe>

似乎如预期的那样万事大吉。但是太阳的图片相当大。移动用户可能不开心了。

再仔细看下图片。只是从图片中心发出来几道光线而已。光线是一样的，因此我们可以保存单个光线，复用它来实现最终的图片。最后，我们仅用了一个单光线的图片，相比刚开始的图片，大小少了一个数量级。

对于这种优化，我们的标记语言就必须复杂一点了：`.sun`是光线图片元素的容器。每条光线在特定的角度旋转。
<iframe width="350" height="402" scrolling="no" src="https://codepen.io/sergeche/embed/qaJraq/?height=402&amp;theme-id=light&amp;default-tab=css&amp;embed-version=2" frameborder="no" allowtransparency="true" allowfullscreen="true"></iframe>

视觉效果是一样的，但是网络传输的数据量会少得多。另外，合成层的大小保持不变：500 × 500 × 4 ≈ 977 KB。

为了保证简单，例子中太阳光线的大小是相当小的，只有500 × 500像素。在实际网站中，服务于不同尺寸的设备（手机、平板和桌面电脑）和不同分辨率，最终图片的大小大约000 × 3000 × 4 = 36 MB！而这仅仅是页面中的一个动画元素。

在“Layers”面板中再看下页面的元素。通过旋转整个太阳容器，使得动画实现更容易。因此，这个容器被提升到一个合成层，被绘制进一个大的纹理图像中，之后发送给GPU。但是由于我们的简化，现在纹理中包含**无用的数据**：光线之间的间隔。

此外，无用的数据比有用的数据大很多！这不是利用有限内存资源的最佳方式。

解决办法和我们优化网络传输时一样：只发送有用的数据（即光线）给GPU。我们可以计算下节约多少内存：

+ 整个太阳容器：500 × 500 × 4 ≈ 977 KB
+ 12个太阳光线：250 × 40 × 4 × 12 ≈ 469 KB

内存消耗可以减少一倍，为实现这一方案，我们必须**为每个光线单独实现动画**，而不是整个容器。因此，只有光线图像会被发送到GPU当中；它们之间的间隔不会占用任何资源。

为了实现独立的光线动画，标记语言已经有点复杂了，此处的CSS更是一个障碍。我们已经为光线的初始旋转使用了`transform`，必须从同样的角度启动动画，然后旋转360度。基本上，我们得为每个光线分别实现一个`@keyframes`，这是个不小的网络传输。

光线的初始放置和微调动画，光线数量等，写个简短的JavaScript来处理这些问题会更容易。

<iframe width="350" height="402" scrolling="no" src="https://codepen.io/sergeche/embed/bwmxoz/?height=402&amp;theme-id=light&amp;default-tab=js&amp;embed-version=2" frameborder="no" allowtransparency="true" allowfullscreen="true"></iframe>

新的动画看起来和前一个一样，但是内存消耗只有一半。

还没结束。从布局合成的角度来说，这个太阳动画不是主元素，而是一个背景元素。并且光线没有鲜明的对比元素。这意味着我们可以发送一个低分辨率的光线纹理给GPU，之后放大它，这可以节省一部分内存。
我们试着减少10%的纹理大小。光线的物理尺寸为50 × 0.9 × 40 × 0.9 = 225 × 36 像素。为了使它看起来和250 × 20一样，我们需要放大250 ÷ 225 ≈ 1.111倍。

我们会在代码中加一行：给`.sun-ray`加上`background-size: cover `——这样背景图就会自动调整到元素的大小，并且为光线的动画添加`transform: scale(1.111)`。
<iframe width="350" height="402" scrolling="no" src="https://codepen.io/sergeche/embed/YGJOva/?height=402&amp;theme-id=light&amp;default-tab=js&amp;embed-version=2" frameborder="no" allowtransparency="true" allowfullscreen="true"></iframe>

注意，我们只改变了元素的大小；PNG图片的大小仍然一样。由DOM元素创建的矩形被渲染成纹理供GPU使用，而不是PNG图片。

在GPU中，太阳光线的新合成大小现在为225 × 36 × 4 × 12 ≈ 380 KB（原来是469KB）。我们已经减少了19%的内存消耗，并且实现了非常灵活的代码，可以通过缩放来实现最优的质量内存比。因此，通过提高动画（起先看起来很简单）的复杂度，我们减少了内存使用量977 ÷ 380 ≈ 2.5 倍！

我想你已经发现了这个方法的缺陷：动画现在工作在CPU上，可能被大量的JavaScript计算阻塞。如果你想更熟悉优化GPU动画，我留个小小的家庭作业。Fork[Codepen of the sun rays](https://codepen.io/sergeche/pen/YGJOva)，然后将太阳光线完全转移到GPU上运行，然而还要和初始的例子一样节省内存和灵活。将你的例子提交到注释中，我会回复你的。

### 获得的教训

优化Chaos Fighters页面的研究使我完全重新思考开发现代web页面的过程。以下是我的主要原则：

+ 一定要和客户端和设计者沟通网站上所有的动画和效果。这可能极大地影响页面的标记语言，并且也有利于更好地合成。
+ 从一开始就要注意合成层的大小和数量——尤其是隐式合成层。浏览器开发者工具中的“Layers”面板是你最好的朋友。
+ 现代浏览器大量运用合成，不仅仅是动画，还有优化页面元素的绘制。比如，`position: fixed`和`iframe`、`video`元素也使用合成。
+ 合成层的大小可能比数量更重要。在某些情况下，浏览器会试图减少合成层的数量（参见“[GPU Accelerated Compositing in Chrome](https://www.chromium.org/developers/design-documents/gpu-accelerated-compositing-in-chrome)”中的“Layer Squashing”一节）；这会阻止所谓的“层爆炸”和减少内存消耗，尤其是当层有大量的交集时。但有时，这种优化有副作用，比如当一个很大的纹理消耗的内存比多个小层多时。为了避免这种优化，我给每个元素加了个小的、唯一的`translateZ()`值，比如`translateZ(0.0001px)`，`translateZ(0.0002px)`等。浏览器会认为处在3D空间的不同层，从而跳过优化。
+ 为了从视觉上提高动画的性能或者避免视觉假象，你不能只是简单地给任何元素添加`transform: translateZ(0)`或者`will-change: transform`。GPU合成有许多缺点和权衡需要考虑。使用不当时，可能会降低整体的性能，甚至导致浏览器崩溃。

请允许我再提醒下免责声明：关于GPU合成，没有任何官方规范，每个浏览器厂商解决同一个问题的方案不尽相同。本篇文章中的某些部分几个月后可能就过时了。比如，Google Chrome 开发者正在想方法减少CPU和GPU之间数据传输的开销，包括使用特殊的共享内存，这样就没有开销了。另外，Safari已经能够将简单元素的绘制（比如具有`background-color`的空DOM元素）代理到GPU，而不是在CPU上为其创建图像。

无论如何，我希望本篇文章已经帮助你更好地理解浏览器采用GPU渲染的原理，从而帮你创建在各种设备上都能快速运行的令人难忘的网站。

