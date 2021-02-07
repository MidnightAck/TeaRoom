# Actionsheet没有show

### 需求

actionsheet作为一个页面，在其中显示loading状态和noNet状态，同时其中要装载数据。

### 问题

因为项目在actionsheet上有着较好的组件化方案，一开始采用的做法是，先生成一个actionsheet，该actionsheet内的contentview是一个空view，同时装载了无网络样式的view。如果拉取到了数据，就在生成一个actionsheet。我们期望的是，后一个actionsheet将前一个顶掉，从而显示新的值。

但是并没有。

第一次的显示总是ok的，但第二次生成actionsheet之后就不会再更新。

### 原因

首先从设计上讲是非常不好的设计。明明逻辑上是一个view，却不断反复的去生成新的view，这是非常不好的逻辑，正确的做法应当是去使用一个view，在其中不断的更换数据，这才是比较好的做法。

做法是将actionsheet的contentview作为一个大的view，里面包含有数据的view和无网络时的view。在不同是时机隐藏或者开启对应的view即可。

### Learning

* 避免多次的初始化，生成一个，复用一生
* 在做的时候充分考虑设计

