# **单个Activity + 多个Fragment的一些坑**  

　　目前单个（多个）Activity + 多个Fragment已经成为主流APP的页面呈现方式，例如：微信、今日头条等。下面就阐述一下Activity + 多个Fragment的使用过程中的一些问题和经验教训。  

<font size = 5>**1 。 getActivity() = null**</font>  

　　当你用Fragment的时候，经常会碰到这个问题，容器对象为空。

<font size = 5>**2 。 Fragment接口的选择：replace or show hide**</font> 

 
<font size = 5>**3 。 Fragment导致的节目重叠**</font>  

<font size = 5>**4 。 Frament的出栈**</font>  

<font size = 5>**5 。 Fragment与Fragment之间的数据传输**</font>  

<font size = 5>**6 。 Fragment栈内容的查看**</font>  

<font size = 5>**7 。 **</font>  
