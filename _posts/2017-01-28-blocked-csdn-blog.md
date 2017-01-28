---
layout: post
title: "被CSDN误判的博文，到底哪个是敏感信息啊？"
date: 2017-01-28 23:13 +0800
categories: CSDN
tags:
---

被CSDN误判的博文，全文原封不动贴在下面，到底哪个是敏感信息啊？

----------------------  分割线  ----------------------

**Android中bindService的细节之三：多次调用bindService()，为什么onBind()只执行一次**

<!--more-->

# **0. 场景**

为了更方便的说明问题，而又不失共性，本文中考虑下面两种情况：

* **情况一：** App A绑定App B的service，App A多次调用bindService()，而不调用unbindService()，此时App B的service的onBind()只执行一次

* **情况二：** App A，App C绑定App B的service，App A和App C各调用一次或多次bindService()，而不调用unbindService()，此时App B的service的onBind()只执行一次

上面提到的两种情况有2个共同点：

* （1）每次调用bindService()时，绑定的服务是一样的；
* （2）没有调用unbindService()

例如，下面的示例代码，设置intent中的包名和service名，是通过：
```java
intent.setClassName("com.galian.app_b", "com.galian.app_b.MyService")
```
com.galian.app_b是App B的包名；
com.galian.app_b.MyService是App B提供的Service。
这样App A绑定的服务和App C绑定的服务就是一样的了。

App A绑定App B的MyService的代码，如下：

```java
    Intent intent = new Intent();
    intent.setClassName("com.galian.app_b", "com.galian.app_b.MyService");
    intent.putExtra("KEY_STRING", "From AppA!");
    intent.putExtra("KEY_INT", 42);
    Uri uri = Uri.parse("content://com.galian.app_a/table");
    intent.putExtra("KEY_PARCEABLE", uri);
    bindService(intent, mSvcConn, Context.BIND_AUTO_CREATE);
```

App C绑定App B的MyService的代码，如下：

```java
    Intent intent = new Intent();
    intent.setClassName("com.galian.app_b", "com.galian.app_b.MyService");
    intent.putExtra("KEY_STRING", "From AppC!");
    intent.putExtra("KEY_FLOAT", 42.0);
    Uri uri = Uri.parse("content://com.galian.app_c/table");
    intent.putExtra("KEY_PARCEABLE", uri);
    bindService(intent, mSvcConn, Context.BIND_AUTO_CREATE);
```

App A 调用bindService()之后，等ServiceConnection的onServiceConnected()执行完，切换到App C，App C再调用bindService()，这时com.galian.app_b.MyService的onBind()只执行了一次。

**注意：** 实际上可以有其他的方式可以让onBind()调用多次，本文中只讨论onBind()只执行一次的原因。

上面的场景中，为什么onBind()只执行一次？

关于bindService的基本流程，可以参考链接：

[《Android中bindService的细节之一：从进程的角度分析绑定Service的流程【Service所在进程首次启动】》](http://blog.csdn.net/u013553529/article/details/54698123)

[《Android中bindService的细节之二：从进程的角度分析绑定Service的流程【Service所在进程已存在】》](http://blog.csdn.net/u013553529/article/details/54698123)

本文中分析这种情况：App B进程之前已经启动。因为这样可以方便对比App A绑定服务和App C绑定服务的区别。否则，如果App B进程之前不存在，则App A绑定服务和App C绑定服务的流程首先会有App B进程创建流程的差别（关于这个差别，可以参考上面2个链接）。而这不是本文讨论的重点。

**再次说明本文分析的前提：App B进程已经存在，但是App B的服务MyService之前没有被绑定过。**

#  **1. 关键的类：IntentBindRecord**

```java
final class IntentBindRecord {
    /** The running service. */
    final ServiceRecord service;
    /** The intent that is bound.*/
    final Intent.FilterComparison intent;
    /** All apps that have bound to this Intent. */
    final ArrayMap<ProcessRecord, AppBindRecord> apps
            = new ArrayMap<ProcessRecord, AppBindRecord>();
    /** Binder published from service. */
    IBinder binder;
    /** Set when we have initiated a request for this binder. */
    boolean requested;
    /** Set when we have received the requested binder. */
    boolean received;
    /** Set when we still need to tell the service all clients are unbound. */
    boolean hasBound;
    /** Set when the service's onUnbind() has asked to be told about new clients. */
    boolean doRebind;
    ...
    ...
}
```

IntentBindRecord是一个关键的类，它记录着以下信息：

* （1）ServiceRecord service： 被绑定的服务的ServiceRecord；

* （2）Intent.FilterComparison intent：绑定服务时的intent。绑定服务时，将bindService()的intent与IntentBindRecord中的intent进行比较，如果服务已经绑定（received为true），且intent能匹配上（FilterComparison的equals()返回true），则直接返回binder（IntentBindRecord中的IBinder binder），这些内容后面会详细讲。

* （3）ArrayMap&lt;ProcessRecord, AppBindRecord&gt; apps：使用相同Intent.FilterComparison intent的所有apps

* （4）IBinder binder：这是第一次绑定服务成功后，onBind()返回来的IBinder，通过publishService()传过来的。

* （5）boolean requested：标志位，requested为true表示绑定服务的请求已经发出了，是与`r.app.thread.scheduleBindService()`对应的（r为ServiceRecord实例），即调用`r.app.thread.scheduleBindService()`之后，通常要将requested置为true，以避免重复调用`r.app.thread.scheduleBindService()`。
如果doRebind为true，调用`r.app.thread.scheduleBindService()`之后，不会将requested置为true。

* （6）boolean received：标志位，received为true表示App B已经将onBind()返回的binder传给了AMS。是与IBinder binder同时起作用的。

* （7）boolean hasBound：表示是否有client绑定此服务。这里的client具体来说就是某一个Activity。

* （8）boolean doRebind：Service的`onUnbind()`的返回值为true时，将通过AMS的`unbindFinished()`传给AMS，记录在doRebind标志位中。下一次再有client绑定服务时，将根据doRebind的值来决定是调用onBind()，还是调用onRebind()。具体代码见：
handleBindService() @ ActivityThread.java
```java
    private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            ...
            if (!data.rebind) {
            	// 如果之前onUnbind()返回false，则执行onBind()。（如果不重写onUnbind()，默认返回false）
                IBinder binder = s.onBind(data.intent);
				...
            } else {
            	// 如果之前onUnbind()返回true，则执行onRebind()。
                s.onRebind(data.intent);
				...
            }
            ...
        }
    }
```

如果对这个类的成员变量还是模糊，也不要紧，后面还会在具体流程中提到。

# **2. App A绑定App B的服务MyService**

**执行流程大致是这样的：**

```
bindService() @ ContextImpl
-> bindService() @ AMS
-> bindServiceLocked() @ ActiveServices
-> retrieveServiceLocked() @ ActiveServices
根据intent获取service组件信息，返回service对应的ServiceRecord

-> AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
参数service是intent。这是本文讨论的重点。后面详细说。

-> bringUpServiceLocked()
	if (r.app != null && r.app.thread != null) {
        sendServiceArgsLocked(r, execInFg, false);
        return null;
    }
需要注意这段代码，r为ServiceRecord实例。r.app为ProcessRecord实例，r.app.thread为实现IApplicationThread接口的实例。
r.app是在执行realStartServiceLocked()时赋值的，所以此时r.app是null。


-> realStartServiceLocked()
-> app.thread.scheduleCreateService()
此处会出现并行处理的情形，即AMS进程继续执行，同时App B进程处理CREATE_SERVICE消息，即handleCreateService()。

-> requestServiceBindingsLocked()
-> requestServiceBindingLocked()
-> r.app.thread.scheduleBindService()
此处会出现并行处理的情形，即AMS进程继续执行，同时App B进程处理BIND_SERVICE消息，即handleBindService()。需要注意的是，BIND_SERVICE消息是在CREATE_SERVICE消息处理完之后才能处理，因为这两个消息放入了消息队列，队列先入先出。所以handleBindService()的执行是在handleCreateService()之后。

-> i.requested = true;
i.hasBound = true;  // 已经有client绑定服务，所以hasBound置为true
i.doRebind = false;  // 默认为false
i为IntentBindRecord实例，在此情景中，执行完r.app.thread.scheduleBindService()，就把requested置为true，这个requested值将在后面的流程中判断。

-> 回到调用bringUpServiceLocked()方法的地方（在bindServiceLocked()中），继续执行bringUpServiceLocked()之后的代码

-> 执行下面的代码 （在bindServiceLocked()中）
	if (s.app != null && b.intent.received) {
        try {
            c.conn.connected(s.name, b.intent.binder);
        }...
        ...
    } else if (!b.intent.requested) {
        requestServiceBindingLocked(s, b.intent, callerFg, false);
    }
这段代码也是本文分析的重点，留待后面详细说明。

-> handleCreateService() @ ActivityThread
handleCreateService()运行在App B进程中，是与AMS进程并行执行的，但是从代码的执行来看，handleCreateService()不会很快执行完，所以放在这里来理解也没有问题。

-> handleBindService() @ ActivityThread
运行在App B进程中。肯定是在handleCreateService()执行完才执行的。
这也是本文分析的重点。后面会详细分析。

```

## **2.1 重点代码1：retrieveAppBindingLocked()**

retrieveAppBindingLocked() @ ServiceRecord

```
    public AppBindRecord retrieveAppBindingLocked(Intent intent,
            ProcessRecord app) {
        Intent.FilterComparison filter = new Intent.FilterComparison(intent);
        IntentBindRecord i = bindings.get(filter);
        if (i == null) {
        	// 执行这里
            i = new IntentBindRecord(this, filter);
            bindings.put(filter, i);
        }
        AppBindRecord a = i.apps.get(app);
        if (a != null) {
            return a;
        }
        a = new AppBindRecord(this, i, app);
        i.apps.put(app, a);
        return a;
    }
```
retrieveAppBindingLocked()根据intent来查询服务是否已经被绑定了，对应的代码是`IntentBindRecord i = bindings.get(filter)`。

在本文讨论的场景中App A是第一个绑定Myservice的，所以`bindings.get(filter)`的结果为null，需要创建新的IntentBindRecord。

需要说明的是，`bindings.get(filter)`，get的过程中，需要判断bindings中的intent与filter这个intent是否相同，这种intent的比较都是由Intent.FilterComparison类来完成的。
Intent类本身并没有重写`equals()`方法，所以不能用来判断intent是否相同。

Intent.FilterComparison重写了`equals()`方法，如下：
equals() @ FilterComparison @ Intent

```
    @Override
    public boolean equals(Object obj) {
        if (obj instanceof FilterComparison) {
            Intent other = ((FilterComparison)obj).mIntent;
            return mIntent.filterEquals(other);
        }
        return false;
    }
```

filterEquals() @ Intent

```java
    public boolean filterEquals(Intent other) {
        if (other == null) {
            return false;
        }
        // 比较Intent中的各项指标：Action，Uri，MIME type，包名，Component，Category
        if (!Objects.equals(this.mAction, other.mAction)) return false;
        if (!Objects.equals(this.mData, other.mData)) return false;
        if (!Objects.equals(this.mType, other.mType)) return false;
        if (!Objects.equals(this.mPackage, other.mPackage)) return false;
        if (!Objects.equals(this.mComponent, other.mComponent)) return false;
        if (!Objects.equals(this.mCategories, other.mCategories)) return false;

        return true;
    }
```

## **2.2 重点代码2：对IntentBindRecord中的标志位进行判断**


```java
	if (s.app != null && b.intent.received) {
        try {
            c.conn.connected(s.name, b.intent.binder);
        }...
        ...
    } else if (!b.intent.requested) {
	// 不执行这里
        requestServiceBindingLocked(s, b.intent, callerFg, false);
    }
```
s为ServiceRecord实例。s.app为ProcessRecord实例
b为AppBindRecord实例，
b.intent为IntentBindRecord实例。
这里对IntentBindRecord实例中的received和requested进行判断。

b.intent.received的赋值：只有在service的onBind()执行完，执行publishService()之后才会将received置为true。这是还没有执行到publishService()，所以received为false。

从前面的流程可以看到，b.intent.requested已经被置为true。

所以，对于App A绑定App B的服务的场景中，上面的代码什么也没做。
啥都没做，咋是重点代码呢？为了与App C绑定服务的情况进行对比。因为App C也会执行这段代码，而且是要执行c.conn.connected()的。


## **2.3 重点代码3：handleBindService()**

```java
    private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        if (s != null) {
            try {
            	...
                try {
                    if (!data.rebind) {
                    	// 执行这里
                        // data.token是service对应的ServiceRecord
                        IBinder binder = s.onBind(data.intent);
                        ActivityManagerNative.getDefault().publishService(
                                data.token, data.intent, binder);
                    } else {
                    	// 在此场景中，不执行这里
                        s.onRebind(data.intent);
                        ...
                    }
                } catch ...
            } catch ...
        }
    }
```

执行onBind()返回binder对象，然后将此binder对象通过publishService()传给AMS。

publishService() @ AMS

```java
    public void publishService(IBinder token, Intent intent, IBinder service) {
        synchronized(this) {
            mServices.publishServiceLocked((ServiceRecord)token, intent, service);
        }
    }
```

publishServiceLocked() @ ActiveServices

```java
    void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
    	/* Binder.clearCallingIdentity()的目的是:
        	切换进程后，要提升执行代码的权限，将此时执行代码的权限提升为AMS，
            否则还是调用方App B的权限，这里的权限是指uid、pid对应的权限。
        */
        final long origId = Binder.clearCallingIdentity();
        try {
            if (r != null) {
                Intent.FilterComparison filter
                        = new Intent.FilterComparison(intent);
                // r.bindings.get(filter)返回不为null，因为之前创建了IntentBindRecord
                // b.received初始值是false
                IntentBindRecord b = r.bindings.get(filter);
                if (b != null && !b.received) {
                	// 执行这里
                    /*更新IntentBindRecord中的数据，b.binder更新为onBind()返回的binder对象；b.requested更新为true，表示绑定请求已经处理了；b.received更新为true，表示binder对象已经收到了，目的是下次不用再调用onBind()了，这在App C绑定服务时会介绍。*/
                    b.binder = service;
                    b.requested = true;
                    b.received = true;
                    for (int conni=r.connections.size()-1; conni>=0; conni--) {
                        ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                        for (int i=0; i<clist.size(); i++) {
                            ConnectionRecord c = clist.get(i);
                            if (!filter.equals(c.binding.intent.intent)) {
                                continue;
                            }
                            try {
                            	// 执行这里，最终会调用到App A的onServiceConnected()
                                c.conn.connected(r.name, service);
                            } catch ...
                        }
                    }
                }

                serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

至此，App A绑定App B的MyService的流程就介绍完了。
接下来，App C要绑定App B的MyService了。

# **3. App C绑定App B的服务MyService**

**注意**，此时App B的服务MyService已经被App A绑定过了。

执行流程大致是这样的：

```java
bindService() @ ContextImpl
-> bindService() @ AMS
-> bindServiceLocked() @ ActiveServices
-> retrieveServiceLocked() @ ActiveServices
这里与App A绑定服务时有区别。App A执行时，将创建的ServiceRecord实例放到了ServiceMap的mServicesByName中，mServicesByName是ArrayMap<ComponentName, ServiceRecord>类型的。
App C执行时，根据intent中的Component可以直接从mServicesByName中获取ServiceRecord。

-> AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
参数service是intent。
执行IntentBindRecord i = bindings.get(filter);获取到App A绑定服务时创建的IntentBindRecord实例，存入AppBindRecord中，返回给b。

-> bringUpServiceLocked()
	if (r.app != null && r.app.thread != null) {
        sendServiceArgsLocked(r, execInFg, false);
        return null;
    }
r为ServiceRecord实例。r.app为ProcessRecord实例。
这里的r是通过retrieveServiceLocked()得到的，是之前App A绑定服务时创建的ServiceRecord实例，之前已经将r.app赋值了。
所以，此处会执行return null，不会执行后面的realStartServiceLocked()。

-> 回到调用bringUpServiceLocked()方法的地方（在bindServiceLocked()中），继续执行bringUpServiceLocked()之后的代码

-> 执行下面的代码 （在bindServiceLocked()中）
	if (s.app != null && b.intent.received) {
        try {
        	// 执行这里
            c.conn.connected(s.name, b.intent.binder);
        }...
        ...
    } else if (!b.intent.requested) {
    	// 不执行这里
        requestServiceBindingLocked(s, b.intent, callerFg, false);
    }
s为ServiceRecord实例，s.app不为null。
b.intent为IntentBindRecord实例，在之前执行publishService()时，将IntentBindRecord的received标志和binder赋值了。
所以，b.intent.received为true，b.intent.binder为有效值，是onBind()返回的binder对象。c.conn.connected(s.name, b.intent.binder)将会执行到。
c.conn.connected()最终会调用到App C的onServiceConnected()。
```

至此，App C绑定App B的MyService的流程结束。
可以看到App C绑定服务的流程中，没有再调用MyService的onBind()。

# **4. 总结**

简略概括一下App A和App C绑定App B的MyService的流程，如下：

第一次App A绑定Service时，创建了IntentBindRecord实例，并记录在AppBindRecord中。
执行onBind()后，App B进程通过publishService()将binder传给了AMS，记录在IntentBindRecord实例中，并设置标志位received。

第二次App C绑定Service时，获取之前的IntentBindRecord实例，判断标志位received为true，则直接调用App C的onServiceConnected()将binder对象传给App C。
所以第二次没有调用onBind()。



**继续阅读**

[《Android中bindService的细节之一：从进程的角度分析绑定Service的流程【Service所在进程首次启动】》](http://blog.csdn.net/u013553529/article/details/54698123)

[《Android中bindService的细节之二：从进程的角度分析绑定Service的流程【Service所在进程已存在】》](http://blog.csdn.net/u013553529/article/details/54698123)
