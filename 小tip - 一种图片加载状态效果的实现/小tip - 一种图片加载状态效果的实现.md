
做的一个需求，其中有一个是实现类似于下图的一个图片上传效果：

![img](gf.gif)

从本地上传图片到服务器，然后服务器响应返回这个图片在服务器上的链接地址，将这个链接地址所对应的图片显示到屏幕上，并且在此图片资源完全下载下来之前，呈现一个动态 `loading`的展位图，直到图片完全下载后进行替换

## 小tip

这是个很常见的需求，关键点在于检测图片的加载完成事件，一般的做法是，先将 `img`标签的 `src`属性指向一个当图片加载完成后，再将 `src`指向真正的链接即可
```js
const img = new Image()
// 真实图片地址
const realSrc = 'http://a.com/real.png'
img.src = realSrc
img.onload = () => {
  // domImgElement 为 img 的 HTML DOM对象
  domImgElement.src = realSrc
}
```
这种做法是没什么问题的，大多数情况下也都是这么做的，不过由于我擅(xian)长(de)思(dan)考(teng)，然后就想到一个小问题

我手头的这个需求是基于 `vue`，在页面渲染完毕之后，再想改变页面上的内容，就需要重新渲染一遍页面，尽管 `vue`存在 `vnode`的概念，更新速度很快，但就算再快都还是要把整个页面刷一遍的，这个过程无论如何是跑不掉的，如果页面上存在十几张甚至几十张类似于上面那种图片异步加载效果，那岂不是要改变几十次 `data`值，`vue`内部就要对应 `patch`几十次，然后页面也就跟着整个更新几十次，想想都觉得可怕

<del>每多 `patch`一次 `vnode tree`，每多更新一次页面，都将引起 `CPU` 和 `GPU` 的能耗升高，日积月累四舍五入那就是一个庞大而隐形的能源消耗黑洞，对个人的发展、对公司的财务、对地球的温室效应都将是一个沉重的负担！</del>

更何况，一般这种列表类型的图片数据，都是存在于一个数组中，每当有图片下载完毕，在替换 `src`之前，还要先根据某个标志位，例如图片 `id`或者 `url`来从这个图片数组中找到对应的图片，然后才能进行替换，而且这一步还要注意`vue`不自动对嵌套对象的属性进行响应式操作的问题：
```js
loadImg (imgId) {
  const img = new Image()
  // 真实图片地址
  const realSrc = 'http://a.com/real.png'
  img.src = realSrc
  img.onload = () => {
    this.imgList.some(imgObj => {
      if (imgObj.id === imgId) {
        imgObj.src = realSrc
        this.$forceUpdate()
        return true
      }
      return false
    })
  }
}
```

这只是一个小需求而已，为什么要搞得这么复杂？于是我决定最好简单一点，不要那么依赖 `js`，最好将大部分的工作交给 `css`来完成，因为相比之下，`css`的消耗更低

关键就是不进行替换 `img`标签 `src`的操作，这样一下子就能节省大半的 `js`操作，以下述 `DOM`结构为例：
```html
<div class="img-item">
  <img :src="item.realUrl" alt="" />
</div>
```
`item.realUrl`就是当前图片的真实地址，拿到就直接赋值上去，后续也不会再操作 `src`这个属性了，因为大部分工作移到了 `css`上：

```css
.img-item {
  position: relative;
  width: 80px;
  height: 80px;
  background-color: #aeaeae;
}
.img-item::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 22px;
  height: 22px;
  margin-left: -11px;
  margin-top: -11px;
  /* 这是加载效果的图片 */
  background: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAABAUlEQVQ4T6WSzyrEYRSGn2crF6GmJJTsLLGyEVM2lAshdriUsbAwsWeWlpJ/Kck1KNtXM330G8b84due8z7fed9z5J/P3/RJNoCDUt9XT3r1/gAkmVSfktwB00V0r84MBCS5AJaAc6A2EiDJOPBW+WUb2KtaSDKr3lYn6bKQ5AxYBS7V5WHy/QIkWSmCV/VhGHG7pwNIsg6cFlFdbfbZzlRHqI9VwCbQKKIt9bgPYKEArqqAMWCriBrq+0gWPpuTzAG7wKF68x2SpKY+99tC2/sa0FTr1cYkE8ALMK9ef9a+r7F9RDvAkdpK0ip+F0vY/SfoMXIXYOApDxvcrxn8BfABIiRjEYfmQAcAAAAASUVORK5CYII=') no-repeat;
  background-position: center center;
  background-size: contain;
  animation: loadpic .5s infinite linear;
  z-index: 1;
}

@keyframes loadpic {
  from { transform: rotate(0) }
  to { transform: rotate(360deg) }
}

img {
  position: relative;
  height: 100%;
  width: 100%;
  z-index: 2;
}
```

`.img-item`是包裹单个 `img`的父元素，它的作用有两个：
第一，撑开占据的空间
第二，用它的 `::before`伪元素来加载用于标识图片正在加载的动效 `loading`图，也就是说，将 `loading`图放到了 `::before`上

`::before`会一直存在，直到 `.box`内的 `img`资源下载完毕后显示出来，将其遮盖住，因为 `img`元素的宽高与 `.box`相同，并且 `img`的 `z-index`更大，所以只要 `img`没有加载完毕，那么就不会显示，那么就会显示 `::before`元素，也就是显示 `loading`状态，而只要 `img`加载完毕了，那么就会显示出来，就会遮盖掉 `::before`，这个过程视觉上看起来就能达到文章开头那个图片上的效果，而且消耗更低

`emmmm`，既然说到了图片，那么再顺便说个关于 `img`标签不太常用但有时候会比较有用的功能点吧

对于一张图片：
```html
<img src="http://a.com/b.png" />
```
如果 `src`的指向没有对应的资源或者不合法，那么在浏览器上将会呈现一种特殊的样式，例如：

![img](12.png)

不同的浏览器对于无效图片的显示可能也不一样，在我的 `Chrome`上，就是上述这种破碎图片的样子，就是告诉你图片找不到了，这种用户体验肯定是不太好的，对于这种情况，大部分的做法就是监听 `img`标签的 `onerror`事件，如果触发了此事件，则说明失败，然后就可以走失败的逻辑了

不过这也是 `js`的方法，每个图片都要设置一个监听事件，那么如果页面上有几十个图片，岂不是要设置几十个监听事件？

<del>每多设置一个监听事件，都将引起...</del>

这只是一个渐进增强的体验，不应该投入那么多的资源，于是需要再次借助 `css`的能力，不监听 `error`事件了，直接给需要加载失败就友好显示的图片加上如下样式：
```css
img:before {
  /* 关键是这句 */
  content: '图片加载失败~';

  /* 下面都是辅助的样式代码 */
  display: block;
  padding: 0 20px;
  background-color: #eaeaea;
  color: indianred;
  line-height: 30px;
}
```
如果图片加载失败，那么原先应该显示破碎样式的地方，将会显示 `图片加载失败~`的提示文案

![img](13.png)

主要就是利用了 `:before`和 `:after`这两个伪元素在 `img`标签上的特性，`img`正常加载，则这两个伪元素的内容就不显示，如果 `img`加载失败，就显示这两个伪元素的内容，这两个伪元素的能力还是很多的，上面只是我随便举的一个例子

## 总结

我觉得 `css`对于前端工程师来说也是一项很重要的能力，别的不说，减少了 `js`的代码量，自然也就能减少报错的概率，有时候还可以很轻松地实现一些需要一大堆 `js`才能实现的效果，提升页面的性能

所以我有时候看到有的前端说 `ta`根本不 `care` `CSS`，认为这东西连编程语言都算不上，语法规范更是乱七八糟，学都懒得学，`ta`只需要 `JS`就能写出任何效果来，特别是在如今各种 `js`框架大行其道的当下，很多初学者更是只知 `js`而不知 `css`，甚至连 `html`都用不好，不仅是初学者，就算是一些工作了好些年的前端工程师，看到他们写的代码的时候，我有时候也会惊叹于他们的 `css`和 `html`代码居然还可以这么写

大多数情况下这种现象或许也没什么毛病，毕竟屁股决定脑袋，就算是出去面试，面试官也几乎只会问 `js`方面的问题，而不会问你 `section` 和 `article`标签适合在哪些情况下使用，字体的基线是怎么回事，`line-height`又有什么作用，我只是很纳闷，作为一名前端开发工程师，前端三剑客只会其一，悲哀或许谈不上，但三种技能开局就丢掉了俩，最起码可以算是自毁功力了吧

>这篇文章似乎有点水...突然想写就写了...