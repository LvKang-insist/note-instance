RxJava  的优点

- RxJava 能提高工作效率
- 简洁
- RxJava 能优雅的解决复杂的业务场景
- RxJava 越来越流行
- RxJava 屌爆了

响应式编程？

​	是一种基于异步数据流概念的编程模式

RxJava 是什么？

- 异步
- 异步数据处理库
- 扩展的观察者模式

RxJava 特点

- jar 包小于 1mb
- 轻量级框架
- 支持 Java8 lambda
- 支持java6+ & Android2.3+
- 支持异步和同步

RxAndroid 是什么？

- 是RxJava 针对Android 平台的一个扩展，用于Android 开发
- 提供响应式扩展组件 快速、易于开发 android 程序
- RxAndroid 其实就是 RxJava 在Android平台的一个扩展库

Schedulers(调度器)

​		简单的理解，就是在 RxJava 中我们可以指定一些我们相应的操作，在某一个线程中，我们想更新 UI ，我们使用 调度器来指定这个 UI 的更新操作就是在主线程。

- 解决 Android 主线程的问题(针对Android)

- 解决多线程的问题

  ​	比如有一个耗时的操作，必须在子线程中完成，子线程做完了之后要通过 Handler 发送到主线程。调度器解决了 子线程 和 主线程之间的通信问题，通过调度我我们就可以很简单的指定这些操作是在子线程 还是在主线程中。

观察者模式：

​	rxjava的实现主要是通过观察者模式实现的，下面简单介绍一下

​	观察者模式，发生改变的对象称为 观察目标(被观察者)，而被通知的对象称为观察者。一个观察者目标可以对应多个 观察者，且这些 观察者之间没有任何相互联系。可以根据需要增加或者删除 观察者。是的系统易于扩展。 

​	他们之间是一对多的关系，只要被观察者发生了改变，就会立马通知 观察者

​	例子：有多个警察 在 观察一个人，警察认为这个人是小偷，但是却没有证据。此时，只要那个人 进行了任何的违反活动，警察就需要去抓捕他。在这个例子中，小偷就是被观察者，警察则是观察者。



### 基本使用

创建 被观察者对象

```java
 Observable<Integer> observable = Observable.create(emitter -> {
                //发射一些数据，相当于调用被观察对象内部方法，通知所有的观察这
                emitter.onNext(1);
                emitter.onNext(2);
                emitter.onNext(3);
                //一旦调用 onComplete ，下面将不再接受事件
                emitter.onComplete();
            });
```

​	上面的 emitter 是一个发射器，他就是用来发送事件的，他发送事件后 观察者则会接收到他发送的事件，他可以发送三种类型的事件，通过调用emitter的onNext(T value)、onError(Throwable error)和onComplete()就可以分别发出next事件、error事件和complete事件。

创建 观察者模式

```java
 //观察者
            Observer<Integer> observer = new Observer<Integer>() {
                @Override
                public void onSubscribe(Disposable d) {
                    Log.e(TAG, "onSubscribe: " + d);
                }

                @Override
                public void onNext(Integer integer) {
                    Log.e(TAG, "onNext: " + integer);
                }

                @Override
                public void onError(Throwable e) {
                    Log.e(TAG, "onError: " + e);
                }

                @Override
                public void onComplete() {
                    Log.e(TAG, "onComplete: ");
                }
            };
```

其方法的含义如下：

- onSubscribe：他会在事件为发送之前调用，可以用来做一些中准备工作，而里面的Disposable 则是用来切断上下游的关系的。
- onNext：普通的事件，将要处理的事件添加到队列中
- onError：事件队列异常，在事件处理中出现异常时，此方法会被调用，同时队列会被终止，也就是不允许在有事件发出。
- onComplete：时间队列完成。RxJava 不仅会把每个时间单独处理，而且会把他们当成一个队列，当不在有 onNext 事件发出时，需要出发onComplete 方法作为完成标识符。



订阅，观察者订阅 被观察者。当被观察者发送事件后，观察者就会收到事件。

```java
observable.subscribe(observer);
```

结果如下所示：

```java
2019-06-28 19:30:58.455 13911-13911/com.admin.frame E/IndexDelegate: onSubscribe: CreateEmitter{null}
2019-06-28 19:30:58.455 13911-13911/com.admin.frame E/IndexDelegate: onNext: 1
2019-06-28 19:30:58.455 13911-13911/com.admin.frame E/IndexDelegate: onNext: 2
2019-06-28 19:30:58.455 13911-13911/com.admin.frame E/IndexDelegate: onNext: 3
2019-06-28 19:30:58.455 13911-13911/com.admin.frame E/IndexDelegate: onComplete: 
```

先调用 onSubsrcibe ,然后是 onNext，最后是 onComplete 收尾。

### 操作符

一般操作符是指，刚开始创建 被观察者的时候调用的，最基本的就是 create，在上面已经使用过了下面看一些常用的

#### just

此操作符是将传入的参数依次发出来

```java
//被观察者
Observable<Integer> observable = Observable.just(1,3,4,5,6);
```

结果如下：

```java
nSubscribe: io.reactivex.internal.operators.observable.ObservableFromArray$FromArrayDisposable@e4668fa
2019-06-28 19:43:48.346 14717-14717/com.admin.frame E/IndexDelegate: onNext: 1
2019-06-28 19:43:48.346 14717-14717/com.admin.frame E/IndexDelegate: onNext: 3
2019-06-28 19:43:48.346 14717-14717/com.admin.frame E/IndexDelegate: onNext: 4
2019-06-28 19:43:48.346 14717-14717/com.admin.frame E/IndexDelegate: onNext: 5
2019-06-28 19:43:48.346 14717-14717/com.admin.frame E/IndexDelegate: onNext: 6
2019-06-28 19:43:48.346 14717-14717/com.admin.frame E/IndexDelegate: onComplete: 
```

#### fromarray

将传入的数组 按照索引依次发出去

```java
Integer[] ints ={1,3,4};
Observable observable = Observable.fromArray(ints);
```

```java
2019-06-28 19:54:19.638 15097-15097/com.admin.frame E/IndexDelegate: onSubscribe: .....
2019-06-28 19:54:19.638 15097-15097/com.admin.frame E/IndexDelegate: onNext: 1
2019-06-28 19:54:19.638 15097-15097/com.admin.frame E/IndexDelegate: onNext: 3
2019-06-28 19:54:19.639 15097-15097/com.admin.frame E/IndexDelegate: onNext: 4
2019-06-28 19:54:19.639 15097-15097/com.admin.frame E/IndexDelegate: onComplete: 
```

#### interval

这其实就是一个定时器，用了它你就可以抛弃CountDownTimer 了，用法如下

```java
Observable.interval(3, TimeUnit.SECONDS)
                    .subscribe(new Observer<Long>() {
                        @Override
                        public void onSubscribe(Disposable d) {
                            Log.e(TAG, "onSubscribe: " + d);
                        }

                        @Override
                        public void onNext(Long aLong) {
                            Log.e(TAG, "onNext: " + aLong);
                        }

                        @Override
                        public void onError(Throwable e) {
                            Log.e(TAG, "onError: " + e);
                        }

                        @Override
                        public void onComplete() {
                            Log.e(TAG, "onComplete: ");
                        }
                    });
```

```java
2019-06-28 20:07:10.535 15391-15454/com.admin.frame E/IndexDelegate: onNext: 0
2019-06-28 20:07:13.536 15391-15454/com.admin.frame E/IndexDelegate: onNext: 1
2019-06-28 20:07:16.536 15391-15454/com.admin.frame E/IndexDelegate: onNext: 2
2019-06-28 20:07:19.536 15391-15454/com.admin.frame E/IndexDelegate: onNext: 3
2019-06-28 20:07:22.536 15391-15454/com.admin.frame E/IndexDelegate: onNext: 4
```

上面每个三秒 调一次onNext，上面的代码直接链式调用。

####  fromIterable 

​	直接发送一个List 集合数据给观察者

### 变换操作符

变换操作符的作用是对Observable发射的数据按照一定规则做一些变换操作，然后讲变换后的数据发射出去。变换操作符有map，flatMap，concatMap，switchMap，buffer，groupBy等等

#### map 

map 可以将被观察者发送的数据类型转变成其他的类型

```java
 Observable.just(1, 3, 4)
                    .map(integer -> integer + "")
                    .subscribe(new Observer<String>() {
                        @Override
                        public void onSubscribe(Disposable d) {

                        }
                        @Override
                        public void onNext(String s) {
                            Log.e(TAG, "onNext: " + s);
                        }

                        @Override
                        public void onError(Throwable e) {

                        }

                        @Override
                        public void onComplete() {

                        }
                    });
```



上面在链式调用的 map 方法里面使用了 lambda 表达式，箭头左边是参数，右边是 内容，因为只有一行代码，所以就没加 {}。在map 方法中 将 一个 Integer 类型的值 转为了一个字符串。

#### flatMap

这个方法可以将事件序列中的元素进行整合，返回一个新的 被观察者，

flatMap 返回的是一个 Observable。如下所示：

假设有一个 Person 类，如下

```java
public class Person {

    private String name;
    private List<Plan> planList = new ArrayList<>();

    public Person(String name, List<Plan> planList) {
        this.name = name;
        this.planList = planList;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public List<Plan> getPlanList() {
        return planList;
    }

    public void setPlanList(List<Plan> planList) {
        this.planList = planList;
    }

}

```

Person 类有一个 name 和 planList 两个变量，分别代表的是人名和计划清单。

```java
public class Plan {

    private String time;
    private String content;
    private List<String> actionList = new ArrayList<>();

    public Plan(String time, String content) {
        this.time = time;
        this.content = content;
    }

    public String getTime() {
        return time;
    }

    public void setTime(String time) {
        this.time = time;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public List<String> getActionList() {
        return actionList;
    }

    public void setActionList(List<String> actionList) {
        this.actionList = actionList;
    }
}

```
现在需要将这个 Person集合中的每个元素的 Plan 的action 打印出来，如下所示

```java
  		  //插入一些数据
  		   List<Person> list = new ArrayList<>();
            List<Plan> plans = new ArrayList<>();
            Plan plan = new Plan("时间", "内容");
            List<String> strings  = new ArrayList<>();
            strings.add("哈哈哈");
            strings.add("嘻嘻嘻");
            plan.setActionList(strings);
            plans.add(plan);
            Person person = new Person("张三",plans);
            person.setPlanList(plans);
            list.add(person);

            // fromIterable 直接发送一个List 集合数据给观察者
            Observable.fromIterable(list)
            		//Fction 类的两个泛型，第一个为接收的类型，第二个则是返回的类型
                    .flatMap(new Function<Person, ObservableSource<Plan>>() {
                        @Override
                        public ObservableSource<Plan> apply(Person person) throws Exception {
                            return Observable.fromIterable(person.getPlanList());
                        }
                    })
                    //上面返回的类型会这里接收的类型，这里返回 String
                    .flatMap(new Function<Plan, ObservableSource<String>>() {

                        @Override
                        public ObservableSource<String> apply(Plan plan) throws Exception {
                            return Observable.fromIterable(plan.getActionList());
                        }
                    })
                    //这里观察者泛型 必须和上面返回的数据的类型一样，否则报错
                    .subscribe(new Observer<String>() {
                        @Override
                        public void onSubscribe(Disposable d) {

                        }

                        @Override
                        public void onNext(String s) {
                            Log.e(TAG, "onNext: "+s );
                        }

                        @Override
                        public void onError(Throwable e) {

                        }
                        @Override
                        public void onComplete() {

                        }
                    });
```

结果如下：

```java
2019-07-01 15:38:08.110 19565-19565/com.admin.frame E/IndexDelegate: onNext: 哈哈哈
2019-07-01 15:38:08.110 19565-19565/com.admin.frame E/IndexDelegate: onNext: 嘻嘻嘻
```

#### concatMap

concatMap 和 flatMap 基本是一样的，只不过concatMap 转发出来的事件时有序的 ，而 flatMap 是无序的。

### 过滤操作符

过滤操作符用于过滤和小鸟则 Observable 发射的数据序列，让 Observable 只返回满足我们条件的数据

#### filter

filter 操作符是对源 Observable 产生的结果进行有规则的过滤，只有满足规则的结果才会提交到观察者手中。 

```java
   Observable.just(1, 2, 3, 4)
                    .filter(new Predicate<Integer>() {
                        @Override
                        public boolean test(Integer integer) throws Exception {
                            return integer < 3;
                        }
                    })
                    .subscribe(new Observer<Integer>() {
                        @Override
                        public void onSubscribe(Disposable d) {

                        }

                        @Override
                        public void onNext(Integer integer) {
                            Log.e(TAG, "onNext: " + integer);
                        }

                        @Override
                        public void onError(Throwable e) {

                        }

                        @Override
                        public void onComplete() {

                        }
                    });
```

结果如下

```java
2019-07-01 15:51:23.872 19845-19845/com.admin.frame E/IndexDelegate: onNext: 1
2019-07-01 15:51:23.872 19845-19845/com.admin.frame E/IndexDelegate: onNext: 2
```

#### distinct

他使用来去重的，他只允许还没有发射的数据通过，发射过的数据直接 pass。

```java
Observable.just(1, 3, 4, 5, 6, 76, 2, 3, 2, 1)
                    .distinct()
                    .subscribe(new Observer<Integer>() {
                        @Override
                        public void onSubscribe(Disposable d) {

                        }

                        @Override
                        public void onNext(Integer integer) {
                            Log.e(TAG, "onNext: " + integer);
                        }

                        @Override
                        public void onError(Throwable e) {

                        }

                        @Override
                        public void onComplete() {

                        }
                    });
```

结果如下：

```java
2019-07-01 15:54:50.513 20077-20077/com.admin.frame E/IndexDelegate: onNext: 1
2019-07-01 15:54:50.513 20077-20077/com.admin.frame E/IndexDelegate: onNext: 3
2019-07-01 15:54:50.513 20077-20077/com.admin.frame E/IndexDelegate: onNext: 4
2019-07-01 15:54:50.514 20077-20077/com.admin.frame E/IndexDelegate: onNext: 5
2019-07-01 15:54:50.514 20077-20077/com.admin.frame E/IndexDelegate: onNext: 6
2019-07-01 15:54:50.514 20077-20077/com.admin.frame E/IndexDelegate: onNext: 76
2019-07-01 15:54:50.514 20077-20077/com.admin.frame E/IndexDelegate: onNext: 2
```

​	他将重复的数据全部保留了一个。

#### buffer

缓存，把源Observable 转换成一个新的 Observable ，这个新的 Observable 每次发射的都是一组List，而不是一个个的的发送数据源。

如下所示：

```java
button.setOnClickListener((view) -> {
            Observable.just(1,2,3,4,5,6)
                    .buffer(2)
                    .subscribe(new Observer<List<Integer>>() {
                        @Override
                        public void onSubscribe(Disposable d) {

                        }
                        @Override
                        public void onNext(List<Integer> integers) {
                            for (Integer ints :
                                    integers) {
                                Log.e(TAG, "onNext: "+ints );
                            }
                            Log.e(TAG, "onNext: --------------------------->" );
                        }

                        @Override
                        public void onError(Throwable e) {

                        }

                        @Override
                        public void onComplete() {

                        }
                    });
```

结果如下：

```java
2019-07-01 16:00:57.485 20274-20274/com.admin.frame E/IndexDelegate: onNext: 1
2019-07-01 16:00:57.485 20274-20274/com.admin.frame E/IndexDelegate: onNext: 2
2019-07-01 16:00:57.485 20274-20274/com.admin.frame E/IndexDelegate: onNext: --------------------------->
2019-07-01 16:00:57.485 20274-20274/com.admin.frame E/IndexDelegate: onNext: 3
2019-07-01 16:00:57.485 20274-20274/com.admin.frame E/IndexDelegate: onNext: 4
2019-07-01 16:00:57.485 20274-20274/com.admin.frame E/IndexDelegate: onNext: --------------------------->
2019-07-01 16:00:57.485 20274-20274/com.admin.frame E/IndexDelegate: onNext: 5
2019-07-01 16:00:57.485 20274-20274/com.admin.frame E/IndexDelegate: onNext: 6
2019-07-01 16:00:57.485 20274-20274/com.admin.frame E/IndexDelegate: onNext: --------------------------->
```

#### kaip，take

skip 操作符将源Observable 发射过的数据过滤掉前 n 项，而 take 操作则只取前 n 项，另外还有 skipLasat 和 takeLast 则是从后往前进行过滤。

先看一下 skip 操作符

```java
Observable.just(1, 2, 3, 4, 5, 6)
                    .skip(2)
                    .subscribe(new Observer<Integer>() {
                        @Override
                        public void onSubscribe(Disposable d) {

                        }

                        @Override
                        public void onNext(Integer integer) {
                            Log.e(TAG, "onNext: " + integer);
                        }

                        @Override
                        public void onError(Throwable e) {

                        }

                        @Override
                        public void onComplete() {

                        }
                    });
```

结果如下：

```java
2019-07-01 16:06:31.843 20389-20389/com.admin.frame E/IndexDelegate: onNext: 3
2019-07-01 16:06:31.843 20389-20389/com.admin.frame E/IndexDelegate: onNext: 4
2019-07-01 16:06:31.843 20389-20389/com.admin.frame E/IndexDelegate: onNext: 5
2019-07-01 16:06:31.843 20389-20389/com.admin.frame E/IndexDelegate: onNext: 6
```

take 操作符

```java
Observable.just(1, 2, 3, 4, 5, 6)
                    .take(2)
                    .subscribe(new Observer<Integer>() {
                        @Override
                        public void onSubscribe(Disposable d) {

                        }

                        @Override
                        public void onNext(Integer integer) {
                            Log.e(TAG, "onNext: " + integer);
                        }

                        @Override
                        public void onError(Throwable e) {

                        }

                        @Override
                        public void onComplete() {

                        }
                    });
```

```java
2019-07-01 16:08:07.022 20616-20616/com.admin.frame E/IndexDelegate: onNext: 1
2019-07-01 16:08:07.022 20616-20616/com.admin.frame E/IndexDelegate: onNext: 2
```

### 组合操作符

#### merge

merge 是将多个操作符 合并到一个 Observable 中进行发射，merge 可能让合并到Observable 的数据发送错乱(并行无序)，

```java
Observable<Integer> observable1 = Observable.just(1, 2, 3, 4, 6);
            Observable<Integer> observable2 = Observable.just(34, 54);
            Observable.merge(observable1, observable2).subscribe(new Observer<Integer>() {
                @Override
                public void onSubscribe(Disposable d) {

                }

                @Override
                public void onNext(Integer integer) {
                    Log.e(TAG, "onNext: " + integer);
                }

                @Override
                public void onError(Throwable e) {

                }

                @Override
                public void onComplete() {

                }
            });
```

结果如下：

```java
2019-07-01 16:12:34.267 20757-20757/com.admin.frame E/IndexDelegate: onNext: 1
2019-07-01 16:12:34.267 20757-20757/com.admin.frame E/IndexDelegate: onNext: 2
2019-07-01 16:12:34.267 20757-20757/com.admin.frame E/IndexDelegate: onNext: 3
2019-07-01 16:12:34.267 20757-20757/com.admin.frame E/IndexDelegate: onNext: 4
2019-07-01 16:12:34.267 20757-20757/com.admin.frame E/IndexDelegate: onNext: 6
2019-07-01 16:12:34.267 20757-20757/com.admin.frame E/IndexDelegate: onNext: 34
2019-07-01 16:12:34.267 20757-20757/com.admin.frame E/IndexDelegate: onNext: 54
```

#### concat

将多个Observable发射的数据进行合并并且发射，和merge不同的是，merge是无序的，而concat是有序的。（串行有序）没有发射完前一个它一定不会发送后一个

#### concatEager

前面说的串行有序，而concatEager则是并行且有序。我们来看看如果修改：

```java
Observable<Integer> observable1 = Observable.just(1, 2, 3, 4);
            Observable<String> observable2 = Observable.just("a", "b", "c", "d");
            Observable.concatEager(Observable.fromArray(observable1, observable2))
                    .subscribe(new Observer<Serializable>() {
                        @Override
                        public void onSubscribe(Disposable d) {

                        }

                        @Override
                        public void onNext(Serializable serializable) {
                            Log.e(TAG, "onNext: " + serializable);
                        }

                        @Override
                        public void onError(Throwable e) {

                        }

                        @Override
                        public void onComplete() {

                        }
                    });
```

```java
2019-07-01 16:27:10.659 21416-21416/com.admin.frame E/IndexDelegate: onNext: 1
2019-07-01 16:27:10.659 21416-21416/com.admin.frame E/IndexDelegate: onNext: 2
2019-07-01 16:27:10.659 21416-21416/com.admin.frame E/IndexDelegate: onNext: 3
2019-07-01 16:27:10.659 21416-21416/com.admin.frame E/IndexDelegate: onNext: 4
2019-07-01 16:27:10.659 21416-21416/com.admin.frame E/IndexDelegate: onNext: a
2019-07-01 16:27:10.659 21416-21416/com.admin.frame E/IndexDelegate: onNext: b
2019-07-01 16:27:10.659 21416-21416/com.admin.frame E/IndexDelegate: onNext: c
2019-07-01 16:27:10.659 21416-21416/com.admin.frame E/IndexDelegate: onNext: d
```

#### zip

此操作符和合并多个Observable发送的数据项，根据他们的类型就行重新变换，并发射一个新的值。

```java
Observable<Integer> observable1 = Observable.just(1, 2, 3, 4);
            Observable<String> observable2 = Observable.just("a", "b", "c","d");

            Observable.zip(observable1, observable2, new BiFunction<Integer, String, String>() {
                @Override
                public String apply(Integer integer, String s) throws Exception {
                    return integer + s;
                }
            })
                    .subscribe(new Observer<String>() {
                        @Override
                        public void onSubscribe(Disposable d) {

                        }

                        @Override
                        public void onNext(String s) {
                            Log.e(TAG, "onNext: "+s );
                        }

                        @Override
                        public void onError(Throwable e) {

                        }

                        @Override
                        public void onComplete() {

                        }
                    });
```

结果如下：

```java
2019-07-01 16:21:15.470 21292-21292/com.admin.frame E/IndexDelegate: onNext: 1a
2019-07-01 16:21:15.470 21292-21292/com.admin.frame E/IndexDelegate: onNext: 2b
2019-07-01 16:21:15.470 21292-21292/com.admin.frame E/IndexDelegate: onNext: 3c
2019-07-01 16:21:15.470 21292-21292/com.admin.frame E/IndexDelegate: onNext: 4d
```

### 线程控制

​	线程切换的用法：

```java
			   .observeOn(Schedulers.io())
                .subscribeOn(AndroidSchedulers.mainThread())	
```

- subscribeOn是指定之前的事件流处理的线程，设置的第一个有效，后面的效。
- observeOn是指定之后的事件流处理的线程，设置的最后一个有效，前面的无序。

并且提供了四种类型线程模式

AndroidSchedulers.mainThread（）：主线程

Schedulers.io（）：io操作的线程，通常用于网络，读写文件等io密集型的操作

Schedulers.computation（）：CPU计算密集型的操作

Schedulers.newThread（）：新建一个线程

例子

```java
 Observable.create(new ObservableOnSubscribe<Integer>() {
                @Override
                public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                    emitter.onNext(2);
                    Log.e(TAG, "subscribe: " + Thread.currentThread().getName());
                }
            })
            .subscribeOn(Schedulers.io()) //指定 subscribe 发生在 io 线程      --第一个有效
            .subscribeOn(AndroidSchedulers.mainThread()) //指定 subscribe 发生在 主 线程       --无效

            .observeOn(Schedulers.io()) //指定下面的发生在 IO 线程        --无效
            .observeOn(AndroidSchedulers.mainThread()) //指定下面的发生在 主线程       --最后一个有效
            .map(new Function<Integer, String>() {
                @Override
                public String apply(Integer integer) throws Exception {
                      og.e(TAG, "apply: " + Thread.currentThread().getName());
                      return integer + "";
                  }
             })
             .observeOn(Schedulers.io())//指定下面的发生在 子线程
             .observeOn(AndroidSchedulers.mainThread())//指定下面的发生在 主线程 --最后一个有效
             .subscribe(new Observer<String>() {
             	@Override
                 public void onSubscribe(Disposable d) {

                 }

                @Override
                public void onNext(String s) {
                      Log.e(TAG, "onNext: " + s + "   " + Thread.currentThread().getName());
                }

                @Override
                public void onError(Throwable e) {

                }

                Override
                public void onComplete() {

                }
           });
```

结果如下所示：

```java
2019-07-01 18:06:56.808 23754-23818/com.admin.frame E/IndexDelegate: subscribe: RxCachedThreadScheduler-1
2019-07-01 18:06:56.809 23754-23754/com.admin.frame E/IndexDelegate: apply: main
2019-07-01 18:06:56.824 23754-23754/com.admin.frame E/IndexDelegate: onNext: 2   main
```

修改上面的代码如下所示：

将subsrcible 中的 方法加上 log

```java
.subscribe(new Observer<String>() {
                        @Override
                        public void onSubscribe(Disposable d) {
                            Log.e(TAG, "onSubscribe: " + Thread.currentThread().getName());
                        }

                        @Override
                        public void onNext(String s) {
                            Log.e(TAG, "onNext: " + s + "   " + Thread.currentThread().getName());
                        }

                        @Override
                        public void onError(Throwable e) {
                            Log.e(TAG, "onError: " + Thread.currentThread().getName());
                        }

                        @Override
                        public void onComplete() {
                            Log.e(TAG, "onComplete: " + Thread.currentThread().getName());
                        }
                    });
```

运行后结果如下

```java
2019-07-01 18:10:59.873 23932-23932/com.admin.frame E/IndexDelegate: onSubscribe: main
2019-07-01 18:10:59.885 23932-23985/com.admin.frame E/IndexDelegate: subscribe: RxCachedThreadScheduler-1
2019-07-01 18:10:59.888 23932-23932/com.admin.frame E/IndexDelegate: apply: main
2019-07-01 18:10:59.890 23932-23932/com.admin.frame E/IndexDelegate: onNext: 2   main
```

**可以看到 onSubscribe 这个方法是第一个执行，然后才是重上往下依次执行。**

### RxJava 配合 Retrofit 使用

```java
 String url = "http://192.168.167.2:8090/Frame_Api/";
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(url)
                .addConverterFactory(ScalarsConverterFactory.create()) //转换器
                //使用 RxJava
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build();
```

创建接口

```java
public interface MyService {
    @GET("index.php")
    Call<Student> getTop(@Query("name") String name, @Query("age") int age);

    @POST("index.php")
    Observable<String> post(@Body RequestBody body);
}
```

网络请求

```java
Observable<String> observable = myService.post(body);
            observable
                    .subscribeOn(Schedulers.io())
                    .flatMap((Function<String, ObservableSource<List<String>>>) s -> {
                        List<String> list = new ArrayList<>();
                        list.add(s);
                        return Observable.just(list);
                    })
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(new Observer<List<String>>() {
                        @Override
                        public void onSubscribe(Disposable d) {
                        }

                        @Override
                        public void onNext(List<String> strings) {
                            Log.e(TAG, "onNext: " + strings.get(0) + "   " + Thread.currentThread().getName());
                        }

                        @Override
                        public void onError(Throwable e) {
                            Log.e(TAG, "onNext: " + e + "   " + Thread.currentThread().getName());
                            e.printStackTrace();
                        }

                        @Override
                        public void onComplete() {

                        }
                    });
```

