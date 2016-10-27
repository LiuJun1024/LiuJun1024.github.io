# View的事件分发机制
  
##one.基本的流程(三个重要方法)：  
**1、public boolean dispatchTouchEvent(MotionEvent ev)**  
用来进行事件的分发，如果事件能够传给当前View，那么此方法一定会被调用，返回结果是由当前View的onTouchEvent和下级View的dispatchTouchEvent共同影响，表示是否消耗当前事件。  

**2、public boolean  onInterceptTouchEvent(MotionEvent ev)**   
这是ViewGroup的方法，用来判断是否拦截当前事件，view中没有此方法   
 
**3、public boolean onTouchEvent(MotionEvent ev)**  
是否消耗当前事件

###三个方法之间的联系(伪代码)：  
	public boolean dispatchTouchEvent(MotionEvent ev){

	boolean consume = false;

	if(onInterceptTouchEvent(ev)){
		consume = onTOuchEvent(ev);
	}else{
		consume = child.dispatchTouchEvent(ev);
	}
		return consume;
	}
>**对于根ViewGroup来说，点击事件产生后会先传递给它，这时它的dispatchTouchEvent会被调用。  
>1、如果这个ViewGroup的onInterceptTouchEvent方法返回true，就表示该ViewGroup要拦截当前事件，那么，该ViewGroup的onTouchEvent方法会被调用。  
>2、如果这个ViewGroup的onInterceptTouchEvent方法返回false，就表示该ViewGroup不拦截当前事件，那么，事件就会传递给它的子View，接着子View调用自己的disPatchTouchEvent
>**

*以上是view基本的事件分发，下面是其他一些需要注意的问题*

##two.
**1、当一个view设置了OnTouchListener，那么OnTouchListener的onTouch方法会被调用，  
如果此onTouch方法返回false，则当前view的onTouchEvent方法会被调用；  
如果此onTouch方法返回true，则当前view的onTouchEvent方法不会被调用；**
>OnTouchListener的优先级高于onTouchEvent方法，原因：看看View的disPatchTouchEvent方法:

    public boolean dispatchTouchEvent(MotionEvent event) {  
    if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&  
            mOnTouchListener.onTouch(this, event)) {  
        return true;  
    }  
    return onTouchEvent(event);  
	}  
>第二个条件(mViewFlags & ENABLED_MASK) == ENABLED是判断当前点击的控件是否是enable的，按钮默认都是enable的，因此这个条件恒定为true。

##
  

>OnClickListener的优先级最低，OnClickListener的优先级最低的原因：View的onTouchEvent方法的ACTION_UP的部分源码：

    switch (action) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true, x, y);
                       }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();//追踪这个方法！！！！！！！！！！！！！！！！！！！！！！！
                                }
                            }
                        }
					。。。。。
					。。。。。

    public boolean performClick() {
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);//！！！！！！！！！！！！！！！！这里调用了onClick
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        return result;
    }