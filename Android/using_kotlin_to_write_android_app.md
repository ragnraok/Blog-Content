Title: 使用Kotlin进行Android开发
Slug: using-kotlin-to-write-android-app
Date: 2015-06-24

### What is Kotlin
Kotlin，原意是在俄罗斯的一个[小岛](https://zh.wikipedia.org/wiki/%E7%A7%91%E7%89%B9%E6%9E%97%E5%B3%B6)，JetBrain在2011年推出了以这个来命名的一个运行在JVM上的语言， 看上去有点类似C#和Scala的结合，并且同为静态类型，作为一门JVM上的语言，可以轻松兼容Java，并且整个语言设计的非常轻量。目前的版本为``0.12.200``，尚未发布正式版。

----

Kotlin的下载和配置在其[官网](http://kotlinlang.org/docs/tutorials/getting-started.html)上有，在这里就不再赘述了，值得一提的是，作为JetBrains家出品的语言，自家的IDEA当然全力支持！

###基本语法介绍
Kotlin的语法非常简洁，熟悉Java或者Scala的人都可以快速上手：

#### 函数声明：
```
fun foo(va: Int): Int {
	return 1
}
```
也可以单行声明：
```
fun foo(va: Int): Int = 1
```
lambda当然也是支持的：
```
var c = {foo: Int -> println(foo)}
```
Kotlin中的函数是一等对象，自然支持高阶函数：
```
var c = {foo: Int -> println(foo)}
fun fooTest(func: (Int)->()) = println("I'm Groot")    
fooTest(c)
```
<br />
#### 类与接口
类可以这样进行声明：
```
class Bar(var b: Int): Foo() {
	var c = 1
	init {
		println("class initializer")
	}
	
	constructor(): this(1) {
        println("secondary constructor")
	}
}
```
Bar类在这里继承了Foo类，Bar类有两个构造函数，直接在Bar类头的是primary constructor，另外一个构造函数使用``constructor``关键字定义，注意必须要先调用primary constructor，另外，``init``标明的是class initializer，每个构造函数都会首先调用class initializer里面的代码，再调用构造函数

<br />
##### Inner class:
```
class Outer {
	class Inner {      
    }
}
```
Kotlin同样支持嵌套的内部类，不过和Java不一样的是，Kotlin的内部类不会默认包含一个指向外部类对象的引用，也就是说，Kotlin中所有的内部类默认就是**静态**的，这样可以减少很多内存泄露的问题。另外，如果需要在内部类中引用外部类对象，可以在Inner类的声明前加上``inner``关键字，然后在Inner类中使用标记的``this``：``this@Outer``来指向外部类对象

<br />
##### Singleton:
```
object Single {
    var c = 1
    
    fun foo() = println("foo")
}
```
Kotlin中使用``object``关键字声明一个singleton对象，后面这里的方法就可以直接使用``Single.foo()``来调用了

<br />
##### Interface:
```
interface Interface {
    fun foo() {
        println(1)
    }
    fun bar()
}
```
Kotlin中的interface，跟其他语言的``trait``非常像，而且也可以带有默认的实现方法，并且不允许通过属性来维护状态。事实上，在上个版本中，interface的原来名称是``trait``，而在M12现在这个版本中又改成了interface而已

<br />
#### Null safe and Smart type cast
##### Null safe:
在Kotlin中，严格区分了nullable和非nullable对象，甚至在编译期解决了不少潜在的空指针问题:

我们先来看下普通的变量声明
```
var c: String = "12123"
```
这里声明了一个String对象，其值为"12123"，我们可以正常的使用这个对象的成员方法：``c.length()``

但是，如果在初始化的时候，变量c为空的话，这样声明就是错误的，会编译不过：
```
var c: String = null
```
正确的声明应该是这样：
```
var c: String? = null
```
这里在``String``后面加多了一个问号，表明这里是一个**Nullable**的对象，说明这个变量在使用的过程中**可能为空**，而且，在调用这个变量的成员的时候，必须要使用这种语法：``c?.length()``，在调用的时候添加了一个问号，表明，如果``c``为空的时候，``length()``这个方法就不会调用。coffe-script也有类似的，这种语法糖减少了很多平时用到的Null-checked，简化了代码，而且从编译器开始介入null-checked，大大减少了潜在的``NullPointerException``，而事实上，null的确也是一个**[billion dollar mistake](http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare)**

常年进行如此的调用语法常常会很恼人，因此在你进行显式的Null-checked的时候，Kotlin的编译器会认为后续的调用已经无需进行Null-checked，可以直接调用了：
```
if (c != null) {
	c.length()
}
```

<br />
##### Smart type cast
在Kotlin中，进行强制类型转换可以使用``as``关键字，但有可能会抛出异常，因此，Kotlin引入了smart type cast:
```
if (c is String) {
	c.length()
}
```
在上面的例子中，如果``c``是一个String对象，则在if块中，可以直接使用String的方法，编译器会智能的帮你识别出c在if-blcok里面是一个String对象

<br />
#### Pattern Matching
Kotlin在一定程度上支持了一些FP的特性，包括强大的Pattern Matching，在Kotlin中可以使用``when``关键字：
```
var x = 3
when (x) {
    1 -> print("x == 1")
    2 -> print("x == 2")
    in 1..10 -> print("x is in the range")
    !in 10..20 -> print("x is outside the range")
    is Int -> println("is int")
    else -> { // Note the block
      print("x is neither 1 nor 2")
    }
}
```

<br />
#### Function Extension
在Java中我们经常需要给系统的类添加一些实用的方法，但苦于不能直接扩展，于是就有了各种的xxxUtils类，导致代码非常恶心，但是在Kotlin中，我们可以直接扩展库里面类的方法，通过function extension:
```
fun String.fucker() {
    println("a fucker")
}
```
上面给``String``类添加了一个fucker方法，我们可以直接使用：
```
"123123".fucker()
```
这大大的减少了我们写xxxUtils类的必要性

---
###配置使用Kotlin进行Android开发
使用Kotlin开发Android app的配置非常简单，按照[官方给出的配置即可](http://kotlinlang.org/docs/tutorials/kotlin-android.html)，直接在Gradle的配置文件build.gradle中添加一个依赖：
```
compile "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
```
然后添加Kotlin插件的使用：
```
apply plugin: 'kotlin-android'
``` 
进行一次Gradle Sync之后，就可以直接在项目使用Kotlin编写代码了，另外，如果安装了Intellij的Kotlin插件，可以选择
``Tools->Kotlin->Configure Kotlin in Project``，就可以自动进行上述的配置，一步到位

我写了一个简单的Demo app放到了Github上，有兴趣可以看下使用Kotlin开发android app具体是怎样的：[Demo地址](https://github.com/ragnraok/MovieCamera)

----
###对于dex方法数目的影响
dex有个65535方法数的限制，这对Android开发造成了很大的影响，在使用Kotlin进行android app开发的时候，需要将Kotlin的标准库打包进入apk中，这意味着如果标准库过大，对分包会造成很大的限制（因为这必须得打包在主dex中），所幸的是，Kotlin的哲学是“Java中有的，就尽量复用，不再自行创造一套”，使得整个Kotlin的标准库非常小，我们可以简单将Kotlin的标准库和其他比较常用库进行一下对比：

<table border="1" cellpadding="5">
	<tr>
		<td> 包名</td>
		<td>android-support-v13.jar</td>
		<td>android-support-v4.jar</td>
		<td>android-support-v7-appcompat.jar</td>
		<td>guava-18.0.jar</td>
		<td>scala-library-2.12.0-M1.jar</td>
		<td>kotlin-stdlib-0.12.213.jar</td>
	</tr>
	<tr>
		<td>方法数</td>
		<td>8219</td>
		<td>8118</td>
		<td>4624</td>
		<td>14833</td>
		<td>51248</td>
		<td>7228</td>
	</tr>
</table>

<br />
可以看出来Kotlin的标准库相当小，只有7000多个方法，比support-v13和support-v4还小，这体现了Kotlin的设计哲学之一："100% interoperable with Java"，基本上Java已经有的，Kotlin会尽量复用。而对比来看，同样是JVM上的语言，我们也可以选择使用Scala来进行Android开发，但Scala标准库有5万多个方法，全部打包进主dex中，很容易就导致app爆主dex了。所以综合来看，轻量形的Kotlin还是相当适合进行Android开发的。

----
### Project Anko
[Anko](https://github.com/JetBrains/anko) 是JetBrains推出的一个简化Android开发的库，同样由Kotlin来编写，主要的革命在于，声明UI的方式，完全抛弃了xml的使用，使用Anko，声明UI是这样做的：
```
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    val customStyle = { v: Any ->
        when (v) {
            is Button -> v.textSize = 26f
            is EditText -> v.textSize = 24f
        }
    }

    verticalLayout {
    
        padding = dip(34)
        imageView(android.R.drawable.ic_menu_manage).layoutParams {
            margin = dip(16)
            gravity = Gravity.CENTER
        }

         val name = editText {
             hintResource = R.string.name
        }
        val password = editText {
            hintResource = R.string.password
            inputType = TYPE_CLASS_TEXT or TYPE_TEXT_VARIATION_PASSWORD
        }

        button("Log in") {
            onClick {
                tryLogin(name.text, password.text)
            }
        }
  	}.style(customStyle)
}
```
你没看错，的确是在Activity类的onCreate方法中直接声明UI的布局。

Anko看起来像是使用了一种类似DSL的方式声明了界面的UI，这里主要是使用了Kotlin的其中两个特性：

- Function Extension，Anko扩展了Activity类，提供了额外的方法和属性
- Kotlin在调用函数的时候，如果最后一个参数为函数的话，则可以直接使用Lambda，并省略括号

因此这里声明布局的方式，其实全是Kotlin的原生代码，鹅妹子嘤！这样做有显然的好处：

- 由于实际上全是由代码来布局，省去了解析xml的时间
- xml本身有许多缺点，例如不可重用，非类型安全等，使用代码布局的话，我们可以很容易的就解决这个问题了

---
### Other References:
这里列出一些国外的关于Kotlin的介绍文章：

- [Using Project Kotlin for Android](https://docs.google.com/document/d/1ReS3ep-hjxWA8kZi0YqDbEhCqTt29hG8P44aA9W0DM8/edit?hl=zh-CN&forcehl=1) Square的JakeWharton曾经考察过几种不同的语言来进行Android开发，最后还是认为Kotlin比较优秀
- [Kotlin, the Swift of Android](http://blog.gouline.net/2014/08/31/kotlin-the-swift-of-android/) 这篇文章把Kotlin比喻为Android上的swift，虽然比较老，但是也可以看做Kotlin的介绍性文章