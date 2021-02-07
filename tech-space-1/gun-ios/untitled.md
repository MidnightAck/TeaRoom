# 关于系统相册的若干问题

### 背景

需求是要读取用户的相册，通过一定的策略进行展示，并且允许用户去多选照片进行后续的操作，具体的几项要求是

* 优先读取截图文件夹，没有的话读取recent
* 展示顺序是时间由新到旧
* 用户对相册的改变实时更新到屏幕

在实际开发中遇到了以下这些问题

* 无法获取正确的图片顺序
* 监听系统相册带来的问题
* 大量图片如何加载
* 异步加载图片导致错乱

### 关于相册的常用方法和类

关于图片的方法都集中在`Photos`这个系统库中，几个常用的方法和类如下

#### PHPhotoLibrary

PHPhotoLibrary是获取图片的接口，类似Metal的MTLDevice。

```swift
//获取图片的接口
@available(iOS 8, *)
open class PHPhotoLibrary : NSObject {


    @available(iOS 8, *)
    open class func shared() -> PHPhotoLibrary

        //返回当前的授权状态
    @available(iOS 8, *)
    open class func authorizationStatus() -> PHAuthorizationStatus
        //申请权限
    @available(iOS 8, *)
    open class func requestAuthorization(_ handler: @escaping (PHAuthorizationStatus) -> Void)

        //存储app内对照片的修改
    // handlers are invoked on an arbitrary serial queue
    // Nesting change requests will throw an exception
    @available(iOS 8, *)
    open func performChanges(_ changeBlock: @escaping () -> Void, completionHandler: ((Bool, Error?) -> Void)? = nil)

        //监听系统相册的改变
    @available(iOS 8, *)
    open func register(_ observer: PHPhotoLibraryChangeObserver)

    @available(iOS 8, *)
    open func unregisterChangeObserver(_ observer: PHPhotoLibraryChangeObserver)
}
```

#### PHAssetCollection

PHAssetCollection继承自PHCollection。后者是高级的抽象对象。我们并不使用PHCollection，而是使用其两个继承的子类，一个是PHAssetCollection，表示照片asset资源的集合，可以是一个特殊的相册等。另一类是 PHCollectionList，他是collection的集合。

我们可以通过PHAssetCollection的subtype属性来获取特定的相册类型。

```swift
        //包含三个枚举：album、smartAlbum、moment
    @available(iOS 8, *)
    open var assetCollectionType: PHAssetCollectionType { get }
        //包含多个不同类型的文件夹
    @available(iOS 8, *)
    open var assetCollectionSubtype: PHAssetCollectionSubtype { get }
```

关于album和smartAlbum的区别，网上的回答是：

> The difference is how they are populated with photos.  
> An Album is populated by you - you select the pics that go into the Album.  
> A Smart Album is the result of a search: “find me all the photos with keyword ‘x’” or taken between dates x and y  
> 在`assetCollectionSubtype`中包含了多种多样的照片分类类型，直接去代码中看吧，总能找到你想要使用的。

#### PHAsset

相册中资源的类，基本的属性如下

```text
        //包含unknown、image、video、audio
@available(iOS 8, *)
    open var mediaType: PHAssetMediaType { get }

    @available(iOS 8, *)
    open var mediaSubtypes: PHAssetMediaSubtype { get }

    @available(iOS 8, *)
    open var pixelWidth: Int { get }

    @available(iOS 8, *)
    open var pixelHeight: Int { get }

    @available(iOS 8, *)
    open var creationDate: Date? { get }

    @available(iOS 8, *)
    open var modificationDate: Date? { get }

    @available(iOS 8, *)
    open var location: CLLocation? { get }

    @available(iOS 8, *)
    open var duration: TimeInterval { get }
```

#### 获取图片方法

获取图片的方法可以通过下面两个方法：

```text
        //获取某个特定collection
    // Fetch asset collections of a single type and subtype provided (use PHAssetCollectionSubtypeAny to match all subtypes)
    @available(iOS 8, *)
    open class func fetchAssetCollections(with type: PHAssetCollectionType, subtype: PHAssetCollectionSubtype, options: PHFetchOptions?) -> PHFetchResult<PHAssetCollection>

        //获取某个特定collection的图片
    @available(iOS 8, *)
    open class func fetchAssets(in assetCollection: PHAssetCollection, options: PHFetchOptions?) -> PHFetchResult<PHAsset>
```

### 无法获取正确的图片顺序

#### 问题

在获取到图片的时候，其展示的顺序和系统相册内展示的顺序是不同的，尤其是recent文件夹。

#### 原因

如果我们将图片一次性全部取出来，顺序是对的，就是系统相册的顺序。但是在代码上我们做了一个优化：分为两个回调去取图片，第一次取前50张返回，先交给服务进行加载，这样保证页面上会有5-6屏的图片进行展示；第二次再把所有的图片取回来。

这样就必须保证，第一次取回来的50张是最后的50张图片，也就是在取图片的时候进行一个排序。

```swift
fetchOptions.sortDescriptors = [NSSortDescriptor(key: "modificationDate", ascending: false)]
```

可以看到我们使用修改时间来进行取图，顺序是从后到前（ascending: false）。但是这样会产生问题：当用户对一张图片进行修改，比如裁剪、调色、删除后恢复等，就会更新修改时间。在程序的顺序就会提前，但是在系统相册中顺序还是不变。

接下来考虑将排序的key设置为`creationDate`。一切都很好，但会有一个特殊情况：**当别的设备上发送一张很久之前的图片到本设备，在recent相册中，这张图片会展示在最新的位置；在这张图片隶属的相簿中，会根据他原本的creationDate进行排序；在程序中也会根据原本的creationDate进行排序。**

#### 解决

猜测原因是，recent文件夹内的排序可能有着不同的策略。不是仅根据creationDate来进行排序的。

但无论如何，我们无法通过聪明的办法去获得一部分后面顺序的图片。最后的做法就是一次性获取全部的图片，在reload的时进行逆序。

这样也并不是非常聪明的办法，当用户手机性能比较差而图片资源数量比较多的时候，可能就会卡。我们并没有复现出这样的情况，但理论上存在可能，可以考虑再优化。

### 监听系统相册带来的问题

#### 问题

在没有一行代码申请权限的情况下，程序自己申请了相册权限。但需求要求用户点击button后再去申请。

#### 原因

需求要求当相册更新时，即使程序在前台，也要实时更新内容。这一点可以通过监听library的change完成，获得通知后再去reload一次。但它会默认的去检查权限，如果没有就会申请。

#### 解决

将注册监听的步骤放在手动获取权限后。

### 大量图片加载优化

#### 问题

如题，怎么加载性能最优

#### 方案

因为【无法获取正确的图片顺序】这个问题，我们必须一次获取全部的图片。这个过程可能涉及到大量的图片，最终呈现上就会卡。

卡的原因有两方面：首先是进行了大量IO操作，但实际上iOS的IO操作并不慢；第二点是因为，在生成对应的model时，需要对图片资源进行解析，解析成对应的UIImage，这个过程是异步的，并且会比较耗时，事实上，也正是因为这一点造成了阻塞，他还会引起另一个问题，我们会放到后面去讲。

但在这里，采取的一个措施是，既然IO造成的压力并不是很大（事实上在这个地方也没有其他更聪明的办法）。我们依旧是一次拿到全部的照片，但依旧分两个回调返回，第一个回调返回前50个（此时已经经过排序），第二个回调返回后面的图片。

另外值得一提的是，在有些非常大的数据的情况下，加载图片也是慢的。本需求最终也是设计提出做一个l类似于loading的兜底状态。有些时候，技术上的优化成本较高时，要学会用设计或者产品的方法去解决问题。

### 异步加载图片导致错乱

#### 问题

在reload图片的时候，最明显的是低端机第一次进入图片页面的时候，有些图片会显示错乱，比如第三张图显示在第一张图的位置。在左右滑动，重新的viewWillAppear之后就会正常。

#### 原因

之前提到在生成对应的model时，model会持有其对应的asset。在viewWillAppear时会根据对应的asset进行解析，得到UIImage展示。

出现这个问题是以下因素造成的：

* 解析asset获取UIImage的方法本身是异步的，返回的时间不固定，可能后一个的image先返回，就错误的赋给了前一个cell
* 获取图片方法也是异步的，因此考虑后续的操作在回调中做。同时要做没有截图文件夹就读取recent文件夹的兜底。在最初的设计中，就变成了以下的代码

  ```text
        PhotoLibrary.fetchScreenshotsAssets(mediaType: .image) { [weak self] (fetchResult, _, _) in
            if fetchResult.count > 0 {
                self?.reloadAssets(fetchResult)
            }else{
                PhotoLibrary.fetchRecentAlbumAssets(mediaType: .image) { [weak self] (fetchResult, _, _) in
                    self?.reloadAssets(fetchResult)
                }
            }

        }
  ```

  这样带来的问题就是，如果没有截图文件夹，那么recent里面的reload方法实际上会reload四次。

上面两点最终造成了这个问题，因为要reload多次，每一次的异步图片解析加载都耗时较长，最终体现是，有可能前面的reload是成功的，但是最后一次会失败，就会造成图片的错乱。

#### 解决

解决方案同样是一波三折。一个解决思路是，既然在viewWillAppear中解析会造成错乱，那是否可以把整个解析的过程提前。在生成model的时候直接得到对应的UIImage，直接赋给model。

想法很美好，但这个过程耗时很长，并且占内存。一旦用户图片量变大，进入这个页面就会卡顿，最后OOM。

我们这里只能做出判断，如果保持原来的逻辑，当图片willAppear的时候，才调用解析方法。那么实际上在用户滑动的时候才会调用这个耗时的方法，在这个场景中一屏显示3张图片，带来的压力较小。并且在第一次加载之后，系统会做一层缓存，并且如果系统没有做，我们也可以在model中附加一个img属性来存储对应的img，之后也就不需要解析。

保持此处逻辑不动，就必须在reload次数上面做减少。

```swift
    public func fetchAssets(mediaType: BNImagePickerMediaType,
                            collectionType: BNImagePickerFromCollectionType,
                            completion: @escaping (_ assets: PHFetchResult<PHAsset>, _ collectionName: String) -> Void) {
        guard PHPhotoLibrary.authorizationStatus() == .authorized else { return }

        DispatchQueue.global().async {
            var assets = PHFetchResult<PHAsset>()
            var collectionName: String = ""
            let fetchOptions = PHFetchOptions()

            switch collectionType {
            case .screenshots:
                let screenshotsCollection = PHAssetCollection.fetchAssetCollections(with: .smartAlbum, subtype: .smartAlbumScreenshots, options: nil)
                guard let collection = screenshotsCollection.firstObject else {
                    return
                }
                collectionName = collection.localizedTitle ?? ""
                fetchOptions.predicate = NSPredicate(format: "mediaType == %d OR mediaType == %d",
                                                     PHAssetMediaType.image.rawValue,
                                                     PHAssetMediaType.video.rawValue)
                assets = PHAsset.fetchAssets(in: collection, options: fetchOptions)
            case .recent:
                let smartAlbums = PHAssetCollection.fetchAssetCollections(with: .smartAlbum, subtype: .smartAlbumUserLibrary, options: nil)
                guard let collection = smartAlbums.firstObject else {
                    return
                }
                collectionName = collection.localizedTitle ?? ""
                fetchOptions.predicate = NSPredicate(format: "mediaType == %d OR mediaType == %d",
                                                     PHAssetMediaType.image.rawValue,
                                                     PHAssetMediaType.video.rawValue)
                assets = PHAsset.fetchAssets(in: collection, options: fetchOptions)
            }

            DispatchQueue.main.async {
                completion(assets, collectionName)
            }
        }
    }
```

问题得到了解决，虽然从逻辑上想，异步加载的问题并没有解决，但是上述的现象也无法复现了。只能归结于，因为reload次数减少，异步加载图片的次数减少，在这有限次的次数中，机器有足够的性能去handle。

### Learning

* 涉及到IO操作的API多半会是异步的操作
* 涉及到异步操作就要考虑时序问题，错乱、拿不到数据、丢失，往往可能是异步的锅。通过回调去处理这些异步操作之后要干的事情。
* 多线程一定要注意线程安全问题。保证其中的每一个类和数据结构都是线程安全的。多线程虽然好用，但是要小心。
* 如果一行代码没有用，当然一定要确保没用。就删掉它。谁知道哪块云上有雨。

