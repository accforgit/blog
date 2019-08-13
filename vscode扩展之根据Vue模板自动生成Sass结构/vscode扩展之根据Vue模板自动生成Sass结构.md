某次小组内开周会，提到开发效率的问题，有个小伙伴提到写新页面的时候，`template`大概布局写完后，对着 `template`结构写 `scss`是件比较耗时耗力的事情，如果能作出一个自动依据 `template`结构生成 `scss`文件的 `vscode`插件就好了

我当时也没在意，后来周会结束后觉得这事情可以做一下，于是抽空看了下 `vscode`扩展的[开发文档](https://code.visualstudio.com/api)，就上手 `code`了，做出来后效果还不错，最起码不用再做人工对着 `template`写 `scss`这种没技术含量的事情了，写好了一大堆 `template`之后，一键转换，还是挺爽的

插件已经发布到 `vscode`扩展市场，可以在[`vscode`插件市场](https://marketplace.visualstudio.com/)查找安装，或者 `vscode`上直接搜索 **AutoScssStruct4Vue** 安装

插件的源码已经上传到 [Github](https://github.com/accforgit/AutoScssStruct4Vue) ，需要的可自取，如果有问题，直接提 [issue](https://github.com/accforgit/AutoScssStruct4Vue/issues) 即可

## 模板解析

`scss`文件的关键就是选择器，能在模板上体现出来的选择器属性有 `class`、`id`和标签名，所以必须要从模板中取得所有的选择器

模板的处理实际上就是字符串的处理，通过正则表达式从 `<template>...</template>`中提取所需的选择器名，既然是 `vue`，我第一个就想到直接从 `vue`源码中将 `vue`处理 `template`的部分抄过来，不过后来看了一下源码，想法可行，但是性价比不高

`vue`将 `template`解析成 `ast`的部分，与其他的一些处理逻辑耦合在一起了，而且处理了很多我并不需要的东西，比如指令、组件这些，我只是想取个选择器属性而已，所以这个想法就放弃了

再仔细一想，抛开 `vue`的那些东西来看（比如指令处理这些），其实处理 `template`跟处理普通的 `html`片段没太大区别，于是问题就变成了将`html`片段转为 `ast`，这个就容易很多了

不过还是有些差别，毕竟 `vue`的 `template`和普通 `html`片段之间的处理方式还是有点不同的，差异点在于 `vue`的一些特性，例如 `v-bind`语法，以及组件标签

### v-bind

以下以 `class`这个选择器属性为例，`id`同此

因为考虑到 `v-bind`语法，所以在 `parse`模板时，正则表达式需要将这个考虑进去，属性匹配正则为：

```js
const attrRE = /((:|v-bind:)?[\w$\-]+)|(?<word>["'"]){1}[\s\S]*?\k<word>/g;
```

这个正则会将标签的属性全都匹配，因为`vue` 的`class`属性有三种写法：

- 字符串
- 数组
- 对象

所以转换为 `ast`后，可能的结构有：

```js
{
  attrs: {
    class: 'box1 box2'
  },
  bindAttrs: {
    class: "['box1', name]"
  },
  // bindAttrs: {
  //   class: "{isActive: 'name'}"
  // }
}
```
考虑到直观性，在解析模板的时候，如果遇到是 `v-bind`的选择器属性，我会将此属性记录在 `bindAttrs`字段上，如果是普通的字符串尚需经，则记录在 `attrs`属性上，主要是为了方便后续提取属性值

对于 `attrs`上的选择器属性，这个没什么好说，直接字符串匹配即可：

```js
// 获取标签的 选择器名
seletorStr.split(/\s+/).filter(item => item)
```

至于 `v-bind`属性的处理，其实也很好办，对于 `data`字段，例如下面代码段中的 `name`，这个 `name`变量到底是什么，只有在运行的时候动态获取，字符串静态处理的情况不可能知道 `name`是什么东西的，所以直接跳过

```js
bindAttrs: {
  class: "['box1', name]"
}
```
那这就好办了，对于 `"['box1', name]"`或者 `"{isActive: 'name'}"`字符串来说，我只需要取引号中间的字符串即可，即 `box1` 和 `name`

最后，如果一个标签既没有 `class`也没有 `id`，那么就取这个标签的标签名作为选择器

### 组件标签

相比于普通 `html`片段，`vue`是有组件的概念的，除了内置组件还有自定义组件，这些组件可以加选择器属性也可以不加，可以自闭合也可以不自闭合，所以要根据这两种情况分别处理下

这里暂且认为某个标签只要不是标准的 `html`标签，那么就是组件，组件可以加 `class`、`id`这种选择器属性，也可以不加，如果加了选择器属性，则提取，否则就将组件的子元素当成组件不为组件的父元素的子元素进行处理

可能有点绕，看个例子就明白了：
```html
<div class="box1">
  <List>
    <p id="box2"></p>
  </List>
  <List class="box3">
    <p id="box4"></p>
  </List>
</div>
```
对于 `#box2`来说，其父元素是个组件，并且此组件没有选择器属性，又不可能将组件的标签名`List`当成选择器，所以跳过 `<List>`，将 `.box1`当成是 `#box2`元素的父元素来进行处理

而对于 `#box4`来说，虽然它的父元素也是个组件，但这个组件有选择器属性 `class`，所以不跳过其父元素，最后生成的 `scss`结构如下：
```scss
.box1 {
  #box2 {}
  .box3 {
    #box4 {}
  }
}
```

## Scss文件解析

将 `template`转成 `ast`，获取到 `template`结构及选择器，接着按照层级转为 `scss`字符串即可

然而这是一次性操作，如果你后续还对 `template`进行修改，如果再按照前面来这么一下，生成新的结构字符串，岂不是把你自己写的 `css`规则给清除了？

比如：

```html
<List class="box3">
  <p id="box4"></p>
</List>
```
对于上述 `template`来说，生成的 `scss`如下：
```scss
.box3 {
  #box4 {}
}
```
这是个结构，那你肯定要添加规则的，比如：
```scss
.box3 {
  width: 100px;
  height: 200px;
  #box4 {
    color: #fff;
  }
}
```
然后你改了下 `template`，变成：
```html
<List class="box3">
  <p id="box4"></p>
  <span></span>
</List>
```
再次生成：
```scss
.box3 {
  #box4 {}
  span {}
}
```
所以你之前写的 `css`规则没了

`template`的修改是很常见的，你不可能一上来就把一个组件的 `template`写得明明白白，所以这是个常见的场景，需要避免问题的产生

先根据新 `template`生成新 `Scss Ast`，然后对比新旧 `Scss Ast`，根据二者之间的差异修正新 `ast`，填补对应结构上的 `css`规则，这是一条可行之路，但考虑到`scss`文件就是由 `template Ast`映射而来的，所以选择器的结构肯定能对的上，直接对比 `template ast`和 `Scss Ast`之间的差异，在旧 `Scss Ast`上进行修正，保留原有 `css`规则也是可行的

生成新的 `template`之后，将之与旧 `scss`结构进行对比，因为考虑到 `css`规则写法的放飞性，为了避免误删规则，我这里在对比结构时，只会在旧版本的 `scss`上进行增加操作，而不会删减内容，删除操作由你自己来做，毕竟相比于写，删是一件很容易的事情

 不可能直接对比 `scss`字符串的，还需要将 `scss`字符串转为 `ast`才好

这比 `template`的转译还简单，无非是字符串遍历递归罢了，只不过由于 `css`规则的写法太随心所欲，所以需要注意的小点很多，比如因为同一级别下，一个选择器规则可能会被同时应用在多个标签上，但是这些标签只需要一个 `css`规则就够了，即存在标签与 `css`规则之间的多对一关系，需要进行过滤

## 新旧 AST 结构对比

得到新的 `template ast`之后，与旧 `scss`结构进行对比，对比同级别下的节点的差异，前面说了，对旧版 `scss`文件实行只增不减操作，所以如果发现新 `ast`相比于同级别旧 `ast`新增了节点，则在 `Scss Ast`的同级别上进行新增 `Scss Ast`节点的操作，如果发现 `template`删除了某个节点，则跳过不做处理

```js
if (matchIndex === -1) {
  // 没找到，说明 template 中新增了标签
  scssObj.children = scssObj.children.slice(0, childIndex + i).concat(
    {
      rule: '',
      selectorNames: selector,
      children: i == 0 ? trackChildren(templateObj) : []
    },
    scssObj.children.slice(childIndex)
  )
}
```
我这里严格映射了子节点之间的先后顺序，如果一个选择器属性对应的 `template`节点是其父元素的第 `n`个子元素，则在 `Scss Ast`中，此选择器属性也会是其父元素的第 `n`个子元素（不考虑重复选择器的情况下），这种顺序将会被保留

对比修正完毕之后，将会得到新 `Scss Ast`结构，并且结构上保留了已有的 `css`规则

## Scss Ast 转为 scss 字符串

主要是 `ast`结构的遍历操作，额外需要处理下换行缩进等问题，比较简单

## 应用为 vscode 插件

上述核心逻辑完成了，转为 `vscode`插件其实就是做交互了，对着 [vscode插件开发文档](https://code.visualstudio.com/api) 找 `API`即可

对于这个插件，我设置了两个设置项

其一是插件运行时机，其二是 `scss`文件保存的位置

### 插件运行时机

对于这个设置项，提供了两个选项：当文件保存时，和当右键点选菜单栏时

当文件保存时，顾名思义，就是当你写好了 `template`之后，保存当前文件，插件自动解析当前 `vue`文件中的 `template`，得到 `scss`字符串

当右键点选菜单栏时，即你的鼠标在编辑器文档上右键，出现一个菜单栏，点击插件命令即可，此为默认选项

### `scss`文件保存的位置

你可能会在一个 `.vue`文件上同时写 `template`、`script` 以及 `style`，也可能将 `scss`写到单独的文件中，这也是很常见的场景，所以提供了两种选择，要么默认将 `scss`字符串写到当前 `.vue`文件的 `style`标签内（如果不存在，则自动创建），要么你指定 `.scss`文件的保存路径（如果不存在，则自动创建）

## 总结

插件的开发逻辑其实很清晰，主要是需要考虑的场景很多，正则规则需要兼容各种随心所欲的写法，虽然我这里只考虑常见场景（不常见场景，例如类名用中文，这种写法虽然符合规范，但我这里不考虑），但常见场景的写法依旧很多，稍有哪种情况没考虑到就出 `bug`
如果有问题，直接提 [issue]() 即可