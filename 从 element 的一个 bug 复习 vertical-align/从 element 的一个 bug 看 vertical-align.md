# 从 element 的一个 bug 复习 vertical-align



![img](https://cdn.chosan.cn/posts-meta/640-20230119223654060.png)

<p align=center>
    <small>春节马上就要到啦，祝大家新春快乐，幸福美满 </small>
~</p>



最近在画页面的时候发现一个问题，在用 `element` 画表单的时候，如果多个元素包裹在同一个 `el-row` 中将可能出现 float 高度不一致时的元素错位情况，大致表现就像这样：

![错位图片](https://cdn.chosan.cn/posts-meta/%E9%94%99%E4%BD%8D%E5%9B%BE%E7%89%87.png)

实际上`【保护期间】`应该在左侧顶格展示，而不应该变到了第二列。

代码结构大概如下：

```html
<el-form>
    <el-row>
        <el-col :span="8">
            <el-form-item label="合同号">
                <el-select>
                    ...
                </el-select>
            </el-form-item>
        <el-col>
        ... 第二个 col，
        ... 第三个 col，
        <!-- el-row 将一行分为 24 等份，因此从这里开始换行 -->
        <el-col :span="8">
            <el-form-item label="保护期间">
                ...
            </el-form-item>
        </el-col>
        ... 剩下的内容
    </el-row>
<el-form>
```

单纯从表现上看是从`【保护期间】`这里往后挪了一列，但为什么会如此，我很好奇！

## 排查原因

于是打开控制台，查看元素样式，对比发现正常情况下 `el-form-item` 的高度是 `32px`

![高度32px的图片](https://cdn.chosan.cn/posts-meta/height-32.png)

但`【合同号】` 这个地方却是 `32.67px`，多出来 `0.67px`

![高度32.67px](https://cdn.chosan.cn/posts-meta/height-32.67.png)

根据直觉判断，正是这 `0.67px` 把后面一排的元素挤到后面一列，但它是如何产生的呢？

在查看了当前表单项 `el-form-item` 以及它下面的所有子元素的样式之后，发现会影响元素高度的基本只有两个属性，一个是 `height` 另一个是 `line-height`，但 `height` 要么是写死的 `32px` 要么就是 `100%`，`line-height` 基本上全是写死的 `32px`，样式中没有出现小数或者其它通过样式计算会得到小数的值，所以正常情况下不应该会多出来一个 `0.67px` 的高度，看起来不像是 `element` 的样式导致的，为了证明如此，我单独写了段代码来进行测试，[点击这里查看](https://codepen.io/Cinux-Chosan/pen/mdjqBwE)

基本代码如下：

```html
<div class="container">
  <input type="text" class="input" />
</div>
```

```css
input {
  height: 32px;
  background: yellow;
  border: none;
}

.container {
  line-height: 32px;
  margin-top: 50px;
  background: red;
}

/* 以下为重置代码，可忽略 */
* {
  padding: 0;
  margin: 0;
}

html,
body {
  height: 100vh;
}
```

为了方便各位在移动端查看，我添加了相应的背景颜色。

![32px](https://cdn.chosan.cn/posts-meta/32px.png)

![33.67px](https://cdn.chosan.cn/posts-meta/33.67px.png)

内部元素 `input` 实际高度只有 `32 px`，但外面的元素高度却有 `33.67 px`，多了 `1.67 px`，而且这个值会随着内部元素的高度变化 —— 当 `input` 高度变为 `31 px` 时，container 高度为 `33.17 px`，多了 `2.17 px`

![高度 31 像素](https://cdn.chosan.cn/posts-meta/height-31.png)

![高度 33.17 像素](https://cdn.chosan.cn/posts-meta/height-33.17.png)

不仅如此，改变 `font-size` 的值也可能导致 `.container` 的高度变化，这点一开始也将我带入了误区，毕竟 `line-height` 是写死的 `32 px`，是一个完全的绝对值，而非使用 `em` 作为单位的相对值，不应该受到 `font-size` 的影响。

考虑到 `line-height` 是用于设定行高，多出来的高度会在上下进行均分，而 `vertical-align` 可以设置对齐方式，虽然这里 `.container` 的 `line-height` 和 `input` 的 `height` 相同，按道理没有多余的高度，但还是习惯性的尝试了一下设置 `input` 元素的 `vertical-align`，发现将 `input` 的 `vertical-align` 设置为 `bottom` 或者 `top` 后内外高度保持一致，并且设置其它值还可能导致 `.container` 出现不同的高度。

到这里我已经基本清楚原因了，但不得不说一下 `vertical-align` 相关的点：

默认情况下，具有 `inline` 属性的元素会相对于基线对齐其底部，什么是基线，一般来说是小写 `x` 的底线，在设计文字的时候的一个参考线：

![vertical-align-basic](https://cdn.chosan.cn/posts-meta/vertical-align-basic.png)

但基线并不是所有字符的底线，某些字符可能会穿过基线，一部分在下面一部分在上面（就像上图的字母 `j`），从而保持一行文字在视觉上的整体平衡，但字符会出现在顶线和底线之间，实际上顶线和底线的距离可以用 `font-size` 来表示，而行高大于 `font-size` 时，将会均匀分布在顶线和底线之外。

之所以出现本文前面所描述的多出来的 `0.67px`，就是因为 `el-form-item` 中的 `select` 内部使用了 `input` 元素来模拟下拉框，而 `input` 本身是一个 `inline-block`，具有 `inline` 属性，因此它相对于基线进行对齐，但基线到底线之间还有一点距离，这个是在文字设计阶段就存在的，当 `.container` 元素中没有内容时，整体高度为 0，但如果其中有具有 `inline` 属性的内容时，就能观察到这种现象，实际上浏览器插入了一个宽度为 `0` 的字符，某些地方称之为 “幽灵节点”。

另外，张鑫旭在其博文中也有关于这种情况更详细的介绍，参考 [CSS 深入理解 vertical-align 和 line-height 的基友关系 —— 张鑫旭](https://www.zhangxinxu.com/wordpress/2015/08/css-deep-understand-vertical-align-and-line-height/)

## 解决方式

要解决这个问题，可以从以下方面来做：

- `font-size` 设置为 `0`，从而基线到底线的距离也变为了 `0`。
- 去除 `inline` 属性，例如将 `inline-block` 元素改成 `block` 元素，或者 `float` 也会强制元素变成 `block`。
- 设置 `vertical-align` 为 `bottom` 或者 `top` 都可以解决，它本质上是调整了元素的对齐方式，从而绕过基线对齐带来的问题。（在解决底部空白的时候设置为 `middle` 也是可以的，但并不能解决上文中的问题，因为 `middle` 并不是垂直方向上真正的 `中点`）


## 总结

关于 `vertical-align` 所涉及到的知识点实际上是一个面试中老生常谈的问题，其实在早些年刚开始工作那会儿就已经从某些书籍上了解过包括基线、幽灵节点在内的相关概念，但不得不说，如果没有培养这方便的意识，很容易被这种奇怪的问题绕进去。通过这次的问题排查，也算是巩固了以前所学的知识并且培养了这方面的意识，也算小有收获。

另外，虽然这不是 `element` 的问题，但确实造成了 `element` 的一个样式 bug。项目中使用的 `element` 版本为 `2.14`，相对来说比较老，也许在新的版本中已经不存在这样的情况了。

