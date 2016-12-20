---
title: 使用RxJava
date: 2016-10-20 10:26:43
tags:
- RxJava
categories:
- Android
---

很惭愧这么晚才接触RXJava,更惭愧在接触到之后没有立即亲自使用。偶然间的惊鸿一瞥发现原来异步可以有这么简单优雅的方式。
[RxJava](https://github.com/ReactiveX/RxJava)是[ReactiveX](https://mcxiaoke.gitbooks.io/rxdocs/content/Observables.html)的一个分支，还有RxJS,RxSwift等等。RxJava的观察者模式只是一个最基本的标准，而真正强大的地方是它的**Operators**（操作符），各种变化，组合，数据操作等。

Reactive的意思是响应式，最核心的两个东西是**Observer**（观察者）和**Observable**（被观察者）,Observer对Observable的动作做出响应，有时Observer也叫做Subscriber。借用官网的一张图如下：
![](http://ofcnzxaa4.bkt.clouddn.com/RxJava.jpg)

### RxJava是什么
RxJava在GitHub主页上的自我介绍是:"a library for composing asynchronous and event-based programs using observable  sequences for the Java VM"（一个在Java VM上使用可观测的序列来组成异步的、基于事件的程序的库）。

### 什么是基于事件
**基于事件**这个名词猛一看不好理解，换个角度，我们想象成工厂流水线，流水线上有多个工人，每个工人对产品的处理就当成一个事件，我们对数据的每一次操作就想象成每个流水线上的工人对产品的处理，产品处理完毕之后交给产品经理相当于数据处理完毕之后响应的操作。
### 异步
在Android中，Google为我们提供了很多种异步的方法，比如AsyncTask,Handler，或者我们手动建立一个任务线程来处理，不管是哪种方式，都让我们的代码显得臃肿不堪，更为不堪入目的是当我们的业务逻辑复杂的时候，我们可能会在回调中处理数据，然后再回调，可能还需要切换到主线程更新UI.....，并不是说AsyncTask,Handler不好，而是相对于RxJava来说显得臃肿。而RxJava可以让异步如此优雅，让逻辑更加清晰明了，让切成线程变得如此简单。说了这么多怎么能不来个简单的例子就说不过去。假设要再一个工作线程里遍历一个数组，取出来的字符串然后添加到List中上：
#### 先来我们常见的
```java
final String[] names = {"A", "B", "C"};
new Thread(new Runnable() {
    @Override
    public void run() {
        for(String name:names){
            MainActivity.this.runOnUiThread(new Runnable() {
                @Override
                public void run() {
                   mList.add(name);
                }
            });
        }
    }
}).start();
```
这里肯定有人说你遍历个数组还要开一个线程多此一举吧？我这只是在示例，当然实际的时候我们用来做一些耗时任务。如果我们使用了Handler或者AsyncTask代码量也是一堆。
#### 再来RxJava实现的

```java
final String[] names = {"A", "B", "C"};
Observable.from(names)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<String>() {
        @Override
        public void call(String s) {
            mList.add(s);
        }
    });
```

虽然代码量看着没有很明显的减少很多，但是逻辑更加清晰，如果使用Lambda表达式更为简洁。

```java
final String[] names = {"A", "B", "C"};
Observable.from(names)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(s -> mList.add(s));
```

### RxJava的观察者模式
Obeservable和Observer之间通过`subscribe()`（订阅）方法相关联，一个Observable可以发出零个或者多个事件，直到结束或者出错。每发出一个事件，就会调用它的Subscriber的`onNext()`方法，最后调用`Subscriber.onNext()`或者`Subscriber.onError()`结束。可能有时候我们只会用到其中的一个，比如上面例子中只是用了`onNext()`，所以我们使用了Action1<T>,当然还有Action2，Action3....,区别是参数的个数不一样，这里只需要传递一个字符串，所以就选择了Action1。

### 创建Observable

#### 最基本的create()
```java
Observable<String> observable = Observable.create(new Observable.OnSubscribe<String>() {
        @Override
        public void call(Subscriber<? super String> subscriber) {
            subscriber.onNext("A");
            subscriber.onNext("B");
            subscriber.onNext("C");
            subscriber.onCompleted();
        }
});
```

这里用的了一个新的名词**Subscriber**，这个类是对Observer的扩展，多了两个主要的方法。

* `onStart()`:在Observable发出消息之前做一些初始化的操作
* `unsubscribe()`:用于取消订阅，在执行此方法之前可以用`isUnsubscribed()`进行判断，这两个方法都是继承的另一个接口**Subscription**的抽象方法。取消订阅的重要性不言而喻，及时释放引用防止OOM。

#### 常见的just()和from()
```java
Observable<String> observable =Observable.just("A", "B", "C");
```
```java
final String[] names = {"A", "B", "C"};
Observable<String> observable1 = Observable.from(names);
```
创建Observable的操作有很多很多，如**defer**,**interval**,**range**等等，放到以后的例子中演示。
### 变换操作
还是上面的例子，现在在每一个name后面添加一个"D"。
#### 第一种方法可以直接在响应的方法里操作
```java
final String[] names = {"A", "B", "C"};
Observable.from(names)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<String>() {
        @Override
        public void call(String s) {
            mList.add(s+"D");
        }
    });
```
这种方法虽然可以实现，但是违反了RxJava的初衷，应该吧变化操作放到响应之前。
#### 第二种方法用Map操作
```java
final String[] names = {"A", "B", "C"};
Observable.from(names)
    .map(new Func1<String, String>() {
        @Override
        public String call(String s) {
            return s+"D";
        }
    })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<String>() {
        @Override
        public void call(String s) {
            mList.add(s);
        }
    });
```
这里又出现了一个新名词**Func1**，Func跟Action唯一的区别就是Func有返回值。
map操作一般用于对原始的参数进行加工处理，返回值还是基本的类型，更为神奇的是还可以类型的转换比如下面的例子把Integer转化为String。

```java
final Integer[] names = {1, 2, 3};
Observable.from(names)
    .map(new Func1<Integer, String>() {
        @Override
        public String call(Integer integer) {
            return String.valueOf(integer);
        }
    })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<String>() {
        @Override
        public void call(String s) {
            mList.add(s);
        }
    });
```
现在改变需求，假设有多个文件夹folders,每个文件夹里有多个图片，过滤掉大于1M的图片，然后添加到List中，看到这个需求如果用传统的方法嵌套回调简直是噩梦。

```java
new Thread() {
    @Override
    public void run() {
        super.run();
        for (File folder : folders) {
            File[] files = folder.listFiles();
            for (int i = 0;i<5;i++) {
                if(files[i]!=null){
                    if (file.length()<1024*1000) {
                        final Bitmap bitmap = getBitmapFromFile(file);
                        MainActivity.this.runOnUiThread(new Runnable() {
                            @Override
                            public void run() {
                                mList.add(bitmap);
                            }
                        });
                    }
                }
            }
        }
    }}.start();
```

来看看RxJava的实现方式：

```java
Observable.from(folders)
    .flatMap(new Func1<File, Observable<File>>() {
        @Override
        public Observable<File> call(File file) {
            return Observable.from(file.listFiles());
        }
    })
    .filter(new Func1<File, Boolean>() {
        @Override
        public Boolean call(File file) {
            return file.length()<1024*1000;
        }
    })
    .take(5)
    .map(new Func1<File, Bitmap>() {
        @Override
        public Bitmap call(File file) {
            return convertToBitmap(file);
        }
    })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<Bitmap>() {
        @Override
        public void call(Bitmap bitmap) {
            mList.add(bitmap);
        }
    });
```
可以很明显的感觉到随着业务逻辑的繁琐，传统方式的代码越来越臃肿而且思路不明确。
**flatMap**跟map类似，区别在于flatMap的回调方法返回的是一个Observables，flatMap()操作符使用你提供的原本会被原始Observable发送的事件，来创建一个新的Observable。而且这个操作符，返回的是一个自身发送事件并合并结果的Observable。
**filter**操作就是过滤，这里是过滤掉大于1M的图片。
**take**操作是取出来前几个，当然还有takeLast取出来倒数几个等等。

### 创建Observer

```java
Observer<String> observer = new Observer<String>() {
    @Override
    public void onNext(String s) {

    }

    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }};
```

```java
Subscriber<String> subscriber = new Subscriber<String>() {
    @Override
    public void onNext(String s) {
        Log.d(tag, "Item: " + s);
    }

    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }};

```

RxJava的强大之处在于各种神奇的操作符，并且可以结合Retrofit2，Dagger2来使用。
