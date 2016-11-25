#Gradle For Android
参考资料：
##一、Groovy
>**gradle和maven等一样是一个构建工具，maven等一些构建工具是用xml配置的，而gradle采用groovy语言来书写自己的配置。groovy是基于JVM发的语言，groovy代码最终会被编译成.class文件被虚拟机执行，所以说，明白gradle就要对groovy有一定的了解**
  
###一些基本的语法：

----------


- Groovy注释标记和Java一样，支持//或者/**/

- Groovy语句可以不用分号结尾。Groovy为了尽量减少代码的输入，确实煞费苦心
	
- Groovy中支持动态类型，即定义变量的时候可以不指定其类型。Groovy中，变量定义可以使用关键字def。注意，虽然def不是必须的，但是为了代码清晰，建议还是使用def关键字
  
eg：  

	def variable1 = 1   //可以不使用分号结尾 
    def varable2 = "I am a person"
	def  int x = 1   //变量定义时，也可以直接指定类型
- 函数定义时，参数的类型也可以不指定。
  
eg：
  
	String testFunction(arg1,arg2){//无需指定参数类型
 	 ...
	}
- 在调用函数时，如果所定义的函数有参数，在使用的时候可以不使用括号，但是必须得传入参数，参数与函数名以空格隔开。
如果不出入参数必须添加括号，否则代码编译可以通过，但是运行时会出错，会将函数误认为是一个属性。
如果所定义的函数没有参数，在调用的时候必须添加括号，否则运行出错，会将函数误认为是一个属性。
函数调用的时候参数的个数必须匹配，否则也会报错，提示没有定义该函数，如果单个参数除外，可以不用输入参数，系统默认赋值null。

eg:
	
	def printSomething(param01,param02){
    println param01+param02
	}
 
	printSomething ("hello","world")//helloworld
	printSomething "hello","world"//helloworld
 
	//参数个数不对，报错
	//printSomething ("hello")
 
	def printOne(param){
    println param
	}
	printOne()//null

  
- 除了变量定义可以不指定类型外，Groovy中函数的返回值也可以是无类型的。

eg:


	def  nonReturnTypeFunc(){
     last_line   //最后一行代码的执行结果就是本函数的返回值
	}

	//如果指定了函数返回类型，则可不必加def关键字来定义函数
	String  getString(){
   		return "I am a string"
	}


- 函数可以有返回值，如果有显示地使用return关键字，则返回return指定的返回值，其后面的语句不再执行。如果没有显式地使用return关键字，则返回函数最后一行语句的运行结果。如果使用void关键字代替def关键字定义函数，则函数的返回值将为null。
  
eg:

	defprintSomething(){
    	return "hello"
    	println "world"
	}
	println printSomething()//hello
 
	def printSomething01(){
    	"hello world"
    	1
	}
	println printSomething01()//1
 
	def printSomething02(){
    	1
    	"hello world"
	}

	println printSomething02()//hello world	


- 函数内不可以访问函数外的变量(但是，下面的代码中，如果m、n前面没有修饰z则方法中可以访问。)

eg:

	int m=1
	def n="hello"
	//error
	defprintSomething(){
   	 println m//error
   	 println n//error
	}
	printSomething()

- 函数可以赋值给其它函数，使用语法标记&将函数赋予新的函数。

eg:

	def printSomething() {
    	println("hello world")
	}
 
	//printSomething不可以加括号
	def printHello=this.&printSomething
 
	printHello()//hello world
	printSomething()//hello world


>**以上所述的知识groovy语法中一部分，接下来，说说对于我们熟悉gradle时比较重要的一个groovy语法：闭包**


###闭包

---------- 
**Groovy中闭包是这么定义的:可以用作函数参数和方法参数的代码块。可以把这个代码块理解为一个函数指针。**  
####闭包的定义格式：

	def xxx = { params -> code }
	//或者
	def xxx={code}

####闭包的特点：

闭包可以访问外部的变量：  

	def str="hello world"
 
	def closure={
    	println str
	}
 
	closure()//hello world
闭包调用的方式有两种，闭包.call(参数)或者闭包(参数),在调用的时候可以省略圆括号：  
	
	def closure = {
    	param -> println param
	}
 
	closure("hello world")
	closure.call("hell call")
	closure "hello world"

闭包是有返回值的，默认最后一行语句就是该闭包的返回值，如果最后一行语句没有返回任何类型，闭包将返回null。

	def closure = {
    	println "hello world"
    	return "I'm callback"
	}
	//hello world
	//I'm callback
	println closure()

闭包可以有参数，如果没有定义参数，会有一个隐式的默认参数it，如果没有参数可以将[参数]和[->]省略。

如果存在参数，在[->]之前的就是参数，如果只有一个参数，参数可以省略。

闭包中参数名称不能与闭包内或闭包外的参数名重名：

	def closure={
    	println "hello $it"
	}
	closure("admin")
 
	def closure={
   		param01,param02,param03->printlnparam01+param02+param03
	}
	closure "hello","world","ok"
 
	def closure={
    	printlnit
	}
	closure "hello world"
 
	//编译时就不通过
	//def param
	//def closure={
	//    param->println param
	//}

闭包可以作为一个参数传递给另一个闭包，也可以在闭包中返回一个闭包：

	def toTriple = { n -> n * 3 }
	def runTwice = { a, c -> c(c(a)) }
	println runTwice(5, toTriple)//45
 
	def times = { x -> { y -> x * y } }
	println times(3)(4)//12

闭包的一些快捷写法，当闭包作为闭包或方法的最后一个参数，可以将闭包从参数圆括号中提取出来接在最后。

如果闭包中不包含闭包，则闭包或方法参数所在的圆括号也可以省略。

对于有多个闭包参数的，只要是在参数声明最后的，均可以按上述方式省略。

	def runTwice = { a, c -> c(c(a)) }
	println runTwice(5, { it * 3 }) //45 usual syntax
	println runTwice(5) { it * 3 } //45
 
	def closure = {
    	param -> println param
	}
	closure "hello world"
 
	def runTwoClosures = { a, c1, c2 -> c1(c2(a)) }
	//when more than one closure as last params
	assert runTwoClosures(5, { it * 3 }, { it * 4 }) == 60 //usual syntax
	assert runTwoClosures(5) { it * 3 } { it * 4 } == 60 //shortcut form

闭包接受参数的规则，会将参数列表中所有有键值关系的参数，作为一个map组装，传入闭包作为调用闭包的第一个参数。

	def f= {m, i, j-> i + j + m.x + m.y }
	println f(6, x:4, y:3, 7)//20

如果闭包的参数声明中没有list，那么传入参数可以设置为list，里面的参数将分别传入闭包参数。

	def c = { a, b, c -> a + b + c }
	def list = [1, 2, 3]
	println c(list) // 6

>**对闭包有一些了解后，我们可以了解一下Gradle了**

----------

 
##**Gradle基本介绍**：  
  
**Gradle中，每一个待编译的工程(application 或 library)都叫一个Project。** 
 
**每一个Project在构建的时候都包含一系列的Task。比如一个Android APK的编译可能包含：Java源码编译Task、资源编译Task、JNI编译Task、lint检查Task、打包生成APK的Task、签名Task等。**  
  
**一个Project到底包含多少个Task，其实是由编译脚本指定的插件决定。插件是什么呢？插件就是用来定义Task，并具体执行这些Task的东西。**  
  
**刚才说了，Gradle是一个框架，作为框架，它负责定义流程和规则。而具体的编译工作则是通过插件的方式来完成的。比如编译Java的project有Java插件，编译Groovy的project有Groovy插件，编译Android APP有Android APP插件，编译Android Library有Android Library插件**  

**so far，你知道Gradle中每一个待编译的工程都是一个Project，而不同的类型的project会依赖相应的插件，一个具体的编译过程是由相应的插件的一个一个的Task来定义和执行的。**

----------
 
Gradle主要有三种对象，这三种对象和三种不同的脚本文件对应，在gradle执行的时候，会将脚本转换成对应的对端：   
 
1. Gradle对象：当我们执行gradle xxx或者什么的时候，gradle会从默认的配置脚本中构造出一个Gradle对象。在整个执行过程中，只有这么一个对象。Gradle对象的数据类型就是Gradle。我们一般很少去定制这个默认的配置脚本。  


2. Project对象：每一个build.gradle会转换成一个Project对象。  


3. Settings对象：显然，每一个settings.gradle都会转换成一个Settings对象。  

##gradle工作时的流程：
![](http://img.blog.csdn.net/20150905194317170?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)	  

首先是初始化阶段。对我们前面的multi-project build而言，就是执行settings.gradle  
Initiliazation phase的下一个阶段是Configration阶段。  
  
Configration阶段的目标是解析每个project中的build.gradle。比如multi-project build例子中，解析每个子目录中的build.gradle。在这两个阶段之间，我们可以加一些定制化的Hook。这当然是通过API来添加的   
 
Configuration阶段完了后，整个build的project以及内部的Task关系就确定了。一个Project包含很多Task，每个Task之间有依赖关系。Configuration会建立一个有向图来描述Task之间的依赖关系。所以，我们可以添加一个HOOK，即当Task关系图建立好后，执行一些操作。
  
  
最后一个阶段就是执行任务了。当然，任务执行完后，我们还可以加Hook。
##gradle命令介绍(win下)
gradlew properites用来查看所有属性信息  
gradlew clean是执行清理任务  
gradlew projects  查看本工程下包含多少个project(gradle里的project，不要和Android的project搞混)  
gradlew tasks 查看本工程下包含多少个tasks  
gradlew ：projectName：tasks 相应project下包含多少个tasks  
gradlew ：projectName：taskName 执行相应proj下的某个task**(task之间会有依赖关系，若A依赖于B，那么执行A的时候回先执行B)**


##project对象
项目中每一个module下的build.gradle文件都会转换成一个Project对象。在Gradle术语中，Project对象对应的是BuildScript。   
Project包含若干Tasks。另外，由于Project对应具体的工程，所以需要为Project加载所需要的插件，比如为Java工程加载Java插件。其实，一个Project包含多少Task往往是插件决定的。
所以，在Project中，我们要：

- 加载插件
- 根据加载插件的不同进行不同的配置
- 设置属性  

###加载插件(jar包)：

	//添加Gradle的Android构建插件，默认包含一些task和android {...}等特定元素
	apply plugin: 'com.android.application'//apply是一个函数，apply其实是Project实现的PluginAware接口定义的，plugin: 'com.android.application'是一个map类型的参数

>apply函数还可以加载gradle文件，比如有一个可以通用的函数，我们就可以把这种通用的函数放到一个utils.gradle文件中，然后用apply加载这个文件

	apply from: rootProject.getRootDir().getAbsolutePath() + "/utils.gradle" 

###设置属性： 

如果是单个脚本，则不需要考虑属性的跨脚本传播，但是Gradle往往包含不止一个build.gradle文件，比如我设置的utils.gradle，settings.gradle。如何在多个脚本中设置属性呢？
Gradle提供了一种名为extra property的方法。extra property是额外属性的意思，在第一次定义该属性的时候需要通过ext前缀来标示它是一个额外的属性。定义好之后，后面的存取就不需要ext前缀了。ext属性支持Project和Gradle对象。即Project和Gradle对象都可以设置ext属性  

举个例子：我在settings.gradle中想为Gradle对象设置一些外置属性，所以initMinshengGradleEnvironment函数中

	def initMinshengGradleEnvironment(){  
    //属性值从local.properites中读取  
    Propertiesproperties = new Properties()  
    FilepropertyFile = new File(rootDir.getAbsolutePath() +"/local.properties")//也体现了local.properties文件的作用  
   	properties.load(propertyFile.newDataInputStream())  
    //gradle就是gradle对象。它默认是Settings和Project的成员变量。可直接获取  
  	 //ext前缀，表明操作的是外置属性。api是一个新的属性名。前面说过，只在  
   	//第一次定义或者设置它的时候需要ext前缀  
    gradle.ext.api = properties.getProperty('sdk.api')  
     
    println gradle.api  //再次存取api的时候，就不需要ext前缀了  
    ......
	}

我在utils.gradle中定义了一些函数，然后想在其他build.gradle中调用这些函数。那该怎么做呢？  

	//第一个函数：utils.gradle中定义了一个获取AndroidManifests.xmlversionName的函数  
	def  getVersionNameAdvanced(){  
 		//下面这行代码中的project是谁？  
   		defxmlFile = project.file("AndroidManifest.xml")  
  		defrootManifest = new XmlSlurper().parse(xmlFile)  
   		returnrootManifest['@android:versionName']    
	}  

	//第二个函数：对于android library编译，我会disable所有的debug编译任务  
	def disableDebugBuild(){  
  		//project.tasks包含了所有的tasks，下面的findAll是寻找那些名字中带debug的Task。  
  		//返回值保存到targetTasks容器中  
  		def targetTasks = project.tasks.findAll{task ->  
     		task.name.contains("Debug")  
  		}  
 	 	//对满足条件的task，设置它为disable。如此这般，这个Task就不会被执行  
 		targetTasks.each{  
     		println"disable debug task  :${it.name}"  
    		it.setEnabled false  
  		}  
	}  
	//现在，想把这个API输出到各个Project。由于这个utils.gradle会被每一个Project Apply，所以  
	//我可以把getVersionNameAdvanced定义成一个closure，然后赋值到一个外部属性  
 	//下面的ext是谁的ext？  
	ext{ //此段花括号中代码是闭包  
    	//除了ext.xxx=value这种定义方法外，还可以使用ext{}这种书写方法。  
    	//ext{}不是ext(Closure)对应的函数调用。但是ext{}中的{}确实是闭包。  
    	getVersionNameAdvanced = this.&getVersionNameAdvanced  
		disableDebugBuild = this.&disableDebugBuild
 	}

在library的build.gradle中调用utils.gradle中自定义的函数：
 
	。。。。
	/* 
  	如果不需要编译debug版的东西 
  	当Project创建完所有任务的有向图后，我通过afterEvaluate函数设置一个回调Closure。在这个回调 
  	Closure里，我disable了所有Debug的Task 
	*/  
	project.afterEvaluate{  
    	disableDebugBuild()  
	}

	。。。。

上面代码中有两个问题：

1. project是谁？
2. ext是谁的ext？

加载utils.gradle的Project对象和utils.gradle本身所代表的Script对象到底有什么关系？   


1. 问题1：project就是加载utils.gradle的project。由于posdevice有5个project，所以utils.gradle会分别加载到5个project中。所以，getVersionNameAdvanced才不用区分到底是哪个project。反正一个project有一个utils.gradle对应的Script。
2. 问题2：ext：自然就是Project对应的ext了。此处为Project添加了一些closure。那么，在Project中就可以调用getVersionNameAdvanced函数了  

###根据加载插件的不同进行不同的配置(主要是Android中的application插件和library插件)：

1、**multi-project根目录下的build.gradle文件**  
这个文件用来做一些全局配置(关于这些函数或者说script block，可以在[https://docs.gradle.org/current/javadoc/](https://docs.gradle.org/current/javadoc/ "gradle线上文档")查询)：
  
	//下面这个subprojects{}就是一个Script Block  
	subprojects {  
  		println"Configure for $project.name" //遍历子Project，project变量对应每个子Project  
  		buildscript {  //这也是一个SB  
    		repositories {//repositories是一个SB  
       		///jcenter是一个函数，表示编译过程中依赖的库，所需的插件可以在jcenter仓库中  
       		//下载。  
       		jcenter()  
    		}  
    		dependencies { //SB  
        		//dependencies表示我们编译的时候，依赖android开发的gradle插件。插件对应的  
       			//class path是com.android.tools.build。版本是1.2.3  
        		classpath'com.android.tools.build:gradle:1.2.3'  
    		}  
   			//为每个子Project加载utils.gradle 。当然，这句话可以放到buildscript花括号之后  
   			applyfrom: rootProject.getRootDir().getAbsolutePath() + "/utils.gradle"  
 		}//buildscript结束  
	}

	allprojects {
		//全局编码设置
    	tasks.withType(JavaCompile){
        	options.encoding = "UTF-8"
    	}
	}

>**关于密码和签名等信息的全局配置，可以在gradle.properties中配置**  
>格式：
key value  
例子：

	STORE_FILE_PATH ../test_key.jks  
	STORE_PASSWORD test123
	KEY_ALIAS kale
	KEY_PASSWORD test123
	PACKAGE_NAME_SUFFIX .test
	TENCENT_AUTHID tencent123456
配置后，你就可以在build.gradle中随意使用了。


	signingConfigs {
    	release {
        	storeFile file(STORE_FILE_PATH)
        	storePassword STORE_PASSWORD
        	keyAlias KEY_ALIAS
        	keyPassword KEY_PASSWORD
    	}
	}