一个MVVM的学习记录和感悟，我自己也在慢慢学习中，这个DEMO会慢慢完善并且会把学习中需要注意的东西记录下来。感谢关注阅读

## 首先先谈谈MVVM是个什么东西，以及为什么要使用MVVM
在刚接触IOS开发的时候，各种教科书上面都写着IOS是原生的MVC结构，并且按照IOS开发Kit所设计的，IOS中有针对M设计的NSObject类、针对V设计的UIView类、以及针对C设计的UIViewController类，很明显IOS的开发kit就是MVC，我也欣然的接受以所谓的MVC的方式去开发IOS应用程序。

但是现实与理想毕竟是由差距的，虽然IOS开发kit设计了三个看上去分离的类，但其实UIView和UIViewController往往是不分离的，UIViewController中原生就带着一个UIView的成员变量（也就是IOS应用程序中的每一个界面），So －UIViewController其实更像是UIView的管理器，或者说UIViewController封装了一层UIView，所有的数据处理和逻辑全部就写在了UIViewController这个类中，所以其实所谓的IOS中MVC其实就是M（VC）两个部分，后面的VC指的是UIViewController，由于所有的业务逻辑都在这个类中，这样就导致了在UIViewController这个类不伦不类，代码混乱，在这个类中代码的量是非常多的，理想中的MVC中V和C这两个部分都是可以高度复用的，然而因为这里V和C粘在了一个就导致了这里的V也很难被复用（关于复用其实还设计到胖model和瘦model，这里我不做介绍，想了解清楚的自行google），基于这两个让代码强迫症无法忍受的缺点，就想让MVC真正的MVC起来、于是就从（VC）中把逻辑给分离了出来，让（VC）只做一个纯粹的VIew————就是MVVM中的V，而分离出去的逻辑就是VM，M还是那个M；

在平常开发的过程中，view初始化的时候一般要传入一个vm，所以在view中拥有一个VM，但是VM是不能拥有view的，所以这里的拥有关系是单向的。view需要向用户动态展现从vm中不断变化的数据，同样用户所有对于view的操作都写入vm，也就是说view现在变得特别的单纯，单纯的只认识VM一个人，view所有的行为都是在跟VM进行交互。view在这其中的职能就是：
**动态展示VM中不断变化的数据给用户看**
**接收用户对view的操作并写入VM**

上面说到的一个view的职能:动态展现VM中不断变化的数据给用户看，这句话理解为，view自己能及时地更新显示的数据就是更新UI。

但是但是、数据的处理逻辑在是在vm中，而vm中是没有拥有view的，那在vm中逻辑处理完了以后，view自己是怎么更新自己的呢？这就要用到KVO来监听VM的变化了，幸运的是ReactiveCocoa（以下简称RAC）的出现，简化封装了大量这些操作、也彻底把MVVM搬上了IOS开发的大舞台。接下来是大篇幅的RAC教程
----------
刚开始接触RAC肯定是一头的雾水，或许你已经上网看过了一些RAC的简单教程，但你依然不懂RACSignal、RACCommand这些到底是什么东西，你可能知道了signal是水管的比喻或者是排插的比喻。等你真正理解了signal以后觉得这些比喻确实是挺确切的，但是一开始学习的时候这些比喻着实还是不太好理解。

RAC一个重要的优点就是它提供了统一的方法来处理等待。

等待？等待就是等待指定事情的发生，在等待阶段其实是什么都不做的，在等待的事情到来的时候它就会往外通知。在前端开发中其实最多的就是等待，包括异步行为，包括委托方法，回调blocks，target-action机制，通知和KVO。这些东西的本质其实都是等待，他们都最终告诉你事情发生了然后你处理接下来的逻辑。

而这个统一的方法就是RACSignal，所有的等待都可以转成一个RACSignal对象，RACSignal其实就是一个能通知指定事情的到来并传递数据给订阅者的一个东东
而且有一类RACSubject的signal自己本身就可以充当一个订阅者

上面的第一点提到了传递数据，实现起来就是简单的消息发送传参数的过程也就是调用订阅者的方法，显然一个signal的实例中就必须包含着它订阅者的实例才能在**需要的时候**调用订阅者的方法来为这个订阅者传递数据，看一段signal创建的代码

RACSignal *signal = [RACSignal createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
[subscriber sendNext:@"abner"];
[subscriber sendCompleted];
return nil;
}];
创建signal的时候传如一个block，block中要求返回一个RACDisposable的对象（这个东西后面再说），而block中传给你用的参数是一个实现了RACSubscriber协议的对象，这个协议也就是订阅者协议，也就是说这里传给你使用的参数是一个这个signal的订阅者，至于订阅者为什么要用协议的实现方式是因为在这之中各种各样的实例都可以是订阅者（比如我上面提到的那类signal本身就是一个实现了这个协议的订阅者）。回到这个block本身，这个block里面就是订阅这个信号后这个信号要做的事情，主要就只是做了一大件事情就是把数据传递给这个signal的订阅者，我们把要传递的数据分为三大类并通过不同的方法传参数给订阅者，分别是：

普通数据————通过调用订阅者的sendNext:方法来传递普通数据
错误数据————通过调用订阅者的sendError：方法来传递错误数据，传递error数据的同时也做了complete的处理
完成标志数据————通过调用订阅者的sendComplete方法来传递这次的订阅处理已经完成,不会继续传递数据给这个订阅者，会自动调用订阅的释放以及后续的清理工作

也就是说每次订阅都会执行这个block来把这个信号拥有的，或者产生的三种类型的数据传递给这个订阅者，并附带做一些相应的处理。

cancat flatmap then 区别

