# **Activity + 多Fragment的一些坑** 

　　目前单个（多个）Activity + 多个Fragment已经成为主流APP的页面呈现方式，例如：微信、今日头条等。下面就阐述一下Activity + 多个Fragment的使用过程中的一些问题和经验教训。**需要注意的是：本文说讲的Fragment都是support.v4下的Fragment，具体的 support-v4-23.1.1**。

<font size = 5>**1 。 getActivity() = null**</font>  

　　当你用Fragment的时候，经常会碰到这个问题，容器对象为空。  
　　**原因：**  
　　当你将你的Fragment出栈之后，该Fragment仍然在异步执行任务，并且执行完之后调用了getActivity()方法。  
　　**解决方法：**  
　　晓得原因，解决办法很简单了。Fragment出栈之后，不让你的Fragment执行异步任务，或者已补任务执行后，不调用getActivity方法就是了。但是这个说起来容易，当你的团队人多，能力参差不齐的时候，往往会出现问题。

<font size = 5>**2 。 Fragment接口的选择：replace or show hide**</font> 
  
　　show(), hide()最终是让你的Fragment的View调用setVisibility()，不会调用生命周期。  
　　replace()会销毁视图，调用onDestroyView、onCreateView等一系列的生命周期。  
　　如果你业务有一个很高的概率经常使用当前的Fragment，建议使用hide和show，他不会调用Fragment 的onStop方法，比如微信底部的tab场景等。
 
<font size = 5>**3 。 Fragment导致的界面重叠**</font>  

　　若你使用add(), show(), hide()方法进行Fragment的控制，当你将程序放在后台，多一段时间，会发现出现了问题：你的几个Fragment界面发生了重叠。  
　　**原因：**　  
　　系统在页面重启前，帮我们保存了Fragment的状态，但是在重启后恢复时，视图的可见状态没帮我们保存，而Fragment默认的是show状态。这样就导致了再次启动进入系统会从栈底向栈顶的顺序恢复Fragments，并且每一个恢复的Fragment都是以show()的方式显示出来，从而导致界面的重叠。  

　　**解决办法：**  
　　FragmentManager在任何情况都会帮你存储Fragment，你要做的仅仅是在“内存重启”后，找回这些Fragment即可。  

　　**1）通过findFragmentByTag**   
　　在add Fragment的时候，绑定一个tag（一般都是使用Fragment的类名），发生内存重启的时候，通过findFragmentByTag找到对应的Fragment，并hide掉需要隐藏的Fragment(可以在Activity的onSaveInstance中保存需要显示的Fragment的tag值)。  

```
@Overrideprotected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity);

    ShowFragment showFragment;
    HideFragment hideFragment;

    if (savedInstanceState != null) {
        showFragment = getSupportFragmentManager().findFragmentByTag(showFragment.getClass().getName);
        hideFragment = getSupportFragmentManager().findFragmentByTag(hideFragment.getClass().getName);
        getFragmentManager().beginTransaction()
                .show(showFragment)
                .hide(hideFragment)
                .commit();
    }else{  // 正常时
        showFragment = ShowFragment.newInstance();
        hideFragment = HideFragment.newInstance();

        getFragmentManager().beginTransaction()
                .add(R.id.container, showFragment, showFragment.getClass().getName())
                .add(R.id,container,hideFragment,hideFragment.getClass().getName())
                .hide(hideFragment)
                .commit();
    }
}
```  
　　**2）通过Fragment自身的saveInstanceState**  
　　Fragment像Activity一样，可以通过onSaveInstanceState来保存序列化的数据，以防止程序突然丢失数据（内存重启）。具体的通过onSaveInstanceState来实现防止Fragment重叠的方法如下所示： 

```  
 @Override public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        if (savedInstanceState != null) {
            mIsHidden = savedInstanceState.getBoolean(FragmentOpe.FRAGMENT_SAVE_STATE_HIDDEN);
            processRestoreState();
        }
    }

 @Override public void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        outState.putBoolean(FragmentOpe.FRAGMENT_SAVE_STATE_HIDDEN, isHidden());
    }


    private void processRestoreState() {
        FragmentTransaction ft = getFragmentManager().beginTransaction();
        if (mIsHidden) {
            ft.hide(this);
        }
        else {
            ft.show(this);
        }
        ft.commit();
    }
```

　　**方式1和的2区别是：由Activity/父Fragment来管理子Fragment的显示/隐藏状态转变为由Fragment自己来管理自己的显示/隐藏状态。**  

   
<font size = 5>**4 。 Frament的出栈remove的使用问题**</font>  
  
　　FragmentManager提供了remove方法用于Fragment的出栈，但是这个方法是有问题的。    
　　如下面的代码所示：我添加了两个Fragment到Activity容器中，当用户点击back按键的时候，我通过如下方法进行出栈操作，你会发现：getBackStackEntryCount()方法永远都是返回2个。但是通过日志你客可看到你的两个Fragment的onDetach方法都被调用了。

```
public void onBackPressed() {
        int count = mFragmentManager.getBackStackEntryCount();
        Log.d(tag, "fragment count = " + count);
        if (count > 0) {
            String tag = mFragmentManager.getBackStackEntryAt(count - 1).getName();
            Fragment fragment = mFragmentManager.findFragmentByTag(tag);
            FragmentTransaction ft = mFragmentManager.beginTransaction().remove(fragment);
            ft.commit();
        }
        else {
            super.onBackPressed();
        }
    }
```

　　如上所示，正确的使用方法是调用popBackStack系列函数。remove函数仅仅是能让Fragment的视图从容器内移除。正确的回退实现：  

```
public void onBackPressed() {
        int count = mFragmentManager.getBackStackEntryCount();
        Log.d(tag, "fragment count = " + count);
        if (count > 0) {
            mFragmentManager.popBackStackImmediate();
        }
        else {
            super.onBackPressed();
        }
    }
```  
  
　　调用上面的方法进行出栈操作，你会发现：**getBackStackEntryCount()得到了正确的值。此处还需要注意的是：如果你调用FragmentMananger的getFragments()方法返回的List<Fragment>，你会发现他的size
不会变化。但是，List中的Fragment缺随着后退变为null**。  

　　如果你觉着popBackStack系列函数是木有问题的， 那么你就错了。  

<font size = 5>**5。 Fragment的popBackStack系列接口的使用问题**</font>   
 
　　popBackStack()  
　　popBackStackImmediate()  
　　popBackStack(String tag,int flags)  
　　popBackStack(int id,int flags)  
　　popBackStackImmediate(String tag,int flags)  
　　popBackStackImmediate(int id,int flags)  

　　Android总共提供了6个popBackStack()系列函数，区别如下：  
　　a、前两个函数用于单个Fragment的出栈，后面四个用户多个Fragments的出栈；  
　　b、popBackStack和popBackStackImmediate系列函数的区别在于：前者是加入到主线队列的末尾，等其它任务完成后才开始出栈，后者是立刻出栈。  

 　　**1）单个Fragment出栈时事物和动画的问题**    
　　单个Fragment出栈，调用popBackStack()或者popBackStackImmediate()接口，但是如果你在调用这两个接口之后紧接着调用类似如下的事物方法，**此时如果出栈动画还没有完成**，会导致内存重启后按返回键报错的问题：


```
getSupportFragmentManager().popBackStackImmdiate();
getSupportFragmentManager().beginTransaction()
        .add(R.id.container, fragment , tag)
        .hide(currentFragment)
        .commit;
```

　　正确的使用方法如下所示，使用主线程的Handler：

```
getSupportFragmentManager().popBackStackImmdiate();
new Handler().post(new Runnable(){
          @Override
           public void run() {
                // 在这里执行Fragment事务
           }
});
```
　　**需要注意的是：假如你的Fragment没有设置任何的动画，那么上面的问题是不存在的。**  

　　**2）多个Fragment出栈的问题**  
　　在进行多个Fragment的出栈接口中，popBackStack是有问题的，不建议使用，**推荐使用popBackStackImmediate系列接口**。  
　　当你调用popBackStack系列的接口实现多个Fragment出栈时，进行多个出栈和入栈操作后，会导致栈内的Fragment顺序不正确的问题。stackoverflow上的大神给出了：[解决方案](http://stackoverflow.com/questions/25520705/android-cant-retain-fragments-that-are-nested-in-other-fragments)，具体的如下所示：  

```
public class FragmentTransactionBugFixHack {

  public static void reorderIndices(FragmentManager fragmentManager) {
    if (!(fragmentManager instanceof FragmentManagerImpl))
      return;
    FragmentManagerImpl fragmentManagerImpl = (FragmentManagerImpl) fragmentManager;
    if (fragmentManagerImpl.mAvailIndices != null && fragmentManagerImpl.mAvailIndices.size() > 1) {
      Collections.sort(fragmentManagerImpl.mAvailIndices, Collections.reverseOrder());
    }
  }
}
```

　　怎么使用呢？  
　　在你调用popBackStackImmediate出栈多个Fragment之后，调用reorderIndices方法就可以。具体的使用方法如下所示，必须在主线程总调用：  

```
hanler.post(new Runnable(){
    @Override
     public void run() {
         FragmentTransactionBugFixHack.reorderIndices(fragmentManager));
     }
});
```
　　为了防止出栈动画带来的问题，在调用reorderIndices接口之前，最好获取一下动画动画的时间，调动postDelay来进行reorderIndices操作。  


<font size = 5>**6 。 Fragment转场动画的问题**</font>     
　　Activity的转场动画通过overridePendingTransition(int enterAnim, int exitAnim)就可以实现。  
　　Fragment的动画分为出栈动画和入栈动画。FragmentManager提供了setCustomAnimations(enter, exit)接口来设置入栈动画：第一个参数，需要注意的是：第二个参数并不是出栈动画。当你设置了出栈动画，好像根本就不起作用。   
 
　　那么怎么实现Fragment的入栈、出栈动画呢？  
　　通过Fragment的setTransition接口，并且继承实现Fragment的onCreateAnimation()接口，具体如下所示：  

　　**Ativity中启动FragmentA：**  

```
 FragmentTransaction ft = mFragmentManager.beginTransaction();
                ft.setTransition(FragmentTransaction.TRANSIT_FRAGMENT_OPEN);
                ft.add(R.id.fl_container, FragmentDemo.newInstance(), FragmentDemo.class.getName());
                ft.addToBackStack(FragmentDemo.class.getName());
                ft.commit();
```

　　**FragmentA启动FragmentB：** 

```
 FragmentManager fragmentManager = getFragmentManager();
                FragmentTransaction ft = fragmentManager.beginTransaction();
                ft.setTransition(FragmentTransaction.TRANSIT_FRAGMENT_OPEN);
                ft.add(R.id.fl_container, FragmentDemo2.newInstance(),
                        FragmentDemo2.class.getName());
                ft.hide(FragmentDemo.this);
                ft.addToBackStack(FragmentDemo2.class.getName());
                ft.commit();
```

　　**Fragment继承实现onCreateAnimation接口：**  

```
 @Override public Animation onCreateAnimation(int transit, boolean enter, int nextAnim) {
        Log.d(tag, "onCreateAnimation transit = " + transit + "; enter = " + enter);
        if (transit == FragmentTransaction.TRANSIT_FRAGMENT_OPEN) {
            if (enter) {
                return mEnterAnim;
            }
            else {
                return mPopExitAnim;
            }
        }
        else if (transit == FragmentTransaction.TRANSIT_FRAGMENT_CLOSE) {
            if (enter) {
                return mPopEnterAnim;
            }
            else {
                return mExitAnim;
            }
        }

        return super.onCreateAnimation(transit, enter, nextAnim);
    }
```

　　**Activity的onBackPressed:**  

```
 int count = mFragmentManager.getBackStackEntryCount();
        Log.d(tag, "fragment count = " + count);
        if (count > 0) {
            mFragmentManager.popBackStackImmediate();
        }
        else {
            super.onBackPressed();
        }
```

　　上述代码的使用场景：Activity启动FragmentA,FragmentA启动Fragment
B，然后按返回键知道Activity销毁。  onCreateAnimation日志信息如下：  
　　**FragmentA启动时：  
　　onCreateAnimation transit = 4097; enter = true //A执行mEnterAnim动画  
　　FragmentA启动FragmentB时，Fragment hihe:  
　　onCreateAnimation transit = 4097; enter = false//A执行PopExitAnim动画  
　　onCreateAnimation transit = 4097; enter = true//B执启动mEnterAnim动画  
　　按返回键第一次：  
　　onCreateAnimation transit = 8194; enter = true//A执行mPopEnterAnim动画  
　　onCreateAnimation transit = 8194; enter = false//B执行mExitAnim动画  
　　按返回键第二次：  
　　onCreateAnimation transit = 8194; enter = false//A执行mExitAnim动画**  

　　通过上面日志的打印信息，我们就可以知道应该怎样使用Fragment的转场动画。  
  

<font size = 5>**7 。 Fragment与Fragment/Activity之间的数据传输**</font>    
  
　　**a. 通过getctivity和findFragmentByTag/findFragmentById**    
　　尽管Fragment是独立于Activity的一个对象，但是Fragment可以通过getActivity()方法获取Activity的实例，这样就可以执行Activity中的公有方法了。  
　　同样的，Activity可以通过FragmentManager获取一个引用来调用Fragment中的方法，使用findFragmentByTag()或者findFragmentById();　  
	
　　**b. 通过interface接口**  
　　在Fragment中：  

```
public class FragmentDemo extends Fragment {

    public interface OnComListener {
        public void onComListener();
    }

    OnComListener mListener;

    @Override public void onAttach(Context context) {
        super.onAttach(context);
        try {
            mListener = (MainActivity) context;
        } catch (ClassCastException e) {
            throw new ClassCastException("");
        }
    }
}
```

　　在Activity中：  

```
public class MainActivity extends AppCompatActivity implements FragmentDemo.OnComListener {

    @Override public void onComListener() {
    }
}
```
　　这样你就可以通过mListener在Fragment中调用Activity的接口了。同理，在Activity中也可以调用Fragment的方法。   

　　**再进一步**： 通过Interface的方法可以实现Fragment与Fragment之间的通信，但是必须借助Activity进行中转，或者综合使用Interface和findFragmentByTag()方法实现Fragment通信，也必须通过Activity中转。   

　　**c. 通过startActivityForResult()**  
　　Fragment中启动Activity：在support 23.2.0以下的支持库中，对于在嵌套子Fragment的startActivityForResult()，会发现无论如何都不能在onActivityResult()中接收到返回值，只有最顶层的父Fragment才能接收到，这是一个support v4库的一个BUG，不过在最新发布的support 23.2.0库中，已经修复了该问题，嵌套的子Fragment也能正常接收到返回数据了!

　　其实在Fragment中可以自定义这样的方法：FragmentA启动FragmengB，FragmentB回退的时候讲内容返回给FragmentA。 下面我们就实现这个方法。  


<font size = 5>**8 。 Fragment栈内容的查看**</font>    

　　每个Fragment以及容器Activiy都会在创建时初始化一个FragmentManager对象。  

　　**a.** 对于容器Activity，通过FragmentManager可以查看Activity作为容器的Fragment。  

```
List<Fragment> fragmentList = mActivity.getSupportFragmentManager().getFragments();
```
　　**b.** 对于容器Fragment，通过Fragment的getChildFragmentManager可以查看Fragment作为容器的FragmentManager对象  

```
List<Fragment> fragmentList = fragment.getChildFragmentManager().getFragments();
```

　　**c.** 对于容器Fragment，通过getFragmentManager是获取的父Fragment（若没有，获取的是Activity容器）的FragmentManager对象。  

　　通过查看Fragment的栈关系就可以知道自己的程序是不是出现错误，有没有内存泄露的风险等。  

<font size = 5>**9 。懒加载（延迟加载）技术**</font>    

　　我们在做应用开发的时候，如果你的activity里可能会用到多个Fragment。如果每个Fragment都需要去加载数据资源（from本地或者network），那么这个Activity在创建的时候需要初始化大量的资源。正确的做法应该是当切换到这个Fargment的时候，采取初始化。  
　　实现方法就是 



<font size = 5>**10 。防止多次点击加载Fragment**</font>   


<font size = 5>**11 。onBackPressed的使用**</font>  
