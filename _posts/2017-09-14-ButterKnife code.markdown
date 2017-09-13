---
layout:     post
title:      "ButterKnife实现原理探析"
subtitle:   "本文重点讲解ButterKnife的原理，看看JakeWharton大神如何优雅的实现View自动绑定"
date:       2017-9-14 12:00:00
author:     "foxleezh"
header-img: "img/post-bg-2017-09-14.jpg"
catalog: true
tags:
    - ButterKnife 原理 源码
---

## 前言
这个框架大家都是特别熟悉的了，JakeWharton大神的作品，[项目地址](https://github.com/JakeWharton/butterknife)，怎么用我就不多讲了，可以去参考[官方文档](http://jakewharton.github.io/butterknife/)，今天主要来扒一扒里面的源码

本文讲解项目地址 [TestButterKnife](https://github.com/foxleezh/TestButterKnife)   基于butterknife:8.8.1

## 一、ButterKnife能解决什么问题？
我个人认为主要是两个方面：
* 1.避免写烦琐的findviewbyid,以及一些资源的获取，比如getString
* 2.避免写一些交互事件，比如setOnclickListener

平常我们要绑定View到Activity上，都得一步步写findviewbyid，setOnclickListener,代码显得多余，而用了butterKnife就简洁了很多，而且配合一些插件比如Android ButterKnife Zelezny,一键生成所有需要的ButterKnife代码，写起来不要太爽！

## 二、编译器原理
首先我们要明白一点，ButterKnife是一个框架，不是什么高级API，虽然看似新奇的用法，背后也只是调用我们平常使用的那些findviewbyid等等，它之所以能如此简洁，是因为它用了Java Annotation Processing（Java注解解析器）。通俗讲，就是它在Java编译期间通过注解生成了一些实体类，这些类可以辅助我们绑定view到Activity上，也就省去了我们自己去写findviewbyid这些重复的代码。

有关Java Annotation Processing的使用本文不作深入讲解，有兴趣的可以去看下[ButterKnifeProcessor](https://github.com/JakeWharton/butterknife/blob/master/butterknife-compiler/src/main/java/butterknife/compiler/ButterKnifeProcessor.java)这个类的process方法实现。总之在编译后，只要我们在类中使用了ButterKnife的注解，就会生成一个相应的辅助类，比如我们在MainActivity中使用注解，就会生成一个MainActivity_ViewBinding的辅助类。

## 三、神秘的_ViewBinding
我们以MainActivity为例，代码如下：
```java
public class MainActivity extends AppCompatActivity {


    @BindView(R.id.btn_jump)
    Button btnJump;

    @BindString(R.string.jump)
    String stringJump;

    private Unbinder unbinder;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        unbinder=ButterKnife.bind(this);
        btnJump.setText(stringJump);
    }


    @OnClick(R.id.btn_jump)
    public void onViewClicked() {
        startActivity(new Intent(this,FragmentJumpActivity.class));
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbinder.unbind();
    }
}
```
我们从注册方法开始讲起：

#### 3.1 ButterKnife.bind

MainActivity中的onCreate方法调用了unbinder=ButterKnife.bind(this)这一句，该方法返回了一个Unbinder对象，这是一个接口，主要用来注销绑定的
```java
public interface Unbinder {
  @UiThread void unbind();

  Unbinder EMPTY = new Unbinder() {
    @Override public void unbind() { }
  };
}
```
我们接着ButterKnife.bind方法读
```java
  @NonNull @UiThread
  public static Unbinder bind(@NonNull Activity target) {
    View sourceView = target.getWindow().getDecorView();
    return createBinding(target, sourceView);
  }

  /**
   * BindView annotated fields and methods in the specified {@link View}. The view and its children
   * are used as the view root.
   *
   * @param target Target view for view binding.
   */
  @NonNull @UiThread
  public static Unbinder bind(@NonNull View target) {
    return createBinding(target, target);
  }
```
这里边有好几个重载的方法，其实的都是指向的createBinding方法，不同的只是将组件对应的View获取到，比如Activity是getWindow().getDecorView()，返回的是Activity的顶层View
，看过Activity源码的都知道，getDecorView返回的是一个FrameLayout。我们重点看createBinding方法：
```java
  private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    Class<?> targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

    if (constructor == null) {
      return Unbinder.EMPTY;
    }

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
      return constructor.newInstance(target, source);
    } catch (IllegalAccessException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InstantiationException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InvocationTargetException e) {
      Throwable cause = e.getCause();
      if (cause instanceof RuntimeException) {
        throw (RuntimeException) cause;
      }
      if (cause instanceof Error) {
        throw (Error) cause;
      }
      throw new RuntimeException("Unable to create binding instance.", cause);
    }
  }
```
这个方法的逻辑也不多，主要是findBindingConstructorForClass方法得到了一个构造函数，然后通过constructor.newInstance构造一个对象返回，如果constructor为空，默认返回Unbinder.EMPTY，这个在之前Unbinder源码中可以看到，就是一个空实现。我们继续看findBindingConstructorForClass方法
```java
    @VisibleForTesting
    static final Map<Class<?>, Constructor<? extends Unbinder>> BINDINGS = new LinkedHashMap<>();

    @Nullable @CheckResult @UiThread
    private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }
    try {
      Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
      //noinspection unchecked
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
  }
```

这段代码也不复杂，我们一段段看：
```java
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
```
BINDINGS是一个LinkedHashMap，首先是从BINDINGS里读缓存，如果有对应的Unbinder且不为空，直接返回。
```java
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }
```
跳过以android.和java.开头的Class，因为这些是framework层的类，直接返回null
```java
    try {
      Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
      //noinspection unchecked
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
```
将传进来的类名加上_ViewBinding，也就是我们之前讲的那个辅助类，获取它的构造方法，然后将这个构造方法缓存到BINDINGS中，最后返回该构造方法

这样的话，结合之前的逻辑ButterKnife.bind方法就返回了一个辅助类XXX_ViewBinding的实例，那我们那些view怎么绑定到Activity上的呢？其实是在XXX_ViewBinding里面做的

#### 3.2 _ViewBinding

MainActivity对应生成的MainActivity_ViewBinding代码如下：
```java
public class MainActivity_ViewBinding implements Unbinder {
  private MainActivity target;

  private View view2131427419;

  @UiThread
  public MainActivity_ViewBinding(MainActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  @UiThread
  public MainActivity_ViewBinding(final MainActivity target, View source) {
    this.target = target;

    View view;
    view = Utils.findRequiredView(source, R.id.btn_jump, "field 'btnJump' and method 'onViewClicked'");
    target.btnJump = Utils.castView(view, R.id.btn_jump, "field 'btnJump'", Button.class);
    view2131427419 = view;
    view.setOnClickListener(new DebouncingOnClickListener() {
      @Override
      public void doClick(View p0) {
        target.onViewClicked();
      }
    });

    Context context = source.getContext();
    Resources res = context.getResources();
    target.stringJump = res.getString(R.string.jump);
  }

  @Override
  @CallSuper
  public void unbind() {
    MainActivity target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");
    this.target = null;

    target.btnJump = null;

    view2131427419.setOnClickListener(null);
    view2131427419 = null;
  }
}
```
这个类其实也很简单，主要分为两部分：
* 1.在构造方法中绑定view和事件
* 2.在unbind方法解除绑定

在之前ButterKnife.bind方法中，有这样一句代码
```java
constructor.newInstance(target, source);
```
其中target为MainActivity对象，source为decorview,也就是说，会走第二个构造方法，我们一段段分析
```java
======华丽分割线========================================
    this.target = target;
    View view;
    view = Utils.findRequiredView(source, R.id.btn_jump, "field 'btnJump' and method 'onViewClicked'");
    target.btnJump = Utils.castView(view, R.id.btn_jump, "field 'btnJump'", Button.class);
    view2131427419 = view;
======华丽分割线========================================

  public static View findRequiredView(View source, @IdRes int id, String who) {
    View view = source.findViewById(id);
    if (view != null) {
      return view;
    }
    String name = getResourceEntryName(source, id);
    throw new IllegalStateException("Required view '"
        + name
        + "' with ID "
        + id
        + " for "
        + who
        + " was not found. If this view is optional add '@Nullable' (fields) or '@Optional'"
        + " (methods) annotation.");
  }

  public static <T> T castView(View view, @IdRes int id, String who, Class<T> cls) {
    try {
      return cls.cast(view);
    } catch (ClassCastException e) {
      String name = getResourceEntryName(view, id);
      throw new IllegalStateException("View '"
          + name
          + "' with ID "
          + id
          + " for "
          + who
          + " was of the wrong type. See cause for more info.", e);
    }
  }
```
这里其实对应是MainActivity中@BindView(R.id.btn_jump)这个注解，Utils.findRequiredView内部实现其实就是我们经常用的findviewbyid,而Utils.castView其实就是向下强转，所以这几句代码的功能就相当于
```java
Button btnJump= (Button) source.findViewById(R.id.btn_jump);
```
那干嘛要写那么多代码呢？为啥不用这句直接代替，主要是为了找不到id的时候抛出对应的异常，便于开发者发现问题。
```java
    view.setOnClickListener(new DebouncingOnClickListener() {
      @Override
      public void doClick(View p0) {
        target.onViewClicked();
      }
    });
```
这里对应的是MainActivity中@OnClick(R.id.btn_jump)的注解，就是普通的setOnClickListener，然后回调MainActivity中的onViewClicked方法
```java
    Context context = source.getContext();
    Resources res = context.getResources();
    target.stringJump = res.getString(R.string.jump);
```
这里对应的是MainActivity中@BindString(R.string.jump)的注解，也是普通的res.getString方法

所以综上可以看出，ButterKnife就是用了我们平时那些findViewById,setOnClickListener实现view的绑定。

#### 3.3 unbinder.unbind();
我们一般会在onDestroy方法中调用unbinder.unbind
```java
  public void unbind() {
    MainActivity target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");
    this.target = null;

    target.btnJump = null;

    view2131427419.setOnClickListener(null);
    view2131427419 = null;
  }
```
这里边就是一些置空的操作，便于垃圾回收器回收对应的内存

## 小结

###### 1.ButterKnife首先会在编译期间利用Java Annotation Processin生成一个XXX_ViewBinding的类。
###### 2.ButterKnife.bind中通过反射实例化XX_ViewBinding，完成View的各项绑定，实现方式就是普通的findviewbyid等。
###### 3.unbinder.unbind方法是一些置空操作，便于垃圾回收器回收内存。