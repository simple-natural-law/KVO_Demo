# KVO键值观察（Key-Value Observing）


## 引言

键值观察提供了一种机制以允许对象通知其他对象的特定属性的更改，它对应用程序中的模型和控制器层之间的通信特别有用。在OS X中，控制器层绑定技术严重依赖于键值观察。控制器对象通常观察模型对象的属性，视图对象通过控制器观察模型对象的属性。 此外，模型对象还可以观察其他模型对象（通常用于确定依赖的值何时改变），甚至其自身（再次确定依赖的值何时改变）。

可以观察 simple attributes、to-one relationships 和 to-many relationships 这三种类型的属性（关于属性类型的描述，请参看[KVC键值编码（Key-Value Coding）](https://www.jianshu.com/p/13a17bcea48b)）。to-many relationships 属性的观察者被告知所做的更改的类型以及更改涉及哪些对象。

一个简单的例子说明了KVO如何在应用程序中发挥作用的。假设一个`Person`对象和一个`Account`对象代表某个人在银行的储蓄账户，那么`Person`实例可能需要知道`Account`实例的某些详情合适发生变化，例如余额和利率。

![图1-1](https://upload-images.jianshu.io/upload_images/4906302-995a319aca710939.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果这些属性是`Account`的公开属性，那么`Person`可以定期轮询`Account`以发现变化。但这种做法的效率非常低，并且通常是不切实际的。更好的做法是使用KVO，这类似于在发生更改时，`Person`接收一个中断。

要使用KVO，首先必须确保被观察的对象兼容KVO。通常情况下，如果对象继承自`NSObject`并且以常规方式创建属性，对象及其属性将自动兼容KVO。还可以手动实现KVO兼容。[KVO兼容](#turn)描述了自动和手动键值观察之间的区别，以及如何实现它们。

接下来，必须注册观察者`Person`实例和被观察的`Account`实例。`Person`发送一个`addObserver:forKeyPath:options:context:`消息给`Account`，对于每个观察到的键路径，将其自身命名为观察者。

![图1-2](https://upload-images.jianshu.io/upload_images/4906302-5c1ac4a74864d953.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了从`Account`接收更改通知，`Person`实现了所有观察者都需要的`observeValueForKeyPath:ofObject:change:context:`方法。只要注册的键路径的其中一个发生变化，`Account`就会发送`observeValueForKeyPath:ofObject:change:context:`消息给`Person`。然后，`Person`能够基于更改通知采取适当的措施。

![图1-3](https://upload-images.jianshu.io/upload_images/4906302-fed4791e2d6e8bf5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后，当`Person`不再需要通知时，并且在它还没被释放之前，`Person`实例必须通过向`Account`实例发送`removeObserver:forKeyPath:`消息来取消注册。

![图1-4](https://upload-images.jianshu.io/upload_images/4906302-7bede960301157d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


KVO的第一益处是不必在每次属性更改时都实施自己的方案来发送通知。其定义良好的基础架构具有框架级别的支持，使得其易于使用——通常不必向项目添加任何代码。此外，基础架构的功能已经齐全，这使得单个属性以及依赖的值支持多个观察者变得容易。

与使用`NSNotificationCenter`的通知不同，没有中心对象为所有观察者提供更改通知。 取而代之的是，在进行更改时将通知直接发送到观察对象。`NSObject`提供了键值观察的基本实现，很少需要覆盖这些方法。
