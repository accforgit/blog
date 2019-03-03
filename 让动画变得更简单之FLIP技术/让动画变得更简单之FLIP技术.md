某次被问到如何实现以下动画效果：

![img](card_2.gif)

若干个元素卡片从上而下排列，当增加或删除某个卡片的时候，其余的卡片会以一种 `transition`动画的形式移动到适当的位置上，而不是生硬地闪现

当时我恰好看过 `Vue`中的内置组件 `transition`的实现，意识到完全可以用 `transition`组件的部分原理来完成这个效果，但是由于没有深入地探究过为什么是这样，只停留在表面，知其然而不知其所以然，所以尽管我知道如何实现这个效果，但很难解释为什么是这样，语言组织地比较困难

后来我无意间看到一篇文章 [FLIP技术给Web布局带来的变化](https://www.w3cplus.com/javascript/animating-layouts-with-the-flip-technique.html)，立马恍然大悟，原来这个东西叫 **FLIP**

## FLIP

**FLIP**是 `First`、`Last`、`Invert`和 `Play`四个单词首字母的缩写

`First`，指的是在任何事情发生之前（过渡之前），记录当前元素的位置和尺寸，即动画开始之前那一刻元素的位置和尺寸信息，可以使用 `getBoundingClientRect()`这个 `API`来处理（大部分情况下其实 `offsetLeft`和 `offsetTop`也是可以的）

`Last`：执行一段代码，让元素发生相应的变化，并记录元素在动画最后状态的位置和尺寸，即动画结束之后那一刻元素的位置和尺寸信息

`Invert`：计算元素第一个位置（`First`）和最后一个位置（`Last`）之间的位置变化（如果需要，还可以计算两个状态之间的尺寸大小的变化），然后使用这些数字做一定的计算，让元素进行移动（通过 `transform`来改变元素的位置和尺寸），从而创建它位于第一个位置（初始位置）的一个错觉

即，一上来直接让元素处于动画的结束状态，然后使用 `transform`属性将元素反转回动画的开始状态（这个状态的信息在 `First`步骤就拿到了）

`Play`：将元素反转（假装在`first`位置），我们可以把 `transform`设置为 `none`，因为失去了 `transform`的约束，所以元素肯定会往本该在的位置（即动画结束时的那个状态）进行移动，也就是`last`的位置，如果给元素加上 `transition`的属性，那么这个过程自然也就是以一种动画的形式发生了

按照我的理解，就是对动画元素起止状态的一个量化，量化成一个公式，绝大部分的连续动画都可以通过套用这个公式来完成，提升动画的开发效率，更加详细的请自行参见 [FLIP技术给Web布局带来的变化](https://www.w3cplus.com/javascript/animating-layouts-with-the-flip-technique.html)

## 实现卡片 Card增删动画

了解了 `FLIP`这个概念之后，再来实现开头提到的那个动画效果，其实就很简单了

**First**

记录在动画开始之前每个卡片的位置和尺寸信息，这里因为卡片的尺寸在动画过程中其实是不会发生任何变化的， 所以可以略过这一步，只记录卡片的位置信息

另外，如果所有卡片的尺寸都是相同的，那么也无需记录所有卡片的位置信息，因为无论是插入卡片还是删除卡片，都只有那些位于位置坐标在变化卡片坐标的后面的卡片才会受到影响的，前面的是不会变的

```js
// First
activeList.forEach((itemEle, index) => {
  rectInfo = itemEle.getBoundingClientRect()
  transArr[index + stepIndex][0] = rectInfo.left
  transArr[index + stepIndex][1] = rectInfo.top
})
```

**Last**

动画的结束状态，其实就是增加或者删除了卡片之后，其余卡片的状态：

```js
if (updateStatus === 0) {
  // 增加卡片
  newListData = this.state.listData.slice(0, activeIndex).concat({
    index: cardIndex++
  }, this.state.listData.slice(activeIndex))
} else {
  // 删除卡片
  newListData = this.state.listData.filter((value, index) => index !== activeIndex)
}
```
因为这个时候没给卡片加 `transition`属性，所以卡片数量更新这个过程，其实就是一瞬间的事情，人眼是无法察觉到任何变化的，但是页面上的元素确实是发生了变化，然后此时测量卡片的位置信息，即 `Last`所需要的数据

**Invert**

获取了动画起始阶段受影响的卡片的位置信息后，就可以通过 `transform`属性对元素的位置进行反转了

```js
// Last + Invert
const stepIndex = updateStatus === 0 ? 1 : 0
activeList.forEach((itemEle, index) => {
  rectInfo = itemEle.getBoundingClientRect()
  transArr[index + stepIndex][0] = transArr[index + stepIndex][0] - rectInfo.left
  transArr[index + stepIndex][1] = transArr[index + stepIndex][1] - rectInfo.top
}
```

**Play**

准备阶段就绪，就可以进行最后一步 `Play`起来了，这一步的关键就是给元素加上 `transition`属性，并移除 `transform`给元素带来的位置变化：

```js
// Play
// 重置
transArr = getArrByLen(this.state.listData.length)
setTimeout(() => {
  this.setState({
    animateStatus: 3
  })
}, 0)
```
因为浏览器会对页面的 `DOM`变化进行合并优化，所以为了能在视觉上呈现出想要的动画效果，这里必须要打断这种优化，`setTimeout`是一个很常用的方式

到此为止，就完成了文章开头的那个动画效果，我做了个 [Live Demo](https://accforgit.github.io/flip/index.html)，有兴趣的可以亲自试下，另外代码也可以上传到 [Github](https://github.com/accforgit/blog-data/blob/master/FLIP/src/components/Card/index.js)

## 实现图片放大/恢复动画

微信app里聊天界面点击预览图片时，图片从对话框到全屏预览的这个过程，用了一个过渡的动画，呈现出图片从小图到大图和从大图恢复到小图的全过程，缩放过程类似于下面这种：

![img](flip_3.gif)

这种也属于连续动画，当然也可以通过 `FLIP`来轻松实现

**First**

这里涉及到图片的位置和尺寸的变化，图片从 `First`的小图原位置和小图尺寸，变成了 `Last`状态下的大图位置和大图尺寸，所以需要同时获取这两个数据，其实都是可以通过一次调用 `getBoundingClientRect`完成

**Last**

获取图片已经处于预览状态下的尺寸和位置信息，同样使用 `getBoundingClientRect`完成

另外，为了更好地利用 `transform`动画，我这里将图片两个状态下的尺寸变化转变为 `scale`值的变化，`First`与 `Last`状态下宽度或者高度的比例就是这个 `scale`的应当取值（在没有改变图片宽高比例的前提下）

```js
scaleValue = rectInfo.width / lastRectInfo.width
```

**Invert**

使用 `transform`进行位置和尺寸（即改变 `scale`值）的反转

这里有一点需要注意的是，由于 `transform`动画默认的 `transform-origin`为元素的中心，即`50% 50%`，但是计算出来的 `left`和 `top`却是相对于没有缩放的图片而言的，所以当 `scale`取值不唯一时，图片动画的 `First`状态就会发生偏差，需要将 `transform`设为 `0 0`以消除这种偏差

![img](preview_3.png)

**Play**

为图片添加 `transition`属性，并移除相关 `transform`属性，即可启动动画

可以看到，套用了 `FLIP`之后，原本看起来比较棘手的一个动画，被轻易模式化实现了

至于放大后的图片恢复到小图这一个阶段，可以看成是另外一个 `FLIP`动画，继续套用即可，只不过这个动画就是上一个放大动画的逆向，所需的尺寸和位置信息都已经拿到了，可以省去调用 `getBoundingClientRect`的过程

同样做了个 [Live Demo](https://accforgit.github.io/flip/index.html?showIndex=1)，有兴趣的可以亲自试下，另外代码也已经上传到 [Github](https://github.com/accforgit/blog-data/blob/master/FLIP/src/components/Preview/index.js)

## 为什么要用FLIP

有些人可能比较疑惑，如果想要实现动画的话，直接 `transform`不就好了，为什么要多此一举搞个 `FLIP`的概念出来？

我一开始也有这个疑惑，但是当我实际实现一个动画的时候，比如开头的那个卡片动画，这个疑问就立即得到了解答。

对于一些动画，你明确的知道它的初始态(`First`)和结束态（`Last`），比如你就想让一个元素从 `left:10px;`移动到 `left:100px;`，那么你直接 `transform`就好了，根本没必要 `FLIP`，用了反而多此一举；

但除此之外，还有一部分你无法明确的初始态(`First`)或结束态（`Last`）的动画，比如开头那个卡片动画，除非你限定死了每个卡片的尺寸以及整体页面的尺寸，否则你无法明确当你任意插入或者删除了某个卡片之后，其他卡片应当在什么位置。

比如，在你的浏览器下，每个卡片宽高都是 100，浏览器页面宽度为 1380,所以每一列可以排布 13个卡片，但这只是在你的浏览器上，用户的浏览器页面宽度可能是 1280，也可能是1980，每一列排布的卡片数量可能是12也可能是19，不一而足，甚至你还可以任意 `resize`页面的尺寸，那么这个时候，你怎么确定每个时刻所有卡片 `last`状态的信息？

你可能会说，我当然不知道，但是我可以使用浏览器 `API`进行测量啊。

不好意思，这正是 `FLIP`要做的事情之一，你还是在无意识地情况下用到了这个东西，只不过相对于被前人总结并优化后的 `FLIP`来说，你的整体用法可能更零散更不规范一些。

就像标题说的那样，`让动画变得更简单`，你可以不用，但是如果你知道怎么用了，那么动画对于你来说就是一个公式一把梭，更 `easy`。

## 小结

很多前端同学似乎不太在意动画，认为这只是一个辅助能力，业务逻辑才是最重要的，其他的全都靠后站，就算是有时间也要看心情再决定搞不搞

我的看法是，业务逻辑当然是要放在首位的，但是同样也不要小看了其余的细枝末节，例如动画，一个体验良好的动效完全可以吸引用户的更多停留，以一种通用的方式从侧面提升业务的转化效果，某些特定场景下，其所能起到的作用甚至可以与业务的目标并驾齐驱
