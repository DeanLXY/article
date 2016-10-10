##安卓App热补丁动态修复技术介绍
> hotFix    <https://github.com/dodola/HotFix>  
> hotfix 不在修复错误 ,出现了更好的 RecooFix <https://github.com/dodola/RocooFix>




## 背景 ##
当一个App发布之后，突然发现了一个严重bug需要进行紧急修复，这时候公司各方就会忙得焦头烂额：重新打包App、测试、向各个应用市场和渠道换包、提示用户升级、用户下载、覆盖安装。有时候仅仅是为了修改了一行代码，也要付出巨大的成本进行换包和重新发布。

这时候就提出一个问题：有没有办法以补丁的方式动态修复紧急Bug，不再需要重新发布App，不再需要用户重新下载，覆盖安装？

虽然Android系统并没有提供这个技术，但是很幸运的告诉大家，答案是：可以，我们QQ空间提出了热补丁动态修复技术来解决以上这些问题。

## 实际案例 ##
空间Android独立版5.2发布后，收到用户反馈，结合版无法跳转到独立版的访客界面，每天都较大的反馈。在以前只能紧急换包，重新发布。成本非常高，也影响用户的口碑。最终决定使用热补丁动态修复技术，向用户下发Patch，在用户无感知的情况下，修复了外网问题，取得非常好的效果。

## 解决方案 ##
该方案基于的是android dex分包方案的，关于dex分包方案，网上有几篇解释了，所以这里就不再赘述，具体可以看这里https://m.oschina.net/blog/308583。

简单的概括一下，就是把多个dex文件塞入到app的classloader之中，但是android dex拆包方案中的类是没有重复的，如果classes.dex和classes1.dex中有重复的类，当用到这个重复的类的时候，系统会选择哪个类进行加载呢？

> 让我们来看看类加载的代码：

![](http://mmbiz.qpic.cn/mmbiz/0aYRVN1mAJwR6vqR4Yv6V3zIvjqmgdu75fv55IpLkDqm2MibiaVQNFMQ3xFNk9ez1CXPibFPn0ibiboQf2kWPfSL8Mg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

一个ClassLoader可以包含多个dex文件，每个dex文件是一个Element，多个dex文件排列成一个有序的数组dexElements，当找类的时候，会按顺序遍历dex文件，然后从当前遍历的dex文件中找类，如果找类则返回，如果找不到从下一个dex文件继续查找。

理论上，如果在不同的dex中有相同的类存在，那么会优先选择排在前面的dex文件的类，如下图：

在此基础上，我们构想了热补丁的方案，把有问题的类打包到一个dex（patch.dex）中去，然后把这个dex插入到Elements的最前面，如下图：

好，该方案基于第二个拆分dex的方案，方案实现如果懂拆分dex的原理的话，大家应该很快就会实现该方案，如果没有拆分dex的项目的话，可以参考一下谷歌的multidex方案实现。然后在插入数组的时候，把补丁包插入到最前面去。

好，看似问题很简单，轻松的搞定了，让我们来试验一下，修改某个类，然后打包成dex，插入到classloader，当加载类的时候出现了（本例中是QzoneActivityManager要被替换）：

为什么会出现以上问题呢？
从log的意思上来讲，ModuleManager引用了QzoneActivityManager，但是发现这这两个类所在的dex不在一起，其中：
1. ModuleManager在classes.dex中
2. QzoneActivityManager在patch.dex中
结果发生了错误。
这里有个问题,拆分dex的很多类都不是在同一个dex内的,怎么没有问题?
让我们搜索一下抛出错误的代码所在，嘿咻嘿咻，找到了一下代码：

从代码上来看，如果两个相关联的类在不同的dex中就会报错，但是拆分dex没有报错这是为什么，原来这个校验的前提是：

如果引用者（也就是ModuleManager）这个类被打上了CLASS_ISPREVERIFIED标志，那么就会进行dex的校验。那么这个标志是什么时候被打上去的？让我们在继续搜索一下代码，嘿咻嘿咻~~，在DexPrepare.cpp找到了一下代码：

这段代码是dex转化成odex(dexopt)的代码中的一段，我们知道当一个apk在安装的时候，apk中的classes.dex会被虚拟机(dexopt)优化成odex文件，然后才会拿去执行。

虚拟机在启动的时候，会有许多的启动参数，其中一项就是verify选项，当verify选项被打开的时候，上面doVerify变量为true，那么就会执行dvmVerifyClass进行类的校验，如果dvmVerifyClass校验类成功，那么这个类会被打上CLASS_ISPREVERIFIED的标志，那么具体的校验过程是什么样子的呢？

此代码在DexVerify.cpp中，如下：

1. 验证clazz->directMethods方法，directMethods包含了以下方法：
    1. static方法
    2. private方法
    3. 构造函数
2. clazz->virtualMethods
    1. 虚函数=override方法?
概括一下就是如果以上方法中直接引用到的类（第一层级关系，不会进行递归搜索）和clazz都在同一个dex中的话，那么这个类就会被打上CLASS_ISPREVERIFIED：

所以为了实现补丁方案，所以必须从这些方法中入手，防止类被打上CLASS_ISPREVERIFIED标志。
最终空间的方案是往所有类的构造函数里面插入了一段代码，代码如下：

    if (ClassVerifier.PREVENT_VERIFY) {
   		 System.out.println(AntilazyLoad.class);
    }

其中AntilazyLoad类会被打包成单独的hack.dex，这样当安装apk的时候，classes.dex内的类都会引用一个在不相同dex中的AntilazyLoad类，这样就防止了类被打上CLASS_ISPREVERIFIED的标志了，只要没被打上这个标志的类都可以进行打补丁操作。

然后在应用启动的时候加载进来.AntilazyLoad类所在的dex包必须被先加载进来,不然AntilazyLoad类会被标记为不存在，即使后续加载了hack.dex包，那么他也是不存在的，这样屏幕就会出现茫茫多的类AntilazyLoad找不到的log。

所以Application作为应用的入口不能插入这段代码。（因为载入hack.dex的代码是在Application中onCreate中执行的，如果在Application的构造函数里面插入了这段代码，那么就是在hack.dex加载之前就使用该类，该类一次找不到，会被永远的打上找不到的标志)
其中:

之所以选择构造函数是因为他不增加方法数，一个类即使没有显式的构造函数，也会有一个隐式的默认构造函数。

空间使用的是在字节码插入代码,而不是源代码插入，使用的是javaassist库来进行字节码插入的。
隐患:

虚拟机在安装期间为类打上CLASS_ISPREVERIFIED标志是为了提高性能的，我们强制防止类被打上标志是否会影响性能？这里我们会做一下更加详细的性能测试．但是在大项目中拆分dex的问题已经比较严重，很多类都没有被打上这个标志。

如何打包补丁包：

１. 空间在正式版本发布的时候，会生成一份缓存文件，里面记录了所有class文件的md5，还有一份mapping混淆文件。

２. 在后续的版本中使用-applymapping选项，应用正式版本的mapping文件，然后计算编译完成后的class文件的md5和正式版本进行比较，把不相同的class文件打包成补丁包。

备注:该方案现在也应用到我们的编译过程当中,编译不需要重新打包dex,只需要把修改过的类的class文件打包成patch dex,然后放到sdcard下,那么就会让改变的代码生效。
