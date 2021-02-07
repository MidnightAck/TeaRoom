# 响应链

### 背景

最近接连几个需求都碰到了在响应链上的问题。有的是因为点击事件透传到superview，有的是因为子View将事件过早的拦截。针对一些具体的情况，我们对响应链相关的知识总结一下。

### 响应链

关于响应链具体是什么，有很多文章都写到了，这里不再赘述。附上Google搜索出来排名很高的两篇文章，基本上把知识点都囊括了。[iOS响应链\(Responder Chain\) - 简书](https://www.jianshu.com/p/4155c9ffe1a8) [iOS 中事件的响应链和传递链 \| Gsl’s Library](https://gsl201600.github.io/2019/12/25/iOS%E4%B8%AD%E4%BA%8B%E4%BB%B6%E7%9A%84%E5%93%8D%E5%BA%94%E9%93%BE%E5%92%8C%E4%BC%A0%E9%80%92%E9%93%BE/)。

简单描述一下，基本上是以下的过程： 1. 点击屏幕产生触摸事件，系统将这个事件加入到一个由UIApplication管理的事件队列中，UIApplication会从队列里取事件分发下去，首先传给UIWindow 2. 在UIWindow中就会调用`hitTest:withEvent:`方法去返回一个最终响应的视图 3. 在`hitTest:withEvent:`方法中就会去调用`pointInside: withEvent:`去判断当前点击的point是否在UIWindow范围内，如果是的话，就会去遍历它的子视图来查找最终响应的子视图 4. 遍历的方式是使用倒序的方式来遍历子视图，也就是说最后添加的子视图会最先遍历，在每一个视图中都回去调用它的`hitTest:withEvent:`方法，可以理解为是一个递归调用 5. 最终会返回一个响应视图，如果返回视图有值，那么这个视图就作为最终响应视图，结束整个事件传递；如果没有值，那么就会将UIWindow作为响应者

这里有几点要注意的。首先是所有的交互事件是放在一个队列中管理的。为什么是队列？先发生的交互应当先被处理，这符合预期。这其实也能理解为什么有时候会觉得卡顿。当响应这些交互事件的时间要高于我们的反应时间时，就会觉得卡顿。当这个时间要高于我们两次交互的时间时，就会有卡死的感觉。其次，响应链对于view的遍历顺序，应当是一个树的深度遍历方式。这也就解释了为什么是最高层的UIApplication发起的遍历，但是会是最后添加的子view率先响应的问题。同样这也是符合预期的，当我们对页面发起交互事件的时候，点击的对象自然是“离我们最近的”子view。

原理看起来比较简单，我们碰到的主要问题是，响应链什么时候传了下去，什么时候又被拦截掉了。接下来看两个具体的例子。

### 问题

#### scrollView无法滑动

我们设置了三层view。底层的父view是一个scrollView，在其上有一个headerView，在headerView上有一个自定义的View作为button。这三个View都是UIView。在headerView中我们可能展示一些图片之类的资源。想要的效果是：在headerView中依然可以滑动，同时headerView中的按钮是可以点击的。

在整个需求做完之后首先碰到的问题是，headerView并不能滑动。这是因为UIView并不处理滑动手势。并且滑动的手势并不会回传给superView。这是因为响应链和手势识别并不是一套系统。

> A window delays the delivery of touch objects to the view so that the gesture recognizer can analyze the touch first. During the delay, if the gesture recognizer recognizes a touch gesture, then the window never delivers the touch object to the view, and also cancels any touch objects it previously sent to the view that were part of that recognized sequence.  
> 开发者文档里面说明了，当一个交互事件发生的时候，手势识别器首先获得识别触摸的机会。在一定的时间内（150ms），如果发生了手势操作，那么手势识别器将进行接下来的识别操作。如果没有，这个事件将传递给响应链，进行tap等事件的判断。因此要是直接交互的view里面没有响应手势识别的控制器，那就没了。

一种修改方法为，将headerView的`isUserInteractionEnabled`设置为false。直接取消掉交互能力，这个手势就会被传递给父view，也就是scrollView。但是这样带来的问题就是，headerView上面的button就会被禁用掉。

可用的解就是覆盖`point(inside point: CGPoint, with event: UIEvent?) -> Bool`方法。

```swift
    override public func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
        for v in subviews {
            if v == buttonView, v.frame.contains(point) {
                return super.point(inside: point, with: event)
            }
        }
        return false
    }
```

当触摸事件发生在button的bound内部的时候，返回super的event。否则返回false。注意这里的顺序，因为是深度优先遍历，因此这里的super并不是view的superView。而是在调用栈上的super关系，也就是该view的子view。返回false的原因在于，如果点击到了button之外的headerView，那么就视为没有点击到headerView，直接返回给上一层。手势就可以被scrollView拿到。

#### 两个响应事件

第二个问题是，一个父级的View上有一个子View。这两个都是UIView。两个view都有自己的点击事件。点击事件本质是关联对象，将tap的交互动作和一个action的函数关联在一起。在点击事件发生的时候，取到对应的方法进行触发。

在这个情况下，对于子View的点击事件，被子View响应之后，依然传递给了父View，父View依然作出响应。一个简单的做法是，直接将子View的父类作为UIControl。UIControl不会将触摸事件传递给父View。

