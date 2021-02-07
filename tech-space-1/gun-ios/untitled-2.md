# 扩大点击区域

### 背景

有时UE给出的设计稿看起来很美观，对于用户来说并不是那么方便的进行操作。按钮可能小了容易误触，或者不容易触发。开发就需要灵活的去增大某个控件的点击区域，在这里总结一下可以使用的方法。

### touchView

一个很容易想到的方法是去增加一个touchview，增加一个透明的view盖在控件的上方或者下方。响应链会找到对应的事件进行触发。

这样做十分的简单，但是唯一不好的一点是，本质上是增加了一层view，虽然是透明的层，但是我们尽可能的希望在渲染的时候减少渲染的图层的数量。更好的一种做法是直接的，增大可以触发事件的点击区域。

### Hittest

#### 原理

```objectivec
- (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;   // recursively calls -pointInside:withEvent:. point is in the receiver's coordinate system
```

先来看这个方法，hittest会一直回溯响应链上的view，直到找到包含目标point的view或者返回nil。这个目标的point就是用户点击的point。

hittest会回溯整个图层树，对每一个view调用 [func point\(inside: CGPoint, with: UIEvent?\) -&gt; Bool](https://developer.apple.com/documentation/uikit/uiview/1622533-point) 这个方法，来决定哪一个view来接受event并作出响应。

hittest会忽略掉被hidden的view，disable用户交互的view、alpha低于0.01的view。

#### 使用

在实际的业务逻辑里面，这个hittest实现的逻辑要根据具体的业务逻辑进行重写。

来看一个典型的使用流程。

```swift
    public override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        let followButtonRect = followButton.convert(followButton.bounds, to: self)
        let extendedRect = followButtonRect.insetBy(dx: -20, dy: -20)
        if followButton.isHidden == false && extendedRect.contains(point) {
            return followButton
        }
        return super.hitTest(point, with: event)
    }
```

入参是两个：用户点击的位置Point和触发的手势event。返回值是UIView。首先去获取当前控件的大小，通过`convert`方法转换成`CGRect`类型。接下来可以通过insetBy方法扩大点击区域，负数代表着向外延伸。如果扩展后的区域中包含这个Point，就将整个view认定为可以触发事件的view。否则就进行回溯。

#### 注意事项

hittest虽然好用，但是要注意，虽然hittest可以忽略掉被hidden的view，但是在我们上述的写法下，实际上还是根据point的位置进行判断。因此在做diff的时候，注意加上是否为hidden的判断条件。

### 更好的方法？

实际考虑一下，事实上我们在很多场景下都需要进行点击区域的调整。这也是抽象出一个黑箱的方法。理想情况下，或许我们可以直接在继承的底层的View中去扩展一个这样的方法，使用一个extension来实现。这样继承自这个类的子类都拥有这个方法。直接传入原始的view和想要扩大的程度（比如x、y方向的CGFloat值），就可以实现扩大点击区域的方法。

