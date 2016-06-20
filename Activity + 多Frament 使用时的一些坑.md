# **Activity + 多Fragment的一些坑** 

　　目前单个（多个）Activity + 多个Fragment已经成为主流APP的页面呈现方式，例如：微信、今日头条等。下面就阐述一下Activity + 多个Fragment的使用过程中的一些问题和经验教训。  

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
　　出现了内存回收，怎样复现这个场景呢？见“[程序异常数据的丢失](程序异常数据的丢失.md)”。当出现内存回收，系统通过FragmentManager会保存他的Fragments，再次启动进入系统会从栈底向栈顶的顺序恢复Fragments，并且每一个恢复的Fragment都是以show()的方式显示出来，从而导致界面的重叠。  
　　**解决办法：**  
　　在add(), replace()的时候，绑定一个tag（一般都是使用Fragment的类名），发生内存重启的时候，通过findFragmentByTag找到对应的Fragment，并hide掉需要隐藏的Fragment(可以在Activity的onSaveInstance中保存需要显示的Fragment的tag值)。  

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
  
  
<font size = 5>**4 。 Frament的出栈remove的使用问题**</font>  
  
　　FragmentManager提供了remove方法用于Fragment的出栈，但是这个方法是有问题的。  
　　如下面的代码所示：我添加了两个Fragment到Activity容器中，当用户点击back按键的时候，我通过如下方法进行出栈操作，你会发现：getBackStackEntryCount()方法永远都是返回2个。但是通过日志你客可看到你的两个Fragment的onDetac方法都被调用了。

```
public void onBackPressed() {
        int count = mFragmentManager.getBackStackEntryCount();
        Log.d(tag, "fragment count = " + count);
        if (count > 0) {
            String tag = mFragmentManager.getBackStackEntryAt(count - 1).getName();
            Fragment fragment = mFragmentManager.findFragmentByTag(tag);
            FragmentTransaction ft = mFragmentManager.beginTransaction().remove(fragment);
            ft.commit();
            //mFragmentManager.popBackStack();
        }
        else {
            super.onBackPressed();
        }
    }
```

　　如上所示，正确的使用方法是调用popBackStack系列函数。  
　　如果你觉着popBackStack系列函数是木有问题的， 那么你就错了。  

<font size = 5>**5。 Fragment的popBackStack系列接口的使用问题**</font>   
 
　　popBackStack()  
　　popBackStackImmediate()  
　　popBackStack(String tag,int flags)  
　　popBackStack(int id,int flags)  
　　popBackStackImmediate(String tag,int flags)  
　　popBackStackImmediate(int id,int flags)  

　　Android总共提供了6个popBackStack()系列函数，区别如下：  
　　a、前两个函数用户单个Fragment的出栈，后面四个用户多个Fragments的出栈；  
　　b、popBackStack和popBackStackImmediate的区别在于前者是加入到主线队列的末尾，等其它任务完成后才开始出栈，后者是立刻出栈。正因为有了popBackStack和popBackStackImmediate的这个区别，所有在调用的时候就需要格外的注意:  
　　如果你在调动popBackStackImmediate之后紧接着调用类似如下的事物方法，会导致内存重启后按返回键报错的问题：

```
getSupportFragmentManager().popBackStackImmdiate();
getSupportFragmentManager().beginTransaction()
        .add(R.id.container, fragment , tag)
        .hide(currentFragment)
        .commit;
```

　　正确的使用方法如下所示：

```
getSupportFragmentManager().popBackStackImmdiate();
new Handler().post(new Runnable(){
          @Override
           public void run() {
                // 在这里执行Fragment事务
           }
});
```



<font size = 5>**6 。 Fragment与Fragment/Activity之间的数据传输**</font>    
  
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

```
```

<font size = 5>**7 。 Fragment栈内容的查看**</font>    

　　每个Fragment以及容器Activiy都会在创建时初始化一个FragmentManager对象。  

　　**a. ** 对于容器Activity，通过FragmentManager可以查看Activity作为容器的Fragment。  

```
List<Fragment> fragmentList = mActivity.getSupportFragmentManager().getFragments();
```
　　**b.  ** 对于容器Fragment，通过Fragment的getChildFragmentManager可以查看Fragment作为容器的FragmentManager对象  

```
List<Fragment> fragmentList = fragment.getChildFragmentManager().getFragments();
```

　　**c. ** 对于容器Fragment，通过getFragmentManager是获取的父Fragment（若没有，获取的是Activity容器）的FragmentManager对象。  

　　通过查看Fragment的栈关系就可以知道自己的程序是不是出现错误，有没有内存泄露的风险等。  

<font size = 5>**8 。 **</font>  
