# Android_Framework_Releated

    该仓库文章旨在总结Android Framework相关知识，从app开发者日常接触到的场景进入Android Framework，从系统层面理解应用的运行流程及原理。
目前Aosp已经开放Android13源代码，因此基于android-13.0.0_r1分支的源码进行分析总结。

1. 什么是Android framework ?
    众所周知，Android基于Linux系统，从上到下依次分为 App -> Framework -> Hal驱动 -> Kernel。 Framework层为App提供了API支持，将API的实现细节封装起来，App只
需调用相关API就能达到相应的目的，不必关心其实现细节。
    
2. 为什么需要了解学习framework ?
    对于Android系统工程师来讲，熟悉framework是必须的，否则日常工作无法正常进行。
    对于App开发者来说，一般只需关注API的调用是否符合预期；但是在Android碎片化的今天，各大手机厂商会基于AOSP定制符合自己特色的系统，因此相同的代码在不同厂商
的机器上可能会出现不同的行为，此时如果依赖于Google/百度，未免显得有些被动，如果熟悉framework，针对不同厂商的系统进行研究，找到系统的差异点，就可以进行相应的
适配，提升应用的兼容性； 再者来说，熟悉系统的运行原理，在设计应用及定位bug时会更加占据主动权。

3. 如何入门/学习framework ?
    Android app的入门是极其简单的，一个没接触过Android的人在看完教程之后，也能很快的写出Hello World的demo app。 
    很直观的来讲，从桌面点击Hello World demo app, 然后到demo app的界面显示出来，应该是最为常见的场景，那么在这个过程中app开发者只是设置了布局以及app的入口Activity,
从点击到显示完成的过程是由系统来进行的，那么在这个过程中系统中做了什么？ 笔者觉得从这里入门学习framework可能是比较合适的点， 并且这个问题也是面试时会被问到的高频问
题， 如果只能答出create start resume相关的东西，想来是没什么竞争力的。
    因此该仓库会从先这个问题来进行分析，进而延申，来对framework有一个大概的认识。
    
4. 熟悉framework可以做什么？
    笔者觉得系统规则就像法律一样，你越是了解它， 你就知道如何在法律范围内更好的行事； 再或者利用系统的规则去完成一些比较黑科技的事情，诸如虚拟容器、各种类型的hook都
是在对系统深入研究了解之后产生的产物；对于维护系统的工程师来说，往往从0到1的事情比较少，往往都是熟悉了解原有的规则，而后参照进行实现需求/解决bug。

PS: 对于学习系统源码来讲，最好是能够下载编译源码进行真机/模拟器烧录，改动代码之后直接可以看到修改的效果。 
当然也可以直接查看Android官方源码进行学习(https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:)
