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


## 注册KVO

必须执行以下步骤来使对象能够接收键值观察通知：
- 使用`addObserver:forKeyPath:options:context:`方法将观察者注册到被观察对象。
- 在观察者内部实现`observeValueForKeyPath:ofObject:change:context:`方法来接收更改通知消息。
- 当观察者不需要在接收更改通知消息时，使用`removeObserver:forKeyPath:`方法取消注册观察者。

> **重要**：并非所有类的所有属性都是兼容KVO的。可以按照[KVO兼容](turn)中描述的步骤来确保自己的类是兼容KVO的。

### 注册为观察者

观察者对象首先通过发送一个`addObserver:forKeyPath:options:context:`消息并传递其本身以及被观察的属性的键路径，来向被观察对象注册自己。观察者还指定了一个`options`参数和一个上下文指针来管理通知。

#### Options

选项参数会影响通知中提供的变更字典的内容以及生成通知的方式。

观察者对象可以使用选项`NSKeyValueObservingOptionOld`来接收被观察属性被更改之前的值，以及使用选项`NSKeyValueObservingOptionNew`请求属性的新值。还可以使用按位`OR`组合这些选项，以便同时接收新值和旧值。

选项`NSKeyValueObservingOptionInitial`指示被观察的对象在`addObserver:forKeyPath:options:context:`方法返回之前发送一个**立即更改**通知，可以使用此额外的一次性通知来在观察者对象中设置属性的初始值。

选项`NSKeyValueObservingOptionPrior`指示被观察对象在属性更改之前发送一个通知。如果通知提供的变更字典中包含一个键`NSKeyValueChangeNotificationIsPriorKey`，且键对应的值为一个包装了`YES`的`NSNumber`对象，则表示这是一个**预更改**通知。当观察者对象自己也要兼容KVO并且需要调用依赖于其他对象的被观察属性的属性的一个`willChange...`方法时，可以使用此**预更改**通知。寻常的**已更改**通知生成得太晚，会导致无法及时调用`willChange...`方法。

#### Context

`addObserver:forKeyPath:options:context:`消息中的上下文指针包含将在相应的更改通知中回传给观察者对象的任意数据。可以指定该参数为`NULL`并完全依赖于键路径字符串来确定更改通知的接收者，但是这种方式可能会在观察者对象的父类因为不同的原因也观察相同的键路径时出现问题。

更安全和更可扩展的方法是使用上下文来确保收到的通知是发送给观察者对象的，而不是发送给观察者对象的父类。

在类中唯一命名的静态变量的地址是一个很好的上下文，并且在父类或者子类中以相同方式选择的上下文不会重叠。可以为整个类选择单独一个上下文，并依赖通知消息中的键路径字符串来确定更改的内容。或者，可以为每个被观察的键路径创建不同的上下文来完全绕过字符串比较的需要，从而实现更有效的通知解析。以下代码显示了以这种方式选择的`balance`和`interestRate`属性的示例上下文：
```
static void *PersonAccountBalanceContext = &PersonAccountBalanceContext;
static void *PersonAccountInterestRateContext = &PersonAccountInterestRateContext
```
以下代码演示了`Person`实例如何使用给定的上下文指针将自身注册为`Account`实例的`balance`和`interestRate`属性的观察者。
```
- (void)registerAsObserverForAccount:(Account*)account 
{
    [account addObserver:self forKeyPath:@"balance" options:(NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld) context:PersonAccountBalanceContext];

    [account addObserver:self forKeyPath:@"interestRate" options:(NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld) context:PersonAccountInterestRateContext];
}
```
> **注意**：键值观察方法`addObserver:forKeyPath:options:context:`不会保留对观察者对象、被观察对象或者上下文的强引用，我们应该自己在代码中确保在必要时保留对观察者对象、被观察对象或者上下文的强引用。


### 接收更改通知

当对象的一个被观察的属性的值改变时，观察者对象会收到一个`observeValueForKeyPath:ofObject:change:context:`消息。所有的观察者对象都必须实现这个方法。

观察者对象提供了触发通知的键路径、作为相关对象的自身、包含有关更改的详细信息的字典以及观察者为键路径注册时提供的上下文指针。

变更字典中的条目`NSKeyValueChangeKindKey`提供了与所发生更改的类型有关的信息。如果被观察对象的值已经改变，`NSKeyValueChangeKindKey`条目会返回`NSKeyValueChangeSetting`。根据注册观察者时指定的选项，变更字典中的`NSKeyValueChangeOldKey`条目和`NSKeyValueChangeNewKey`条目包含更改之前和之后的属性值。如果属性是一个对象，则直接提供该值。如果属性是一个标量或者结构体，该值会被包装在`NSValue`对象中。

如果被观察的属性是一个to-many relationship，则`NSKeyValueChangeKindKey`条目分别通过返回`NSKeyValueChangeInsertion`、`NSKeyValueChangeRemoval`或者`NSKeyValueChangeReplacement`来表明关系中的对象是否是被插入、移除或者替换的。

变更字典的`NSKeyValueChangeIndexesKey`条目是一个指定关系中已更改的索引的`NSIndexSet`对象。如果在注册观察者时，将`NSKeyValueObservingOptionNew`或者`NSKeyValueObservingOptionOld`指定为选项，变更字典中的`NSKeyValueChangeOldKey`和`NSKeyValueChangeNewKey`条目是一个包含更改之前和之后的相关对象的值的数组。

以下示例显示了`Person`观察者的用于记录属性`balance`和`interestRate`的旧值和新值的`observeValueForKeyPath:ofObject:change:context:`方法实现。
```
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context 
{
    if (context == PersonAccountBalanceContext) {
        // Do something with the balance…

    } else if (context == PersonAccountInterestRateContext) {
        // Do something with the interest rate…

    } else {
        // Any unrecognized context must belong to super
        [super observeValueForKeyPath:keyPath ofObject:object change:change context:context];
    }
}
```
在注册观察者时，如果指定一个`NULL`作为上下文，则将通知的键路径与正在观察的键路径相比较来确定更改的内容。如果对所有被观察的键路径使用单独一个上下文，则首先将通知的上下文与其相比较，找到匹配项后再使用键路径字符串比较来确定具体更改的内容。如果为每个键路径提供唯一的上下文，如上所示，一系列简单的指针比较会同时告诉我们通知是否适用于此观察者以及如果是，则哪个键路径已经更改了。

在任何情况下，当观察者不能确定更改通知是否适用于自己（不能识别上下文或者键路径）时，观察者应该总是调用其父类的`observeValueForKeyPath:ofObject:change:context:`实现。

> **注意**：如果通知传递到类层次结构的顶部，则`NSObject`会抛出一个`NSInternalInconsistencyException`。

### 移除观察者

通过向被观察对象发送一个`removeObserver:forKeyPath:context:`消息并指定观察者对象、键路径和上下文来移除观察者。 以下示例显示了移除`balance`和`interestRate`的观察者`Person`。

```
- (void)unregisterAsObserverForAccount:(Account*)account 
{
    [account removeObserver:self forKeyPath:@"balance" context:PersonAccountBalanceContext];

    [account removeObserver:self forKeyPath:@"interestRate" context:PersonAccountInterestRateContext];
}
```
收到`removeObserver:forKeyPath:context:`消息后，观察者对象将不会接收到指定的键路径和对象的任何`observeValueForKeyPath:ofObject:change:context:`消息。



