## Android Muitldex热更新修复方案原理

## 前言

    > 做程序开发，基础很重要。同样是拧螺丝人家拧出来的可以经久不坏，你拧出来的遇到点风浪就开始颤抖，可见基本功的重要性。再复杂的技术，也是由一个一个简单的逻辑构成。先了解核心基础，才能更好理解前沿高新技术。

## 正文大纲
 1.   先看效果{github Demo地址}：(https://github.com/18598925736/HotUpdateDemo)
 2.  Demo使用方法
 3.  Demo源码概览
 4.  热修复核心技术

##  基础知识预备
 
 about  hook思路

热更新技术，不是新话题。目前最热门的热更新由两种，一种是腾讯tinker为代表的 需重启app的热更新，一种是美团app为代表的instant Run，无需重启app. 今天先探究 前者的核心原理。

先看效果[github Demo地址] ：(https://github.com/18598925736/HotUpdateDemo)
假如说这是我们的app界面，这个界面有个bug，我们直接用一个 TextView来表示

![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216093644.png)
然而，我们的开发人员发现了这个bug，但是产品已经上线。这时候，由于引起bug的 代码，只有一行，


```java
public  class  MainActivity extends AppCompatActivity {
     
     @Override
     protected void onCreate(Bundle savedInstanceStata) {
           super.onCreate(savedINstanceState);
           srtContentView(R.layout.activity_main);
         
           TextView textView = findViewById(R.id.tv);
           Bug bug = new Bug():
           String s = bug.getstr():
           textView.setText(s):
     }
}
```
![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216093732.png)
这个时候，机智的程序员用最快的方式修复了这个bug，也只是改了一行代码：
![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216093752.png)
那么，产品已经在线上，怎么办？我们通过后台，向app推送了一个 fix.dex文件, 等这个文件下载完成，app提示用户，发现新的更新，需要重启app. 待用户重启，代码修复 即会生效。无需发布新版本！

![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216093824.png)

Demo使用方法

下载Demo代码之后，会在assets下看到一个fix.dex文件
![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216093843.png)
按照正常的逻辑，我们做bug修复一定是把fix.dex放到服务器上， app去服务器下载它，然后存放在app私有目录，重启app之后，fix.dex生效, 当加载到这个类的时候，就会去读fix.dex中当时打包的已修复bug的类. 但是，我这里为了演示方便，直接放在assets，然后使用 项目中的 AssetsFileUtil类 用io流将它读写到 app私有目录下.
演示方法：

删掉 fix.dex ，运行app，你看到 手机屏幕中心 出现："卧槽，有bug!"
还原 fix.dex ，运行app，你看到 手机屏幕中心 出现："嘿嘿，bug已修复"
起作用的是谁？就是这个fix.dex文件.

Demo源码概览

![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216093910.png)
如上图所示： 核心类其实就只有一个: ClassLoaderHookHelper ，它 就是 让 fix.dex这个补丁发挥作用的 " 幕后大佬". 这个核心类：有3个方法，分别是在不同的系统版本上，来对源码程序逻辑进行 hook,提高hook的兼容性.
![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216093932.png)

下面是完整 ClassLoaderHookHelper代码 以及 使用它的 MyApp完整代码 ：

```java
import java.io.File;
import java.io.IOException;
import java.lang.reflect.Array;
import java.lang.reflect.Field
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;
public class ClassLoaderHookHelper {
       
       //23和19的差别，就是 makeXXXElements 方法名和参数要求不同
      //后者是 makeDexElements(ArrayList<File> files, File optimizedDirectory,ArrayList<IOException> suppressedExceptions)
     //前者是 makePathElements(List<File> files, File optimizedDirectory,List<IOException> suppressedExceptions)
     public static void hookV23(ClassLoader classLoader,File outDexFilePath,File optimizedDirectory)throws IllegalAccessException, InvocationTargetException {
     Field pathList =ReflectionUtil.getField(classLoader,"pathList");//1、获DexPathList pathList 属性
     object dexpathListobj =pathList.get(classLoader);//2、获DexPathList pathList对象
     Field dexElementsField =ReflectionUtil.getField(dexPathListObj, "dexElements");//3、获得DexPathList的dexElements属性
     
     Object[] oldElements =(Object[]) dexElementsField.get(dexPathListObj);//4、获得pathList对象中 dexElements 的属性值
     ...

   }
}
```
Multidex热修复核心技术
>其实 热修复的核心技术，就一句话， HookClassLoader ,但是要深入了解它，需要相当多的基础知识，下面列举出必须要知道的一些东西。

### 基础知识预备
####1.Dex文件是什么？
我们写安卓，目前还是用 java比较多，就算是用 kotlin，它最终也是要转换成 java来运行。 java文件，被编译成 class之后，多个 class文件，会被打包成 classes.dex，被放到 apk中，安卓设备拿到 apk，去安装解析( 预编译balabala...)，当我们运行 app时， app的程序逻辑全都是在classes.dex中。所以， dex文件是什么？一句话， dex文件是 android app的源代码的最终打包

2.Dex文件如何生成？
androidStudio 打包 apk的时候会生成 Dex，其实它使用的是 SDK的 dx命令，我们可以用 dx命令自己去打包想要打包的 class. 命令格式为：dx --dex --output=output.dex xxxx.class 将上面的output 和 xxxx换成你想要的文件名即可。

注：dx.bat在 安卓 SDK的目录下：比如我d的`C:\XXXXX\AndroidStudioAbout\sdk1\build-tools\28.0.3\dx.bat

3.ClassLoader是什么？
ClassLoader来自 jdk，翻译为 ：类加载器，用于将 class文件中的类，加载到内存中，生成 class对象。只有存在了 Class对象，我们才可以创建我们想要的对象。 android SDK继承了JDK的 classLoader，创造出了新的 ClassLoader子类。下图表示了 android9.0-28 所有的ClassLoader直接或者间接子类.

![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216094040.png)

比较多的是 BaseDexClassLoader， DexClassLoader , PathClassLoader, 其他这些，应该是谷歌大佬 创造出来新的 类加载器子类吧，还没研究过。
注： 关于 DexClassLoader和 PathClassLoader ，网上资料有个误区，应该不少人都认为， PathClassLoader 用于加载 app内部的 dex文件， DexClassLoader用于加载外部的 dex文件，但是其实只要看一眼 这两个类的关系，就会发现，它们都是继承自 BaseDexClassLoader，他们的构造函数内部都会去执行父类的构造函数。他们只有一个差别，那就是 PathClssLoader不用传 optimizedDirectory这个参数，但是 DexClassLoader必须传。这个参数的作用是，传入一个 dex优化之后的存放目录。而事实上，虽然 PathClassLoader不要求传这个 optimizedDirectory，但是它实际上是给了一个默认值。emmmm............所以不要再认为 PathClassLoader不能加载外部的 dex了，它只是没有让你传 optimizedDirectory而已。

另外： BootClassLoader用于加载 AndroidFramework层class文件( SDK中没有这个BootClassLoader，也是很奇怪) PathClassLoader 是用于Android应用程序类的加载器，可以加载指定的 dex，以及 jar、 zip、 apk中的 classes.dex。 DexClassLoader 可以加载指定的 dex，以及 jar、 zip、 apk中的 classes.dex。

4.ClassLoader的双亲委托机制是什么？
android里面 ClassLoader的作用，是将 dex文件中的类，加载到内存中，生成 Class对象，供我们使用 （举个例子：我写了一个 A类，app运行起来之后，当我需要new 一个 A， ClassLoader首先会帮我查找 A的 Class对象是否存在，如果存在，就直接给我 Class对象，让我拿去 new A，如果不存在，就会出创建这个 A的 Class对象。） 这个查找的过程，就遵循 双亲委托机制。一句话解释 双亲委托机制：某个 类加载器在加载某个 类的时候，首先会将 这件事委托给 parent类加载器，依次递归，如果 parent类加载器可以完成加载，就会直接返回 Class对象。如果 parent找不到或者没有父了，就会 自己加载。

下图是 安卓源码 ClassLoader.java:

![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216094101.png)

红字注解，很容易读懂 ClassLoader去 load一个 class的过程.

### hook思路
OK，现在可以来解读我是如何去hook ClassLoader的了. 解读之前，先弄清楚，我为何 要 hookClassLoader，为什么 hook了它之后，我的 fix.dex就能发挥作用？先解决这个疑问，既然是 hook，那么自然要读懂源码，因为 hook就是在理解源码思维的前提下，更改源码逻辑。 一张图解决你的疑问：

![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216094126.png)


按照上面图，去追踪源码，会发现， ClassLoader最终会从 DexFile对象中去获得一个 Class对象。并且在 DexPathList类中 findClass的时候，存在一个 Element数组的遍历。这就意味着，如果存在多个 dex文件，多个 dex文件中都存在同样一个 class，那么它会从第一个开始找，如果找到了，就会立即返回。如果没找到，就往下一个dex去找。也就是说，如果我们可以在 这个数组中插入我们自己的修复bug的 fix.dex，那我们就可以让我们 已经修复bug的补丁类发挥作用,让类加载器优先读取我们的 补丁类.

OK,理解了源码的逻辑，那我们可以动手了。来解析SDK 23的 hookClassLoader过程吧！

确定思路，我们要改变app启动之后，自带的ClassLoader对象（具体实现类是PathClassLoader ）中 DexPathList 中 Element[] element 的实际值。
那么，步骤：

* 1.取得PathClassLoader的pathList的属性
* 2.取得PathClassLoader的pathList的属性真实值（得到一个DexPathList对象）
* 3.获得DexPathList中的dexElements 属性
* 4.获得DexPathList对象中dexElements 属性的真实值(它是一个Element数组) 做完这4个步骤，我们得到下面的代码

![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216094159.png)

5.用外部传入的Dex文件路径，构建一个我们自己的Element数组
![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216094220.png)

6.将从外部传入的ClassLoader中得到的Element数组和 我们自己的Element数组合并起来， 注意，我们自己的数组元素要放前面！
![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216094239.png)

7.将刚才合并的新Element数组，设置到 外部传入的ClassLoader里面去。
![](https://raw.githubusercontent.com/lixiaodaoaaa/publicSharePic/master/20191216094343.png)

上面的内容，读起来可能会有一些疑问，我预估到了一些，将答案写在下面

* 1. 当我们需要反射获得一个类的某个方法或者成员变量时，我们只想拿getDeclareXX，因为我们只想拿本类中的成员，但是仅仅getDeclareXX不能跨越继承关系 拿到 父类中的非私有成员，所以我写了ReflectionUtil.java，支持跨越继承关系 拿到父类的非私有成员。
* 2. 这种热修复，是不是下载的包会很大，和原先的apk差不多大？答案是，NO，我们只需要将我们修复bug之后的补丁dex下载到设备，让app重启，去读取这个dex即可。补丁包很小，甚至只有1K.
* 3. 这种修复方式必须重启么？ 是的，必须重启，当然，存在不需要重启就可以修复bug的方法，那种方法叫做instant run方案,本文不涉及。而，当前这种方案叫做：MultipleDex 即，多dex方案。
* 4.* 为什么要对SDK 23 ,19,14 写不同的hook代码？因为SDK版本的变迁，导致 一些类的关系，变量名，方法名，方法参数（个数和类型）都会发生变化，所以，要针对各个变迁的版本进行兼容。
