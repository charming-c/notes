# Android 系统源码的软件体系结构分析

## 一、Android 系统介绍

​	Android是一个基于Linux内核与其他开源软件的开放源代码的移动操作系统，由谷歌成立的开放手持设备联盟持续领导与开发。该操作系统的核心为Android开源项目（英语：Android Open Source Project），简称“AOSP”, 是根据Apache许可证授权的免费开源软件。然而，大多数设备使用谷歌开发的专有Android版本，预装谷歌移动服务等专有软件；虽然AOSP是免费的，但“Android”名称和徽标是谷歌的商标，谷歌可以限制未经认证的设备不得使用Android品牌和谷歌的专有版本。Android最初由安迪·鲁宾等人开发制作，最初开发这个系统的早期方向是创建一个数字相机的先进操作系统，但是后来发现相机市场规模不足实现其目标，转而开发智能手机系统，于是Android成为一款面向智能手机的操作系统。于2005年7月11日被美国科技企业Google收购。2007年11月，Google与84家硬件制造商、软件开发商及电信营运商成立开放手持设备联盟来共同研发Android，随后，Google以Apache免费开放源代码许可证的授权方式，发布了Android的源代码，开放源代码加速了Android普及，让生产商推出搭载Android的智能手机，Android后来更逐渐拓展到平板电脑及其他领域上。2010年末数据显示，仅正式推出两年的Android操作系统在市场占有率上已经超越称霸逾十年的诺基亚Symbian系统，成为全球第一大智能手机操作系统。在2014年Google I/O开发者大会上Google宣布过去30天里有10亿台活跃的Android设备。2017年3月，Android全球网络流量和设备超越Microsoft Windows，正式成为全球第一大操作系统。

## 二、 分层的软件体系结构模式

### 1. 分层体系结构

​	Android的系统架构和其操作系统一样，采用了分层的架构。从架构图看，Android分为四个层，从高层到低层分别是应用程序层、应用程序框架层、系统运行库层和Linux内核层。其层次结构如图1所示：

![图1](/Users/charming/Library/Application Support/typora-user-images/image-20240222174931324.png)

1. Android应用层：这一层包含了一系列核心应用程序包，如Email客户端、SMS短消息程序、日历、地图、浏览器和联系人管理程序等。这些应用程序都是使用Java编写的，为用户提供了丰富的功能体验。应用层程序的各项功能均是通过调用下一层应用框架层的接口来进行实现和扩充。
2. Android应用框架层：这一层为开发人员提供了完整的API框架，使他们可以访问核心应用程序所使用的组件。开发人员可以使用视图（Views）来构建应用程序，包括列表（Lists）、网格（Grids）、文本框（TextBoxes）、按钮（Buttons）甚至可嵌入的Web浏览器。此外，还有资源管理器（ResourceManager）提供非代码资源的访问，如本地字符串、图形和布局文件（LayoutFiles）。通知管理器（NotificationManager）允许应用程序在状态栏中显示自定义提示信息。活动管理器（ActivityManager）则用于管理应用程序生命周期并提供常用的导航回退功能。
3. Android系统运行库层：这一层包含了各种程序库，这些库实际上是用C/C++编写的，并被编译为共享库的形式供应用程序使用。这些库为应用程序提供了一系列服务，如图形渲染、网络通信、数据库操作等。不同于java语言在编译运行时要通过字节码运行在Java运行时环境中，C/C++可以直接与内核层交互，通过使用这些库，开发人员可以更加高效地开发出性能优越的应用程序。
4. Linux内核层：作为Android系统架构的最底层，Linux内核层为整个系统提供了核心功能。它负责管理硬件资源、安全性和驱动程序等底层任务。由Android基于Linux，因此它继承了许多Linux内核的特性，包括强大的安全机制和模块化设计。

### 2. 功能性和非功能性特性

​	此外，Android系统架构的灵活性使其不仅限于ARM平台。通过编译控制，Android可以在X86、MAC等体系结构的机器上运行，从而实现了跨平台的兼容性。这种分层架构使得Android系统具有高度的可扩展性和可移植性，为开发人员提供了广阔的创新空间。在实际应用中，开发人员可以利用Android系统架构的特点来优化应用程序的性能。例如，在应用框架层，合理利用视图组件和资源管理器可以创建出更加流畅的用户界面。在系统运行库层，利用高效的库函数可以提升应用程序的处理能力。在应用层，根据不同的需求定制功能模块，以满足不同用户群体的需求。Android支持基于抽象程度递增的系统设计，使得开发者在设计软件或者开发软件时将应用功能不仅仅局限在某一个层次的某一块位置，而是将功能组合和增强，同时也提高了代码的重用。

## 三、事件驱动的软件结构风格

​	Android作为一款主要用于手机等交互式设备的操作系统，其中的应用程序既包含UI，同时也包含UI相关的处理逻辑。为了一个应用可以更好的维护和扩展，需要更加清晰的区分相关层级和功能划分，否则在数据获取多样的实际场景中（例如网络请求和本地数据库），每一次变更都需要修改相关的UI处理逻辑。当各种功能的代码杂糅在一起，没有很好解耦时，维护和扩展的成本几句增加，在实际的Android开发中各种各样的软件架构在版本的迭代中出现。Android架构，即为开发android时使用的架构。Android的开发一般分为三部分：UI逻辑，业务逻辑和数据操作逻辑。Android架构，就是为了更好地协调这三者的关系。达到：

1. 各模块高内聚低耦合的状态，方便进行团队分工合作开发。
2. 代码思路清晰，提高代码的可维护性与可测试性。
3. 减少样板代码，提高开发效率，减少开发错误。

在早期的Android开发中，缺乏框架的支撑，但是也涌现了不同的架构模式来适应不同的开发情景，如MVC，MVP等等。但由于没有历史的沉淀，各种架构模式的弊端也渐渐浮出水面。在这种情境下，谷歌推出了架构组件，用成熟的框架来减少样板代码，提高开发效率，有如SpringMVC的风范，这就是MVVM的框架实现。

### 1. MVC 设计模式

​	MVC全名是Model View Controller，是模型(model)－视图(view)－控制器(controller)的缩写，一种软件设计典范，用一种业务逻辑、数据、界面显示分离的方法组织代码，将业务逻辑聚集到一个部件里面，在改进和个性化定制界面及用户交互的同时，不需要重新编写业务逻辑。MVC被独特的发展起来用于映射传统的输入、处理和输出功能在一个逻辑的图形化用户界面的结构中。其模式架构图如图2所示：

![图2](/Users/charming/Library/Application Support/typora-user-images/image-20240222182000911.png)

其中各个部分职能如下：

- View：负责与用户交汇，显示界面。
- Controller：负责接收来自view的请求，处理业务逻辑。
- Model：负责数据逻辑，网络请求数据以及本地数据库操作数据等。

在MVC架构中，Controller是业务的主要承载者，几乎所有的业务逻辑都在Controller中进行编写。而View主要负责UI逻辑，而Model是数据逻辑，彼此分工。MVC的本质就是按照UI逻辑、业务逻辑、数据逻辑不同的职责分三大模块，彼此分工。

​	在Android中，view一般使用xml进行编写，但xml的能力不全面，需要Activity进行一些UI逻辑的编写，因而MVC中的V即为xml+Activity。Model数据层，在Android中负责网络请求和数据库操作，并向外暴露接口。Controller是争议比较多的写法：一种是直接把Activity当成Controller；一种是独立出Controller类，进行逻辑分离。MVC架构的处理流程一般是：

- view接收用户的点击
- view请求controller进行处理或直接去model获取数据
- controller请求model获取数据，进行其他的业务操作，这一步可以有多种做法：
    - 利用callBack从controller进行回调
    - 把view实例给controller，让controller进行处理
    - 通知view去model获取数据

MVC直接在Activity中书写业务逻辑，只抽离出Model层。这样看起来很美好，但是有严重的问题。MVC的核心就是按照职责分离代码。但几乎所有的业务逻辑代码都在controller中，当项目越来越大时会导致controller极度臃肿，难以维护。view与model直接依赖，模块之间依赖不单一，view因直接通过model获取数据，不可避免的会耦合一些业务代码。但MVC也有他的好处：简单。他不需要写很多的代码来让代码解耦，这在小型项目非常有用。

### 2. MVP 设计模式

相比MVC，MVP的更加的完善。MVP全名是Model-View-Presenter。其模式架构图如下图3所示：

![图3](/Users/charming/Library/Application Support/typora-user-images/image-20240222194353418.png)

其中各个部分职能如下：

- View：UI模块，负责界面显示和与用户交汇。
- Presenter：负责业务逻辑，起着连接View和Model桥梁的作用。
- Model：专注于数据逻辑。

MVP和MVC的区别很明显就在这个Presenter中。为了解决MVC中代码的耦合严重性，把业务逻辑都抽离到了Presenter中。这样View和Model完全被隔离，实现了单向依赖，大大减少了耦合度。view和prensenter之间通过接口来通信，只要定义好接口，那么团队可以合作同时开发不同的模块，同时不同的模块也可以进行独立测试。也因各模块独立了，所以要只要符合接口规范，即可做到动态更换模块而不需要修改其他的模块。

​	在Android中，需要让Activity提供控件的更新接口，prensenter提供业务逻辑接口，Activity持有presenter的实例，prensenter持有Activity的弱引用（不用直接引用是为了避免内存泄露），Activity直接调用prensenter的方法更新界面，prensenter去model获取数据之后，通过view的接口更新view。其结构如下图4所示：

![图4](/Users/charming/Library/Application Support/typora-user-images/image-20240222195655597.png)

​	不同的view可以通过实现相同的接口来共享prensenter。presenter也可以通过实现接口来实现动态更换逻辑。Model是完全独立开发的，向外暴露的方法参数中含有callBack参数，可以直接调用callBack进行回调。MVP的最大特点就是接口通信，接口的作用是为了实现模块间的独立开发，模块代码复用以及模块的动态更换。但是我们会发现后两个特性，在Android开发中使用的机会非常少。presenter的作用就是接受view的请求，然后再model中获取数据后调用view的方法进行展示，但是每个界面都是不同的，很少可以共用模块的情景出现。这就导致了每个Activity/Fragment都必须写一个IView接口，然后还需要再写个IPresenter接口，从而产生了非常多的接口，需要编写大量的代码来进行解耦。如果在小型的项目，这样反而会大大降低了开发效率。 其次，prensenter并没有真正解耦，他还需要调用view的接口进行UI操作，解耦没有彻底。MVP也没有解决MVC中Controller代码臃肿的问题，甚至还把部分的UI操作带到了Presenter中。因此，由于MVP有：

- 过度设计导致接口过多，编写大量的代码来实现模块解耦，降低了开发效率

- 并没有彻底进行解耦，prensenter需要同时处理UI逻辑和业务逻辑，presenter臃肿

这样的缺点，android开发者都在寻找一个更加完善的架构模式。就如同下面的MVVM了，但是其实还有如AAC等结构模式的存在，但因他们的局限性以及上手难度，并没有被广泛使用。而到了MVVM，谷歌通过一系列的架构组件来让开发者可以简单地实现MVVM架构。

### 3. MVVM 设计模式

​	Android开发长期以来，没有规范的框架支撑，虽然可以使用这些框架和技术来达到快速迭代的目的，但是越来越杂的技术选型也让Android开发者无从选择，最终导致做出来的应用质量参差不齐，同时采用各种各样的架构模式也出现许多的弊端，Android一直没有制定一个规范来解决这一问题；终于，在Goole I/O 2018大会上推出了全新的Android Jetpack应用开发架构。通过使用Google提供的一套强大的Jetpack组件，包括LiveData、ViewModel和Data Binding等，可以轻松实现MVVM架构。这些组件可以帮助开发人员更轻松地实现MVVM架构，并提供了许多现代化的开发工具和技术。

​	MVVM，全名为Model-View-ViewModel。其模式架构图如下图5所示：

![图5](/Users/charming/Library/Application Support/typora-user-images/image-20240222200137865.png)

​	MVVM的view和model和前面的两种架构模式是差不多的，重点在ViewModel。viewModel通过将数据和view进行绑定，修改数据会直接反映到view上，通过数据驱动型思想，彻底把MVP中的Presenter的UI操作逻辑给去掉了。而viewModel是绑定于单独的view的，也就不需要进行编写接口了。但viewModel中依旧有很多的业务逻辑，但是因为把view和数据进行绑定，这样可以让view和业务彻底的解耦了。view可以专注于UI操作，而viewModel可以专注于业务操作。因而，MVVM通过数据驱动型思想，彻底把业务和UI逻辑进行解耦，各模块分工职责明确。View只需要关注Viewmodel的数据部分，而无需知道数据是怎么来的；而ViewModel只需要关注数据逻辑，而不需要知道UI是如何实现的。View可以随意更换UI实现，但ViewModel却完全不需要改变。但依旧存在的问题是：viewModel会依旧很臃肿；需要一个绑定框架来对view和数据对象进行绑定。这是MVVM的两大弊端。上面的两种架构模式都是不需要框架的，但MVVM必须要有一个view-data绑定框架，来实现对data的更改可以实时反映到view上，这就造成了需要有一定的上手难度：学习框架。为了解决上面两个问题，需要：1.简单易用的框架；2.为viewModel减少压力。所以谷歌推出了适合android开发的MVVM架构模式，其结构如下图6所示：

![图6](/Users/charming/Library/Application Support/typora-user-images/image-20240222202130692.png)

其中各个部分职能如下：

- View对应的就是Activity和Fragment，在这里进行UI操作。

- ViewModel中包含了LiveData，这是一种可观察数据类型框架。View通过向LiveData注册观察者，当LiveData发生改变时，就会直接调用观察者的逻辑把数据更新到View上。

- ViewModel完全不需要关心UI操作，只需要专注于数据与业务操作。

- Repository代表了Model层，Repository对ViewModel进行了减压，把业务操作般到了Repository中，避免了viewModel臃肿。

- Repository对请求进行判断是要到本地数据库获取还是网络请求获取分别调用不同的模块。
- Room是Android本地数据库的一个封装的SQLite框架
- Retrofit是Android进行网络数据请求的一个第三方HTTP框架

这样，谷歌推出的简单易用的架构框架，使得各个角色内部只需要处理好自己的功能，通过数据驱动的方式，当Model层的数据发生变化时，通过Livedata的观察者模式自动的更新所有与该数据相关的View。 MVVM架构和Jetpack组件为Android应用开发提供了现代化的架构设计和开发体验。通过将UI逻辑和业务逻辑分离，使用ViewModel管理UI状态，以及使用Data Binding实现数据绑定，可以构建更加清晰、可维护和可测试的Android应用。

​	MVC是不同职责代码分离，MVP是在MVC的基础上通过接口通信降低模块间的耦合性，MVC，MVP都是广义上的架构模式，Android只是他们的一个应用场景。MVVM是专注于页面开发的架构模式，更加契合页面开发模式。MVVM的本质是把View和view要显示的数据对象进行分离绑定，通过数据驱动型思想彻底解耦UI和业务逻辑。前提要有view也就是页面，还有数据。在Android开发层面，MVVM和Jetpack的出现为之后的开发提供了一套标准和模版，为大型项目的开发、构建和维护带来了巨大的革新。

### 4. MVVM 中的数据驱动（事件驱动）

​	在MVVM架构思想中，主要是通过Model中数据更新的事件驱动去进行View的更新，而Model中的数据被ViewModel通过LiveData的形式持有，LiveData通过使用观察者模式这种设计模式，当数据更新时，会通知所有观察此LiveData的观察者，而在此架构中，观察LiveData的观察者一般为View，而被通知的View就可以在这个阶段进行界面的更新。

#### （1）观察者模式

观察者模式是一种行为设计模式， 允许你定义一种订阅机制， 可在对象事件发生时通知多个 “观察” 该对象的其他对象。观察者模式包含观察目标和观察者两类对象，一个目标可以有任意数目的与之相依赖的观察者，一旦观察目标的状态发生改变，所有的观察者都将得到通知。观察者模式的别名包括发布-订阅（Publish/Subscribe）模式、模型-视图（Model/View）模式、源-监听器（Source/Listener）模式或从属者（Dependents）模式。其结构如下图7所示：

![观察者模式结构图](/Users/charming/Library/Application Support/typora-user-images/image-20240223141437487.png)

#### （2）LiveData解析

​	LiveData是一个数据持有类，它可以通过添加观察者被其他组件观察其变更。不同于普通的观察者，它最重要的特性就是遵从应用程序的生命周期，如在Activity中如果数据更新了但Activity已经是destroy状态，LiveData就不会通知Activity(observer)。当然。LiveData的优点还有很多，如不会造成内存泄漏等。LiveData通常会配合ViewModel来使用，ViewModel负责触发数据的更新，更新会通知到LiveData，然后LiveData再通知活跃状态的观察者。LiveData通过observe方法添加观察者。其代码如下图8所示：

![observe源码图](/Users/charming/Library/Application Support/typora-user-images/image-20240222205355300.png)

​	看标注1处，如果我们的Activity/Fragment组件已经是destroy状态的话，将直接返回，不会被加入观察者行列。如果不是destroy状态，就到标注2处，新建一个 LifecycleBoundObserver 将我们的 LifecycleOwner 和 observer保存起来，然后调用 mObservers.putIfAbsent(observer, wrapper) 将observer和wrapper分别作为key和value存入Map中，putIfAbsent()方法会判断如果 value 已经能够存在，就返回，否则返回null。如果返回existing为null，说明以前没有添加过这个观察者，就将 LifecycleBoundObserver 作为 owner 生命周期的观察者。LifecycleBoundObserver 源码如下图9所示：

![LifecycleBoundObserver源码图](/Users/charming/Library/Application Support/typora-user-images/image-20240223103801354.png)

​	LifecycleBoundObserver 继承自 ObserverWrapper 并实现了 GenericLifecycleObserver接口，而 GenericLifecycleObserver 接口又继承自 LifecycleObserver 接口，那么根据 Lifecycle 的特性，实现了LifecycleObserver接口并且加入 LifecycleOwner 的观察者里就可以感知或主动获取 LifecycleOwner 的状态。LiveData通过setValue()方法实现数据的更新，下图10展示setValue()源码：

![setValue源码图](/Users/charming/Library/Application Support/typora-user-images/image-20240223104709589.png)

首先调用assertMainThread()检查是否在主线程，接着将要更新的数据赋给mData，然后调用 dispatchingValue()方法并传入null，将数据分发给各个观察者，如Fragment。dispatchingValue()方法实现如下图11所示：

![dispatchingValu源码图](/Users/charming/Library/Application Support/typora-user-images/image-20240223104810598.png)

从标注1可以看出，dispatchingValue()参数传null和不传null的区别就是如果传null将会通知所有的观察者，反之仅仅通知传入的观察者。我们直接看标注2，通知所有的观察者通过遍历 mObservers ，将所有的 ObserverWrapper 拿到，实际上就是上面提到的 LifecycleBoundObserver，通知观察者调用considerNotify()方法，这个方法就是通知的具体实现，如下图12所示：

![considerNotify源码图](/Users/charming/Library/Application Support/typora-user-images/image-20240223104950263.png)

如果观察者不是活跃状态，将不会通知此观察者，看最后一行，observer.mObserver.onChanged((T) mData)，observer.mObserver就是我们调用LiveData的observer()方法传入的 Observer，然后调用 Observer 的 onChanged((T) mData)方法，将保存的数据mData传入，也就实现了更新。实现一个简单的Observer如下图13所示：

![Observer使用示例图](/Users/charming/Library/Application Support/typora-user-images/image-20240223105248987.png)

如果哪个控件要根据user的变更而及时更新，就在onChanged()方法里处理就可以了。

#### （3）ViewModel解析

​	ViewModel，从字面上理解的话，它肯定是跟视图(View)以及数据(Model)相关的。正像它字面意思一样，它是负责准备和管理和UI组件(Fragment/Activity)相关的数据类，也就是说ViewModel是用来管理UI相关的数据的，同时ViewModel还可以用来负责UI组件间的通信。通常Android系统来管理UI controllers（如Activity、Fragment）的生命周期，由系统响应用户交互或者重建组件，用户无法操控。当组件被销毁并重建后，原来组件相关的数据也会丢失，如果数据类型比较简单，同时数据量也不大，可以通过onSaveInstanceState()存储数据，组件重建之后通过onCreate()，从中读取Bundle恢复数据。但如果是大量数据，不方便序列化及反序列化，则上述方法将不适用。UI controllers经常会发送很多异步请求，有可能会出现UI组件已销毁，而请求还未返回的情况，因此UI controllers需要做额外的工作以防止内存泄露。当Activity因为配置变化而销毁重建时，一般数据会重新请求，其实这是一种浪费，最好就是能够保留上次的数据。UI controllers其实只需要负责展示UI数据、响应用户交互和系统交互即可。但往往开发者会在Activity或Fragment中写许多数据请求和处理的工作，造成UI controllers类代码膨胀，也会导致单元测试难以进行。我们应该遵循职责分离原则，将数据相关的事情从UI controllers中分离出来。通过使用ViewModel，这些问题都可以得到很好的解决。ViewModel对象的范围由获取ViewModel时传递至ViewModelProvider的Lifecycle所决定。ViewModel始终处在内存中，直到Lifecycle永久地离开—对于Activity来说，是当它终止（finish）的时候，对于Fragment来说，是当它分离（detached）的时候。下图14展示左侧为Activity的生命周期过程，期间有一个旋转屏幕的操作；右侧则为ViewModel的生命周期过程：

![生命周期示例图](/Users/charming/Library/Application Support/typora-user-images/image-20240223141459311.png)

​	在官方的ViewModel发布之前，ViewModel层的基类多种多样，内部的依赖和公共逻辑更是五花八门。新的ViewModel组件直接对ViewModel层进行了标准化的规范，即使用ViewModel(或者其子类AndroidViewModel)。同时，Google官方建议ViewModel尽量保证 纯的业务代码，不要持有任何View层(Activity或者Fragment)或Lifecycle的引用，这样保证了ViewModel内部代码的可测试性，避免因为Context等相关的引用导致测试代码的难以编写（比如，MVP中Presenter层代码的测试就需要额外成本，比如依赖注入或者Mock，以保证单元测试的进行）。

​	一般通过如下代码初始化ViewModel：viewModel = ViewModelProviders.of(this).get(UserProfileViewModel.class);this参数一般为Activity或Fragment，因此ViewModelProvider可以获取组件的生命周期。Activity在生命周期中可能会触发多次onCreate()，而ViewModel则只会在第一次onCreate()时创建，然后直到最后Activity销毁。

#### （4）Room解析

​	Room是Google推出的Android架构组件库中的数据持久化组件库, 也可以说是在SQLite上实现的一套ORM解决方案。Room主要包含三个部分：

1. Database : 持有DB和DAO

2. Entity : 定义POJO类，即数据表结构

3. DAO(Data Access Objects) : 定义访问数据（增删改查）的接口

其关系如下图15所示：

![Room结构图](/Users/charming/Library/Application Support/typora-user-images/image-20240223141534095.png)

Database是访问底层数据库的入口，管理着真正的数据库文件。开发时使用@Database定义一个Database类：

- 派生自RoomDatabase
- 关联其内部数据库table对应的entities
- 提供获取DAO的抽象方法，且不能有参数

运行时，可以通过Room.databaseBuilder()或者 Room.inMemoryDatabaseBuilder()获取Database实例。一个Entity代表数据库中的一张表（table）。使用@Entity定义一个Entiry类，类中的属性对应表中的Column：

- 所有的属性必须是public、或者有get、set方法
- 属性中至少有一个主键，使用@PrimaryKey表示单个主键，也可以像下面这样定义多主键
- 当主键值为null时，autoGenerate可以帮助自动生成键值
- 默认情况下使用类名作为数据库table名，也可使用tableName指定
- Entity中的所有属性都会被持久化到数据库，除非使用@Ignore
- 可以使用indices指定数据库索引，unique设置其为唯一索引

DAO提供了访问DB的API，使用@Dao定义DAO类，使用@Query 、@Insert、 @Delete定义CRUD方法。Room中查询操作除了返回POJO对象及其List以外， 还支持：LiveData<T>，LiveData是架构组件库中提供的另一个组件，可以很好满足数据变化驱动UI刷新的需求。Room会实现更新LiveData的代码。Room一般处于MVVM的Model层，主要用于保存一些本地数据，通过和LiveData的绑定，可以轻易地通过ViewModel实现数据驱动的UI刷新。

#### （5）DataBinding解析

​	2015年谷歌I/O大会上介绍了一个数据绑定框架DataBinding。2016年，2017年毫无意外成了项目实战中主流框架。使用它我们可以轻松实现MVVM（模型-视图-视图模型）模式，来实现应用之间数据与视图的分离、视图与业务逻辑的分离、数据与业务逻辑的分离，从而达到低耦合、可重用性、易测试性等好处。而使用DataBinding不仅减少了findViewById的出现频率，而且还大大提高解析XML的速度。DataBinding是通过APT技术来实现的，也就是说，在绑定了布局并且编译之后，编译工具会自动生成一些布局相关的代码，因此，可以打开编译后生成的布局文件来进行分析，具体路径：build\intermediates\data_binding_layout_info_type_merge\debug\out\activity_main.xml，其代码样例如下图16所示：

![xml编译后图](/Users/charming/Library/Application Support/typora-user-images/image-20240223143803532.png)

可以看到，编译工具生成的布局文件定义了了多个Target标签，这些Target的定义，其实就是定义对应的tag，将tag与activity_main.xml布局中的对应的View的id对应起来经过DataBinding变化后的布局，就会有多个tag，同时打开build\intermediates\incremental\mergeDebugResources\stripped.dir\layout\activity_main.xml，竟然可以看到没有任何绑定的的xml布局，其结果如下图17所示：

![xml编译处理后图](/Users/charming/Library/Application Support/typora-user-images/image-20240223143939918.png)

也就是说，DataBinding帮开发者在开发过程中编写的xml布局分成了两个部分，一部分是通过定义Target来把View和id关联起来，另一部分就是去掉了DataBinding的布局本身。

#### （6）Lifecycle解析

​	Lifecycle即生命周期，对于Android开发者来说，其对生命周期可太熟悉了，因为不仅经常在Activity、Fragment的生命周期函数比如onCreate、onResume等中做一些逻辑操作；而且组件的生命周期控制不当的话，可能会导致内存泄漏，或者一些无法预想的结果，比如在后台请求网络操作的协程生命周期，要是大于所需要展示UI的界面的生命周期时，就会导致不必要资源浪费。既然Activity、Fragment、协程、ViewModel、LiveData，甚至十分常用的Handler等所有对象，都需要严格管理生命周期，所以管理这些组件的生命周期就非常重要，这里大概能得到2点关于生命周期的需求：

- 给Activtiy和Fragment这种常用的、显示UI的组件，搞一个统一的生命周期表示规则，这样就可以极大地方便开发者。比如我们都希望在页面对用户可见时，才更新UI，对用户不可见时，更新UI是无用功。但是比如Fragment就比Activity多了onCreateView、onViewCreated等几个生命周期函数，Fragment的View对于可见这个判断就和Activity不一样了，这就导致开发者完成一个功能，需要对接多个组件。

- 可以获取和监控组件的统一生命周期函数变化。比如不论是Activity还是Fragment，我们都希望在回调其onDestroy()方法前，释放一些资源，避免造成内存泄漏和资源浪费。


所以Lifecycle组件就被Google开发和推出了，面对需要对接多个组件的复杂情况，一个有用的方法就是抽象出一层，而Lifecycle的思想也是如此，让本来开发者需要多个组件的情况下，现在就变成了对接一个组件。如下图18所示：

![生命周期组件图](/Users/charming/Library/Application Support/typora-user-images/image-20240223151256913.png)

Lifecycle的工作原理，先以Activity为例来解析源码。Acticity的类关系图如下图19所示：

![Acticity类关系图](/Users/charming/Library/Application Support/typora-user-images/image-20240223151238608.png)

这里可以发现ComponentActivity实现了LifecycleOwner接口，即平时使用Activity就是生命周期持有者。当调用getLifecycle()方法时，其实就是返回ComponentActivity中定义的LifecycleRegistry类型的mLifecycleRegistry成员变量。而且LifecycleRegistry实现了Lifecycle接口，所以这里使用了代理模式，所有操作都由这个LifecycleRegistry来实现。首先分析Activity生命周期事件通过ReportFragment派发给LifecycleRegistry的。其调用关系如下图20所示：

![调用关系图](/Users/charming/Library/Application Support/typora-user-images/image-20240223151635259.png)

上面黄色的Note表示Lifecycle的当前State，由这里我们可以验证最开始说的State含义：其中INITIALIZED初始化状态表示组件已经被创建，但是没有收到ON_CREATE事件，当收到ON_CREATE事件，状态切换到CREATED状态。之所以使用ReportFragment来分发Event，是因为虽然Fragment比Activity多几个生命周期函数，但是丝毫不影响我们使用Fragment的生命周期来派发其所依赖的Activity的生命周期事件。通过查看源码，对应关系如下：

- Activity的ON_CREATE事件在Fragment中的onActivityCreated()方法中分发。
- ON_START、ON_RESUME、ON_PAUSE、ON_STOP和ON_DESTROY都是由其同名生命周期方法中分发。

这里分发事件的方法就是调用LifeycleRegistry的handleLifeEvent()方法。

## 四、数据共享的软件结构风格

​	在Activity或者Fragment中实现数据共享，在Android开发领域一直是一个绕不开的命题，而在Jetpack一系列组件推出后，通过ViewModel实现数据共享的思路成为主流，被Google制定成规范力推。ViewModel 类旨在以注重生命周期的方式存储和管理界面相关的数据。ViewModel类让数据可在发生屏幕旋转等配置更改后继续留存。虽然 onSaveInstanceState 也可以做到保存数据，但是只能是简单数据且数据量有限制，方式是序列化和反序列化。ViewModel 比 onSaveInstanceState 使用简单方便、数据可操作性强。同时实现 Avtivity 和 Fragment、多个 Fragment 之间的数据共享。

在整个生命周期过程中，ViewModel的生命周期会与一个View（也就是Activity或者Fragment）绑定，当View没有被销毁的时候，ViewModel会始终存在在缓存中。在ViewModel源码里，可以通过get方法去拿到某一个缓存在内存中的ViewModel类的实例，若没有则会新建一个。当一个ViewModel与一个Activity的生命周期绑定时，由此Activity产生的所有相关的Fragment都可以通过此Activity拿到相同的ViewModel。而ViewModel中保存的是Model中的数据，也即就是通过ViewModel，Activity与Fragment、Fragment和Fragment都能方便快捷的共享数据。

​	ViewModel中的数据不仅来自于内存（Room）本地的持久化数据库，也可以来自预先保存好的小型数据例如（SharedPreferences），与此同时不同的知识源，例如用户的输入输出，网络请求等都会直接作用在ViewModel中，摒弃人他们之间互不干扰，通过ViewModel这个数据代理媒介进行控制和操作。其结构如下图21所示：

![共享数据结构图](/Users/charming/Library/Application Support/typora-user-images/image-20240223163244455.png)

## 五、解释器风格-将xml文件解析成View对象

### 1. 解释器风格

解释器架构风格是一种软件架构模式，其核心思想是将一个程序解释为一系列指令，然后逐一执行这些指令。以下是解释器架构风格的优缺点：

优点：

1. 灵活性：解释器架构风格可以很容易地添加新的指令和功能，因此非常灵活。这意味着开发人员可以快速地修改和扩展软件，以满足客户的需求。
2. 可移植性：解释器架构不需要编译器来将程序编译成特定平台上的机器码。相反，解释器可以在任何平台上直接运行。这样，开发人员就可以轻松地将应用程序移植到不同的平台上。
3. 交互性：解释器架构非常适合实现交互式应用程序，例如控制台程序和脚本解释器。由于指令可以逐个执行，因此用户可以在执行期间与程序交互，输入或输出数据，或修改程序的行为。

缺点：

1. 效率低：解释器架构的效率通常比编译器低。因为解释器需要逐一解释和执行指令，而不是将程序编译为本地机器码，所以程序的运行速度可能较慢。
2. 稳定性：由于解释器架构的程序不是一次性编译的，而是在运行时解释和执行，因此解释器程序可能更容易出现错误和崩溃。
3. 维护难度：由于解释器架构的程序通常是动态的，并且可以逐步修改和扩展，因此代码维护可能会更加困难。特别是在大型和复杂的应用程序中，开发人员需要确保程序的每个部分都能够正确地解释和执行。

综上所述，解释器架构风格适合需要灵活性、可移植性和交互性的应用程序。在Android系统的软件开发中，对于视图View的构建中，采用了xml布局文件的格式，通过xml文件解析成为对应的View对象，在之后的编程过程中作为对象调用。这么做的好处一是可拖拽可预览，二是语法简单清晰。

### 2. 寻找xml解析入口

在Android中通过LayoutInflater的inflate方法直接将一个xml文件转化为一个View对象，而源码中一共有四个inflate方法，如下图22所示：

![LayoutInflater源码图](/Users/charming/Library/Application Support/typora-user-images/image-20240223173826028.png)

上述显示前三个方法。1处调用3处的方法，先获取Resources对象，再获取XmlResourceParser对象，最终调用2处方法，而2处方法中调用的是第四个方法。第四个方法较长，摘取了核心的注释及逻辑，梳理一下大致流程：

- 先解析XML文件中的Attribute属性，保存在属性集
- 遍历查找到第一个根节点。如果是<merge>，则把父容器作为父布局参数。反之，获取到根*view*，则把根*view*作为父布局参数，走*rInflateChildren*逻辑。
- 根据父容器不为是否为空和是否需要附着在父容器来返回不同结果。

总的来说，inflate方法解析出根节点并根据根节点的类型设置不同的父布局，剩余子节点树递归解析成View树添加到父布局中。

### 3. 递归解析子节点

在之后的递归调用子节点过程中，无论根节点是否是<merge>，最终都会调用*rInflate*方法，只是<merge>不作为子View树的父容器，而是使用其上层的ViewGroup作为容器。其代码如下图23所示：

![LayoutInflater源码图（续）](/Users/charming/Library/Application Support/typora-user-images/image-20240223174639886.png)

1处表明，*rInflateChildren*函数本质上是调用*rInflate*函数处理，只是区分语境而做了不同命名而已。5-7处解析处理特殊标签2-11处为*rInflate*函数的主要逻辑，也是递归解析的关键所在。写一个递归函数，核心是理解递归的终止及触发条件，举一个简单的例子，如下图24所示：

![示例xml解析图](/Users/charming/Library/Application Support/typora-user-images/image-20240223175252135.png)

1. 第一次调用*rInflate*函数，depth=0，解析第一个<LinearLayout>，为START_TAG,实例化LinearLayout对象L1，触发第一次递归。
2. depth=1，解析到第一个<TextView>，为START_TAG，实例化TextView对象，并把自己添加到L1，触发第二次递归。
3. depth=2，解析到第一个</TextView>，为END_TAG不满足while条件，结束第二次递归。
4. 回到第一次递归while循环，depth=2，解析第一个<LinearLayout>，为START_TAG，实例化LinearLayout对象L2，触发第三次递归。
5. depth=2,解析到<TextView/>，实际上这种写法等同于第一个TextView，对于解析器而言，都是按照START_TAG -> TEXT -> END_TAG的解析逻辑。解析到第二个TextView,并把自己添加到L2，触发第四次递归。
6. depth=3，不满足while条件，结束第四次递归。
7. depth=2，解析到第一个</LinearLayout>,为END_TAG不满足while条件，结束第三次递归。
8. depth=1，解析到第二个</LinearLayout>,为END_TAG不满足while条件，结束第一次递归。

上述的例子基本走了一遍*rInflate*函数的逻辑，但是并没有解释如何将节点转化为View对象，其实是通过反射标签。

### 4. 反射标签View

Android通过*createViewFromTag*的方法的把Tag转化为View。大致逻辑为：

1. name转化。如果name是View，则需要转化为对应*class*属性的值；
2. name转成View。如果name是*blink*，则直接构建BlinkLayout并返回，否则先让工厂生成，如果工厂都生成不了，则使用默认生成。

具体到源码中，则是*mFactory*，*mFactory2*，*mPrivateFactory*是优先构建View对象。其中Factory和Factory2实际上是一个暴露构建View方法的接口，而接口对象赋值是在构造器内何set方法中。LayoutInflater是一个抽象类，所以先检查其实现类。

1. AsyncLayoutInflater.java的私有静态内部类BasicInflater
2. PhoneLayoutInflater

其中PhoneLayoutInflater为android系统默认实现的LayoutInflater对象，一般情况下都是他来完成xml文件到View的转化。但是实现类中没有处理Factory，但是有setFactory和setFactory2方法。Factory工厂不能为空且只能设置一次。从1-4处也可知两个*set*方法的调用是互斥的。如果开发者希望设置自定义的工厂，则需要从原来的LayoutInflater中复制一个对象，然后调用*setFactory*或者*setFactory2*方法设置解析工厂。另外*setPrivateFactory*是系统底层调用的，所以不开放对外设置。*setFactory*的调用来自于BaseFragmentActivityGingerbread.java和LayoutInflaterCompatBase.java。

- BaseFragmentActivityGingerbread.java处理低于3.0版本的*activity*版本都需要设置解析*fragmen*标签
- LayoutInflaterCompatBase.java，则暴露FactoryWrapper分装类，根据不同的版本实现不同的LayoutInflaterFactory。常规的业务比如换肤，即可在这里实现自己的Factory达到View换肤效果。

*setFactory2*的调用来自于LayoutInflaterCompatHC.java和LayoutInflaterCompatLollipop。其中LayoutInflaterCompatHC.java实际上内部实现了一个静态的FactoryWrapperHC类，该类继承1-2中的FactoryWrapper类，用来提供HC版本的Factory而已。对于LayoutInflaterCompatLollipop，基本逻辑也是如此。

实际上Factory大致就是一种拦截思想。优先通过选择自定义Factory来构建View对象。那么如果这些Factory都没有的话，则需要走默认逻辑，即LayoutInflater#createView。

### 5. compose 声明式UI

2019 年中，Google 在 I/O 大会上公布了 Android 最新的 UI 框架：Jetpack Compose。Compose 可以说是 Android 官方有史以来动作最大的一个库了。2021年正式发布。在传统的命令式UI开发中，开发人员需要编写大量的代码来描述界面的外观和行为。这些代码通常包括繁琐的布局设置、手动管理UI组件的状态和事件处理逻辑。这种方式容易引发bug，而且代码复杂，不易维护。

与之相反，声明式UI开发采用一种更直观的方式来构建用户界面。在声明式UI中，开发人员只需描述期望的界面外观，而不必关心如何实现。这种方式更加接近人类思维，类似于描述你想要的界面样式，而由框架自动处理底层细节。优势：

1. 简洁明了： 声明式UI代码更加简洁、易于理解，让开发人员专注于界面的外观和交互。
2. 可维护性： 声明式UI减少了手动管理状态和事件的需要，减少了错误和bug的产生，提高了代码的可维护性。
3. 响应式： 声明式UI框架通常支持响应式编程，使界面的状态和数据保持同步，减少了手动更新UI的步骤。
4. 可扩展性： 由于不需要关注底层实现细节，开发人员可以更轻松地进行界面的扩展和修改。

在过去，Android的UI开发使用的是XML布局和View体系结构。虽然这种方式在一定程度上可行，但在处理复杂UI、维护和测试方面存在一些挑战。Jetpack Compose将UI开发变得更加简单、易于维护和高效。它使用Kotlin语言的声明式 UI 模式来简化 UI 开发。在这种模式中，你只需描述 UI 应该如何根据应用的状态进行显示，而 Compose 会在状态发生变化时自动更新 UI。

#### （1）compose底层逻辑
Jetpack Compose 的底层逻辑是基于 Kotlin 的协程和函数式编程的概念。这是一种声明式 UI 框架，它的工作方式与传统的命令式 UI 框架有所不同。Jetpack Compose 使用了一种名为 "recomposition" 的技术来实现这一点。当可观察的状态发生变化时，Compose 会找到依赖这些状态的所有 Composable 函数，并重新调用它们。这就是为什么当你更改 mutableStateOf 的值时，所有使用这个状态的 Composable 函数都会自动更新。

Compose 的底层还使用了一种名为 "diffing" 的技术来优化性能。当 Composable 函数被重新调用时，Compose 会比较新旧 UI 树，并只更新实际发生变化的部分。这意味着即使你的应用有大量的 UI，Compose 也能保持高效的性能。

#### （2）recomposition技术的实现
Jetpack Compose 的 recomposition 技术是通过 Kotlin 的协程和 Compose 编译器插件实现的。这个插件会转换你的 Composable 函数，使它们能够在状态发生变化时重新调用。当你在 Composable 函数中使用 mutableStateOf 和 remember 创建状态时，Compose 编译器插件会自动跟踪这些状态。当状态发生变化时，Compose 会找到所有依赖这些状态的 Composable 函数，并重新调用它们。这就是 recomposition 的基本工作原理。

这个过程是自动进行的，你不需要手动触发 recomposition。你只需要使用 mutableStateOf 和 remember 创建状态，并在 Composable 函数中使用这些状态，Compose 就会自动处理 recomposition。这种自动的 recomposition 机制使得在 Compose 中创建响应式 UI 变得非常简单。你只需要描述 UI 在给定状态下应该如何显示，Compose 会在状态发生变化时自动更新 UI。需要注意的是，recomposition 只会影响依赖发生变化的状态的 Composable 函数。如果一个 Composable 函数不依赖任何状态，或者它依赖的状态没有发生变化，那么这个函数就不会被重新调用。这是 Compose 的一种优化机制，它可以确保只有真正需要更新的部分才会被重新调用。

#### （3）recomposition 的基本工作原理

recomposition 的基本工作原理如下图25所示：

![recomposition工作原理图](/Users/charming/Library/Application Support/typora-user-images/image-20240223184903195.png)

1. 状态跟踪：当你在 Composable 函数中使用 mutableStateOf 和 remember 创建状态时，Compose 编译器插件会自动跟踪这些状态。这意味着 Compose 知道哪些 Composable 函数依赖哪些状态。

2. 状态变化：当状态发生变化时（例如，你调用了 count++ 来增加一个计数器的值），Compose 会知道这个状态已经改变。

3. 重新调用 Composable 函数：Compose 会找到所有依赖已改变状态的 Composable 函数，并重新调用它们。这就是所谓的 "recomposition"。重新调用 Composable 函数会导致 UI 更新，因为 Composable 函数描述了 UI 应该如何根据当前状态进行渲染。

4. 优化：Compose 使用一种称为 "diffing" 的技术来优化 recomposition。当 Composable 函数被重新调用时，Compose 会比较新旧 UI，并只更新实际发生变化的部分。这意味着即使你的应用有大量的 UI，Compose 也能保持高效的性能。

 

流程如下：

- Composable函数使用状态。
- 当状态发生变化时，Compose编译器插件会被触发。
- 编译器插件触发recomposition。
- 在recomposition过程中，所有依赖已改变状态的Composable函数会被重新调用。

这个过程是自动进行的，你不需要手动触发 recomposition。你只需要使用 mutableStateOf 和 remember 创建状态，并在 Composable 函数中使用这些状态，Compose 就会自动处理 recomposition。这使得在 Compose 中创建响应式 UI 变得非常简单。你只需要描述 UI 在给定状态下应该如何显示，Compose 会在状态发生变化时自动更新 UI。


## 六、Android系统的虚拟机

Android应用用Java/Kotlin编写，Android虚拟机并不使用JVM字节码，而是将Class文件通过DX编译器（现已换成D8）编译程dex文件，然后由虚拟机执行。

### 1. Dalvik虚拟机

#### (1) Dalvik虚拟机介绍

Dalvik是Google公司自己设计用于Android平台的虚拟机。Dalvik虚拟机是Google等厂商合作开发的Android移动设备平台的核心组成部分之一。它可以支持已转换为 .dex（即Dalvik Executable）格式的Java应用程序的运行，.dex格式是专为Dalvik设计的一种压缩格式，适合内存和处理器速度有限的系统。Dalvik 经过优化，允许在有限的内存中同时运行多个虚拟机的实例，并且 每一个Dalvik 应用作为一个独立的Linux 进程执行。独立的进程可以防止在虚拟机崩溃的时候所有程序都被关闭。

#### (2) Dalvik诞生消亡史

- Android 1.0，使用Dalvik作为Android虚拟机运行环境。
- Android 2.2，Google在Andriod虚拟机中加入了JIT编译器（Just-In-Time Compiler）。
- Android 4.4，Google带来了全新的虚拟机运行环境ART，此时ART和Dalvik是共存的，用户可以在两者之间进行选择。
- Android 5.0，ART全面取代了Dalvik成为了Android虚拟机运行环境，至此Dalvik退出历史舞台。

#### (3) Dalvik 特点

- Dalvik虚拟机运行的是Dalvik字节码，Dalvik字节码由Java字节码转换而来，并被打包到一个dex文件中。而JVM运行的是class文件或jar文件；
- 加载速度快，dex相比于Jar文件会把所有包含的信息整合在一起，减少了冗余信息。这样就减少I/O操作，提高类的查找速度。
- Dalvik虚拟机是基于寄存器，而JVM是基于栈（操作数栈）。虽然基于寄存器执行效率好，但是可移植性差，难跨平台。
- Dalvik虚拟机允许在有限的内存中同时运行多个进程，每一个应用都运行在一个Dalvik虚拟机实例中，拥有独立的进程空间。
- Dalvik虚拟机有共享机制，不同应用之间在运行时可以共享相同的类，拥有更高的效率。

### 2. ART虚拟机

#### （1）ART概念介绍

- ART虚拟机在Android 5.0开始替换Dalvik虚拟机。其处理应用程序执行的方式不同于Dalvik虚拟机，它不使用JIT而是使用了AOT（Ahead-Of-Time），也就是提前编译技术。并且对垃圾收集器也进行了改进和优化。
- ART虚拟机由Android4.4被引入成为可选项，在Android5.0之后替换掉了Dalvik，并且在Android7.0和8.0分别进行了一系列改动。

#### （2) 基本概念和名词

- .dex文件：App所有java源代码编译后生成众多class文件，由DX/D8，编译为一个/多个（multiDex）dex文件，由Android虚拟机编译执行。
- .odex文件：dex文件经过验证和优化后的产物，art下的odex文件包含经过AOT编译后的代码以及dex的完整内容，但Android8.0之后odex中的dex内容移动到了.vdex文件。
- .art文件：art下根据配置文件生成odex文件时同时生成.art文件，主要是为了提升运行时加载odex中热点代码的速度，包含了类信息和odex中热点方法的索引，运行App时会首先根据这个文件来加载odex中已经编译过的代码。
- 解释器（Interpreter）：用于程序运行时对代码进行逐行解释，翻译成对应平台的机器码执行。
- JIT编译（Just In Time）：由于解释器方式运行太慢引入，对于频繁运行的热点代码（判定标准一般是在某个时间段内执行次数达到某个阈值）进行实时编译（在ART下以方法为粒度）执行，并且缓存JIT编译后的代码在内存中用于下次执行。由于以方法为粒度（ArtMethod）进行编译，JIT编较于解释器可以生成效率更高的代码，运行更快。
- AOT编译（Ahead-Of-Time）：应用安装时全量编译所有代码为本地机器码，运行时直接执行机器码。

#### （3）ART如何运作

##### （1）4.4~7.0

最开始ART只采用AOT编译，在App安装时就编译所有代码存储在本地，打开App直接运行，这样做的优点是应用运行速度变快，缺点也很明显，App安装时间明显变长，而且占用存储空间较大

##### （2）7.0

Android N之后对于ART进行改动，重新引入了JIT编译，结合使用AOT/JIT混合编译，主要机制如下：

- 安装时不进行任何编译，前几次运行仅通过解释器解释运行，同时对热点代码进行JIT编译，并将这些代码的相关信息记录在一个配置文件里
- 设备处于空闲和充电状态时，编译守护进程读取配置文件对热点代码进行AOT编译并写入到app对应的odex文件中
- 再次启动应用后优先使用AOT编译过的代码，否则使用解释器+JIT编译，重复这个过程
- 对于一些庞大的APP，比如某宝，有些功能可能你一辈子都不会用到，根据上述策略这部分代码就不会被编译保存，从而减少了存储空间的占用。另外，在系统升级时也避免了全量编译所有现存应用造成的时间空间消耗。

##### （3）8.0

Android 8.0引入了.vdex文件，它里面包含 APK 的未压缩 DEX 代码，以及一些用于加快验证速度的元数据.

#### 4. ART垃圾收集器优化

- 只有一次GC暂停（Dalvik需要两次）。
- 并发复制，可减少后台内存使用和碎片。
- GC暂停的时间不受堆大小影响。
- 在清理最近分配的短时对象这种特殊情况中，回收器的总GC时间更短。
- 优化了垃圾回收的工效，能够更加及时地进行并行垃圾回收，这使得GC_FOR_ALLOC事件在典型用例中极为罕见。

#### 5. ART时间线

- Android 4.4 ，ART和Dalvik是共存的，用户可以在两者之间进行选择。
- Android 5.0，正式取代Dalvik虚拟机成为Android虚拟机运行环境，Dalvik退出历史舞台，AOT取代JIT。
- Android 7.0，JIT回归，采用JIT和AOP混合编译模式。
- ART持续更新优化

#### 6. Dalvik VM 和 ART VM 有什么区别

- ART早期使用AOT技术，后期使用AOT+JIT混合，而Dalvik使用JIT。
- ART支持64位CPU并兼容32位CPU,而Dalvik只支持32位CPU。
- ART对垃圾收集器进行了改进优化，提高了吞吐量。

## 七、独立构件风格-Kotlin 协程

在 2019 年 Google I/O 大会上，Google宣布今后将优先采用 Kotlin 进行 Android 开发，并且也坚守了这一承诺。Kotlin 是一种富有表现力且简洁的编程语言，不仅可以减少常见代码错误，还可以轻松集成到现有应用中。如果想构建 Android 应用，Google建议从 Kotlin 开始着手，充分利用一流的 Kotlin 功能。为了支持使用 Kotlin 进行 Android 开发，Google和另一组织联手创办了 Kotlin 基金会，不断投入人力物力来提高编译器性能和 build 速度。Kotlin 中的协程提供了一种全新处理并发的方式，可以在 Android 平台上使用它来简化异步执行的代码。协程是从 Kotlin 1.3 版本开始引入，但这一概念在编程世界诞生的黎明之际就有了，最早使用协程的编程语言可以追溯到 1967 年的Simula语言。在 Android 平台上，协程主要用来解决两个问题:

1. 处理耗时任务 (Long running tasks)，这种任务常常会阻塞住主线程；
2. 保证主线程安全 (Main-safety) ，即确保安全地从主线程调用任何 suspend 函数。

### 1. 处理耗时任务

获取网页内容或与远程 API 交互都会涉及到发送网络请求，从数据库里获取数据或者从磁盘中读取图片资源涉及到文件的读取操作。这类操作被归类为耗时任务，此时应用会停下并等待它们处理完成，这会耗费大量时间。当今手机处理代码的速度要远快于处理网络请求的速度。以 Pixel 2 为例，单个 CPU 周期耗时低于 0.0000000004 秒，这个数字很难用人类语言来表述，然而，如果将网络请求以 “眨眼间” 来表述，大概是 400 毫秒 (0.4 秒)，则更容易理解 CPU 运行速度之快。仅仅是一眨眼的功夫内，或是一个速度比较慢的网络请求处理完的时间内，CPU 就已完成了超过 10 亿次的时钟周期了。

Android 中的每个应用都会运行一个主线程，它主要是用来处理 UI (比如进行界面的绘制) 和协调用户交互。如果主线程上需要处理的任务太多，应用运行会变慢，看上去就像是 “卡” 住了，这样是很影响用户体验的。所以想让应用运行上不 “卡”、做到动画能够流畅运行或者能够快速响应用户点击事件，就得让那些耗时的任务不阻塞主线程的运行。要做到处理网络请求不会阻塞主线程，一个常用的做法就是使用回调。回调就是在之后的某段时间去执行您的回调代码，使用这种方式，请求的网站数据的代码就会类似于下图26所示:

![网络请求代码图](/Users/charming/Library/Application Support/typora-user-images/image-20240223234217722.png)

在上图的示例中，即使 get 是在主线程中调用的，但是它会使用另外一个线程来执行网络请求。一旦网络请求返回结果，result 可用后，回调代码就会被主线程调用。这是一个处理耗时任务的好方法，类似于Retrofit这样的库就是采用这种方式帮您处理网络请求，并不会阻塞主线程的执行。使用协程可以简化代码来处理类似 fetchDocs 这样的耗时任务。用协程的方法来重写上面的代码，使用协程处理耗时任务，从而使代码更清晰简洁。如下图27所示：

![协程代码图](/Users/charming/Library/Application Support/typora-user-images/image-20240223234400981.png)

协程在常规函数的基础上新增了两项操作。在invoke(或 call) 和return之外，协程新增了suspend和resume:

- suspend — 也称挂起或暂停，用于暂停执行当前协程，并保存所有局部变量；
- resume — 用于让已暂停的协程从其暂停处继续执行。

Kotlin 通过新增 suspend 关键词来实现上面这些功能。只能够在 suspend 函数中调用另外的 suspend 函数，或者通过协程构造器 (如launch来启动新的协程。在上面的示例中，get 仍在主线程上运行，但它会在启动网络请求之前暂停协程。当网络请求完成时，get 会恢复已暂停的协程，而不是使用回调来通知主线程。Kotlin 使用堆栈帧来管理要运行哪个函数以及所有局部变量。暂停协程时，会复制并保存当前的堆栈帧以供稍后使用。恢复协程时，会将堆栈帧从其保存位置复制回来，然后函数再次开始运行。当主线程下所有的协程都被暂停，主线程处理屏幕绘制和点击事件时就会毫无压力。即使代码可能看起来像普通的顺序阻塞请求，协程也能确保网络请求避免阻塞主线程。

### 2. 使用协程保证主线程安全

在 Kotlin 的协程中，主线程调用编写良好的 suspend 函数通常是安全的。不管那些 suspend 函数是做什么的，它们都应该允许任何线程调用它们。

但是在 Android 应用中有很多的事情处理起来太慢，是不应该放在主线程上去做的，比如网络请求、解析 JSON 数据、从数据库中进行读写操作，甚至是遍历比较大的数组。这些会导致执行时间长从而让用户感觉很 “卡” 的操作都不应该放在主线程上执行。使用 suspend 并不意味着告诉 Kotlin 要在后台线程上执行一个函数，这里要强调的是，协程会在主线程上运行。事实上，当要响应一个 UI 事件从而启动一个协程时，使用 Dispatchers.Main.immediate 是一个非常好的选择，这样的话哪怕是最终没有执行需要保证主线程安全的耗时任务，也可以在下一帧中给用户提供可用的执行结果。协程会在主线程中运行，suspend 并不代表后台执行。如果需要处理一个函数，且这个函数在主线程上执行太耗时，但是又要保证这个函数是主线程安全的，那么可以让 Kotlin 协程在 Default 或 IO 调度器上执行工作。在 Kotlin 中，所有协程都必须在调度器中运行，即使它们是在主线程上运行也是如此。协程可以自行暂停，而调度器负责将其恢复。Kotlin 提供了三个调度器，可以使用它们来指定应在何处运行协程:

| 调度器   | Dispatchers.Main                                  | Dispatchers.IO                       | Dispatchers.Default                   |
| -------- | ------------------------------------------------- | ------------------------------------ | ------------------------------------- |
| 工作线程 | Android上的主线程，用来处理UI交互和一些轻量级任务 | 非主线程，专为磁盘和网络IO进行了优化 | 非主线程，专为CPU密集型任务进行了优化 |
| 主要使用 | 调用suspend函数，调用UI函数，更新LiveData         | 数据库，文件读写，网络读写           | 数组排序，JSON数据解析，处理差异判断  |

使用调度器来重新定义 get 函数。在 get 的主体内，调用 withContext 来创建一个在 IO 线程池中运行的块。您放在该块内的任何代码都始终通过 IO 调度器执行。由于 withContext 本身就是一个 suspend 函数，它会使用协程来保证主线程安全。借助协程，可以通过精细控制来调度线程。由于 withContext 可在不引入回调的情况下控制任何代码行的线程池，因此可以将其应用于非常小的函数，如从数据库中读取数据或执行网络请求。一种不错的做法是使用 withContext 来确保每个函数都是主线程安全的，这意味着，可以从主线程调用每个函数。这样，调用方就无需再考虑应该使用哪个线程来执行函数了。在这个示例中，fetchDocs 会在主线程中执行，不过，它可以安全地调用 get 来在后台执行网络请求。因为协程支持 **suspend** 和 **resume**，所以一旦 withContext 块完成后，主线程上的协程就会恢复继续执行。确保每个 suspend 函数都是主线程安全的是很有用的。如果某个任务是需要接触到磁盘、网络，甚至只是占用过多的 CPU，那应该使用 withContext 来确保可以安全地从主线程进行调用。这也是类似于 Retrofit 和 Room 这样的代码库所遵循的原则。同时，协程在这个原则下也可以被主线程自由调用，网络请求或数据库操作代码也变得非常简洁，还能确保用户在使用应用的过程中不会觉得 “卡”。