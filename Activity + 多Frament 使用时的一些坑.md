# **单个Activity + 多个Fragment的一些坑**  

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

<font size = 5>**4 。 Frament的出栈**</font>  
  
　　FragmentManager提供了remove方法用于Fragment的出栈，但是这个方法是有问题的。　

<font size = 5>**5 。 Fragment与Fragment之间的数据传输**</font>    
  
　　**Fragment与Activity通信**    
　　尽管Fragment是独立于Activity的一个对象，但是Fragment可以通过getActivity()方法获取Activity的实例，并且可以很容易的执行比如查找Activity中View的任务。  
　　同样的，Activity可以通过FragmentManager获取一个引用来调用Fragment中的方法，使用findFragmentByTag(), 获取findFragmentById();　

　　**Fragment之间的通信**  
　　在Activity中存在startActivityForResult方法，其实在Fragment中可以自定义这样的方法，来实现Fragment之间的通信。  

```

```


<font size = 5>**6 。 Fragment栈内容的查看**</font>  

　　通过FragmentManager可以查看Activity作为容器下的Fragment。  

```
List<Fragment> fragmentList = mActivity.getSupportFragmentManager().getFragments();
```
　　通过Fragment的getChildFragmentManager可以查看Fragment作为容器下的Fragment  

```
List<Fragment> fragmentList = parentFragment.getChildFragmentManager().getFragments();
```
　　通过查看Fragment的栈关系就可以知道自己的程序是不是出现错误，有没有内存泄露的风险等。  

<font size = 5>**7 。 **</font>  
