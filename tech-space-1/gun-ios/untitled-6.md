# 页面生命周期带来的布局问题

### 问题

在页面上需要有条件的确认哪些view展示，哪些view不展示。针对这些再进行不同的布局。在一开始做的时候，我在viewwillappear中判断了对不同的view是否hidden进行了设置。在viewdidload中，根据这些view的可见状态来进行布局。

出现的问题就是，布局出现错误。

### 解决

问题在于，页面加载的时序是

```text
initWithCoder:(NSCoder *)aDecoder：（如果使用storyboard或者xib）
loadView：加载view
viewDidLoad：view加载完毕
viewWillAppear：控制器的view将要显示
viewWillLayoutSubviews：控制器的view将要布局子控件
viewDidLayoutSubviews：控制器的view布局子控件完成
这期间系统可能会多次调用viewWillLayoutSubviews 、 viewDidLayoutSubviews 俩个方法 
viewDidAppear:控制器的view完全显示
viewWillDisappear：控制器的view即将消失的时候
这期间系统也会调用viewWillLayoutSubviews 、viewDidLayoutSubviews 两个方法 
viewDidDisappear：控制器的view完全消失的时候
```

因此，在布局的时候并不知道哪些view是可见的，因此就会出现错误。

### Learning

* 先didload，后willappear
* 每次hidden结束后，这个view要重新布局
* didload内的布局顺序无所谓

