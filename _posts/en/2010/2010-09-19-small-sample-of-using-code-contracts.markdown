---
layout: post
title: "Small sample of using Code Contracts"
date: 2010-09-19 17:04:39
categories: en
tags: [.NET, C#, WPF, Code Contract, ReSharper]
alias: en/blog/show/242
---
<p>I had read about Code Contracts long time ago, but I didn’t see a reason to use contract instead of simple tests and argument's check with Exception's throws. Only now I’m trying to use them at real project. I didn’t see why I need to write:</p>  

```
Contract.Requires<ArgumentNullException>(argument == null); 
```

<p>Instead of like I did it before:</p>

```
if (argument == null) 
    throw new ArgumentNullException(“argument”);
```

<p>Couple of weeks ago I had reinstall my Windows 7 on my laptop (I bought SSD for my laptop). After I install ReSharper with Visual Studio 2010 after run I accidental choosed “Go to the metadata” instead of like usual “Go to Object Explorer” by Ctrl and Mouse Click. So when I clicked on some class of System.xxx (basic classes of .NET) I looked at source code of these classes and found that they are using Code Contracts. I thought this is why I need to understand why I need to use contracts in my code, so I started to use them in some small applications and tried to find some interesting case.</p>



<p>You can look at <a href="/en/blog/show/240">previous blog post</a> and find that I used code contracts in this sample. Actually it was first using code contracts. If you don’t know how to start using code contract in your project: you should go to the <a href="http://research.microsoft.com/en-us/projects/contracts/">Microsoft Research site</a>, download last code contracts installer and better to download last <a href="http://visualstudiogallery.msdn.microsoft.com/en-us/85f0aa38-a8a8-4811-8b86-e7f0b8d8c71b">Editor Extension</a> as well. When you will do this, you will see two additional tabs in project properties window.</p>

<p>So, let's look at an example. We have class like this:</p>

```
public sealed class HotKey 
{
    private IntPtr _handle;
 
    public HotKey(Window window)
        :this (new WindowInteropHelper(window))
    {
    }
 
    public HotKey(WindowInteropHelper window)
        :this(window.Handle)
    {
    }
 
    public HotKey(IntPtr windowHandle)
    {
        _handle = windowHandle;
    }
}
```

<p>In this class we are working with window’s handle, but we allow to create instance of this class not only with Handle as paramters, but with Window or WindowInteropHelper objects as well. What will happen if user will try to create new HotKey instance and put null instead of Window object (first constructor)? We will get Exception with message “<em>Value cannot be null. Parameter name:window</em>.”, this exception will throw constructor of type WindowInteropHelper. So it is good luck that we will get understandable message. But what about null instead of WindowInteropHelper (second constructor)? Try it. You will get <em>NullReferenceException</em> with message “Object <em>reference not set to an instance of and object</em>”. No good. How we can handle it? We can try to use separate method Initialize:</p>

```
public sealed class HotKey
{
    private IntPtr _handle;
 
    public HotKey(Window window)
    {
        if (window == null)
            throw new ArgumentNullException("window");
 
        Initialize(new WindowInteropHelper(window).Handle);
    }
 
    public HotKey(WindowInteropHelper window)
    {
        if (window == null)
            throw new ArgumentNullException("window");
 
        Initialize(window.Handle);
    }
 
    public HotKey(IntPtr windowHandle)
    {
        Initialize(windowHandle);
    }
 
    private void Initialize(IntPtr windowHandle)
    {
        _handle = windowHandle;
    }
}
```

<p>Аnd also you can use contracts:</p>

```
public sealed class HotKey 
{
    private IntPtr _handle;
 
    public HotKey(Window window)
        : this (new WindowInteropHelper(window))
    {
        Contract.Requires(window != null);
    }
 
    public HotKey(WindowInteropHelper window)
        : this(window.Handle)
    {
        Contract.Requires(window != null);
    }
 
    public HotKey(IntPtr windowHandle)
    {
        _handle = windowHandle;
    }
}
```

<p>Looks like that if you will pass null instead of WindowInteropHelper in second constructor you will get NullReferenceException again, because first will executed code in base constructor and next constructor’s body which we invoke. And R# tell us that we don’t need to check <em>window!=null</em>, because we used this variable before considering that it can not be null (I created <a href="http://youtrack.jetbrains.net/issue/RSRP-190932?projectKey=RSRP&amp;query=created+by:+me">bug</a> in ReSharper Youtrack):</p>

<p><img style="background-image: none; border-right-width: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top-width: 0px; border-bottom-width: 0px; border-left-width: 0px; padding-top: 0px" title="Untitled" border="0" alt="Untitled" src="{{ site.url }}/library/2010/09/14/Untitled_5DC85F5B.png" width="336" height="163" /></p>

<p>Actually contracts works before method execution, so we will see <em>Precondition failed exception</em>:</p>

<p><img style="background-image: none; border-right-width: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top-width: 0px; border-bottom-width: 0px; border-left-width: 0px; padding-top: 0px" title="Capture" border="0" alt="Capture" src="{{ site.url }}/library/2010/09/14/Capture_1D2612EC.png" width="789" height="266" /></p>

<p>Also if you want to see ArgumentNullException instead of ContractException you can write:</p>

```
public sealed class HotKey 
{
    private IntPtr _handle;
 
    public HotKey(Window window)
        : this (new WindowInteropHelper(window))
    {
        Contract.Requires<ArgumentNullException>(window != null);
    }
 
    public HotKey(WindowInteropHelper window)
        : this(window.Handle)
    {
        Contract.Requires<ArgumentNullException>(window != null);
    }
 
    public HotKey(IntPtr windowHandle)
    {
        _handle = windowHandle;
    }
}
```

<p>Or:</p>

```
public sealed class HotKey
{
    private IntPtr _handle;
 
    public HotKey(Window window)
        :this (new WindowInteropHelper(window))   
    {
        if (window == null)
            throw new ArgumentNullException("window");
        Contract.EndContractBlock();
    }
 
    public HotKey(WindowInteropHelper window)
        : this(window.Handle)
    {
        if (window == null)
            throw new ArgumentNullException("window");
        Contract.EndContractBlock();
    }
 
    public HotKey(IntPtr windowHandle)
    {
        _handle = windowHandle;
    }
}
```

<p>So all next code only with contracts! <img style="border-bottom-style: none; border-left-style: none; border-top-style: none; border-right-style: none" class="wlEmoticon wlEmoticon-winkingsmile" alt="Подмигивающая рожица" src="{{ site.url }}/library/2010/09/19/wlEmoticonwinkingsmile_09C46272.png" /></p>
