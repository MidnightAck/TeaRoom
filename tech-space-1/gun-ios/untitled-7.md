# 页面首次加载FPS优化

### 背景

冷启动APP，用户第一次进入某一个页面的时候，往往FPS会降低一大截。这通常是因为刚进去页面的时候，在viewDidLoad里做的事情很多，导致CPU会更多的使用在对象的初始化上。因此帧数会突降。 虽然在整个APP的启动周期中占比比较少，但是优化这一点可以有效的提高用户的使用体验。

### 问题

FPS优化的问题显然是需要case by case的看。但是往往在一些检验和解决的方案上有一致性。本文记录一下在解决问题的过程中使用到的一些方法。

#### 发现问题

发现问题可以通过Time Profile来看，首次进入页面会发现一个比较明显的毛刺（如果没有的话那其实就不用太优化了）。查看毛刺内部的方法都是调用的哪些。 事实上，viewDidLoad内部的方法基本可以想到。大多是对象的初始化，我们能做的基本上就是以下几种：

* 延迟初始化时间，比如使用lazy修饰
* 放到异步初始化。

### 解决方案

在本文的场景下，我们碰到的问题主要是以下四类：

* lottie初始化和动画
* 第三方库的耗时
* 部分对象的初始化耗时
* 页面加载的主逻辑放到异步

#### Lottie

项目使用的lottie是swift版本。而swift版本的lottie有着明显的性能问题。根据开发者的说法是swift版本的ARC机制带来的问题，修复这个问题需要大量的重构。[Performance regression on Lottie 3 · Issue \#895 · airbnb/lottie-ios · GitHub](https://github.com/airbnb/lottie-ios/issues/895)

> Yeah I still haven’t had a chance to work on the performance issues. They are caused by some limitations in the swift language cause by automatic memory management in structs. To fix the issue requires a pretty hefty refactor that I haven’t had the resources to tackle. \(Im working on lottie in my personal free time\)  
> We had to bump to 3.1 because of an API change. I will keep this thread updated when I get a chance to work on the performance issues.

然而，OC版本的lottie并无此问题。因此实际上回退到OC版本就可以解决这个问题。当然了，最好的方法还是尽可能的避免lottie的使用，但是当我们的业务场景变得复杂的时候，可能就显得不是很现实。尽管如此，可以尽量使用原生实现的时候就原生实现。

#### 第三方库的耗时

在前几项大耗时的问题中，一个主要的问题就是调用了第三方库的某个方法。该方法默认使用nib的方法初始化，但是项目并非使用nib进行初始化。因此在第三方库中提供了可配的方法，选择不使用nib。因此少了一次bundle的IO时间。

#### 部分对象初始化耗时

可以lazy的设置为lazy，可以放到异步初始化的可以放到异步。但这个还是要谨慎，毕竟可能大多是的对象都是必须要在这个时机初始化的。

#### 主要加载逻辑异步

页面复杂的话，每一个页面的model和viewmodel都会比较多。因此我们可以将这个页面的model加载放到异步。

这里需要关注的问题是，哪些可以放到异步，哪些必须在主线程加载。放到主线程的至少是：

* UI相关，所有的UIView以及继承自此的对象
* viewController，在异步中使用VC不是一个很安全的做法。因为他涉及的过多，我们无从得知哪个地方会改动到，因此做不到线程安全。

因此我们必须做好拆分，生成model的过程可以放在异步，做完后回到主线程，后续对于layout的更新在这个时机更新。

要注意这里异步的方法是另起一个线程，异步的进行加载。结束之后回到主线程。在主线程异步的完成后续的回调。这里一定注意不要在主线程同步，因为有可能会阻塞到后续的操作。

