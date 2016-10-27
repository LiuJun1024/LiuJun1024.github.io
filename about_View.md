#About View
(View通过绘制自己与事件处理两种方式与用户交互)  

**some questions:**
>Google提出View这个概念的目的是什么？  
>View这个概念与Activtiy、Fragment以及Drawable之间是一种什么样的关系？  
>View能够感知Activity的生命周期事件吗？为什么？  
>View的生命周期是什么？    
>当View所在的Activity进入stop状态后，View去哪了？如果我在一个后台线程中持有一个View的引用，我此时能够改变它的状态吗？为什么？  
>当View所在的Activity进入stop状态后，View去哪了？如果我在一个后台线程中持有一个View的引用，我此时能够改变它的状态吗？为什么？  
>View能够与其他的View交叉重叠吗？重叠区域发生的点击事件交给谁去处理呢？可不可以重叠的两个View都处理？  
>View控制一个Drawable的方法途径有哪些？Drawable能不能与View通信？如果能如何通信？  
>假如View所在的ViewGroup中的子View减少了，View因此获得了更大的空间，View如何及时有效地利用这些空间，改变自己的绘制？  
>假如我要在View中动态地注册与解除广播接收器，应该在哪里完成呢？  
>假如我的手机带键盘（自带或者外接），你的自定义View应该如何响应键盘事件。  
>AnimationDrawable作为View的背景，会自动进行动画，View在其中扮演了怎样的角色？

##View的位置
Activity中通过setcontentview()将我们的布局加入到DecorView(DecorView是根布局，是一个FrameLayout)的一个FrameLayout中，
	
	这个FrameLayout的获取：ViewGroup content = (ViewGroup) findViewById(android.R.id.content);//得到DecorView的Content部分
	我们自己ViewGroup布局的获取：content.getChildAt(0);//得到我们设置的contentView
view的各种位置参数会封装在父view(ViewGroup)的内部类中，父view会把这些参数传给子view，调用inflate时，除了输入布局文件的id外，一般要求传入parent ViewGroup，传入这个参数的目的，就是为了读取布局文件中的layout配置信息，如果没有传入，这些信息将会丢失

##View的大小
我们会通过“layout_***”来指定view的大小，ViewGroup会得这些对子view大小的说明，结合自身考虑最终生成两个MeasureSpec对象（width与height）传给View，这两个对象是ViewGroup向子View提出的要求，就相当于告诉子View：“我已经与你的使用者（开发者）商量过了，现在把我们商量确定的结果告诉你，你的宽度不能违反width MeasureSpec对象的要求，你的高度不能违反height MeasureSpec对象的要求：

	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec)
开发者对两个MeasureSpec的解析：

	int widthMode = MeasureSpec.getMode(widthMeasureSpec);
	int widthSize =  MeasureSpec.getSize(widthMeasureSpec);
	int heightMode = MeasureSpec.getMode(heightMeasureSpec);
	int heightSize =  MeasureSpec.getSize(heightMeasureSpec);

关于Mode：

>1、EXACTLY 表示，ViewGroup对View说，你只能用100dp，原因是多样的，可能是你的使用者说要你完全占据我的空间，而我只有100dp。也可能这是你的使用者的要求，他需要你占这么大的空间，而我恰好也有这么多的空间，你的使用者让你占这么大的空间，肯定有他自己的考虑，你不能不理不顾，不然你达不到他的要求，他可能就不用你了。
>
2、AT_MOST表示，你最多只能用100dp，这是因为你的使用者说让你占据wrap_content的大小，让我跟你商量，我又不知道你到底要占多大区域，但是我告诉你，我只有100dp，你最多也只能用这么多哈。(这里，可以看出，当使用者在布局文件中要求一个View是wrap_content时，此时，View的大小决定权就交给View自己了，默认的View类中的实现，比较粗暴，就是将此时ViewGroup提供的空间全占据，完全没有真正根据自己的内容来确定大小，为什么这么粗暴？因为View是一个基类，所有的组件都是它的子类，每个子类的content都各不相同，View怎么可能知道content的大小呢，所以，它把wrap_content情况下，自己尺寸大小的决定权下放给了不同的子组件，让它们自己根据自己的内容去决定自己的大小，同样，我们自定义View时，也要考虑这一点)

>3、UNSPECIFIED  主要用于系统内部多次measure的情形。

View可以把自己想要的宽和高进行一个resolveSizeAndState处理，就可以达到上述目的。即如果想要的大小没超过要求，一切都Ok，如果超过了，在下面方法内部，就会把尺寸调整成符合ViewGroup要求的，但是也会在尺寸中设置一个标记，告诉ViewGroup，这个大小是子View委屈求全的结果。至于ViewGroup会不会理会这一标记，要看不同的ViewGroup了。如果你实现自己的ViewGroup，最好还是关注下这个标记

	resolveSizeAndState((int)(wantedWidth), widthMeasureSpec, 0),    
    resolveSizeAndState((int) (wantedHeight), heightMeasureSpec, 0);
最后重置一下大小

	setMeasuredDimension(
                resolveSizeAndState((int) (mDialWidth * scale), widthMeasureSpec, 0),
                resolveSizeAndState((int) (mDialHeight * scale), heightMeasureSpec, 0)
        );

resolveSizeAndState()方法也比较简单，如下：


	public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {  
      final int specMode = MeasureSpec.getMode(measureSpec);  
      final int specSize = MeasureSpec.getSize(measureSpec);  
      final int result;  
      switch (specMode) {     
         case MeasureSpec.AT_MOST:         
             if (specSize < size) {            
                  result = specSize | MEASURED_STATE_TOO_SMALL;         
             } else {            
                  result = size;      
             }         
             break;      
         case MeasureSpec.EXACTLY:          
              result = specSize;      
              break;       
         case MeasureSpec.UNSPECIFIED:   
         default:        
              result = size;   
       }   
       return result | (childMeasuredState & MEASURED_STATE_MASK);
	}

##View相关的一些方法

![](http://upload-images.jianshu.io/upload_images/1371984-368cf794d4fe5f7b.png?imageMogr2/auto-orient/strip%7CimageView2/2)    
  
1、View被inflated出来后，系统会回调该View的onFinishInflate方法，你的View可以在这个方法中，做一些准备工作。  
  
2、当veiw依附到window上时，调用onAttachedToWindow()方法，相对应地，view解除依附window时调用onDetachedFromWindow()方法  

3、如果你的View所属的Window可见性发生了变化，系统会回调该View的onWindowVisibilityChanged方法，你也可以根据需要，在该方法中完成一定的工作，比如，当Window显示时，注册一个监听器，根据监听到的广播事件改变自己的绘制，当Window不可见时，解除注册，因为此时改变自己的绘制已经没有意义了，自己也要跟着Window变成不可见了。  
  
4、当ViewGroup中的子View数量增加或者减少，导致ViewGroup给自己分配的屏幕区域大小发生变化时，系统会回调View的onSizeChanged方法，该方法中，View可以获取自己最新的尺寸，然后根据这个尺寸相应调整自己的绘制。

5、与用户交互的一些方法

##view的继承实现

view继承Object不必多言，除此之外，view还实现了三个接口：
  
Drawable.Callback
KeyEvent.Callback
AccessibilityEventSource
