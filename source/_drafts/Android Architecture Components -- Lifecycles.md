# Android架构组件(二) --Lifecycles

## 简介

APP中的某一些组件是跟Activity，Fragment等这些组件的生命周期息息相关的，为了实现这些组件和Activity等的生命周期的联动，一般需要把一些方法写到生命周期方法中。但是这些这么写代码不是太优雅，很容易处理不好，比如忘了解注册造成内存泄漏，加额外的bool变量来控制组件行为。这类组件一多，会造成Activity等的生命周期内代码很混乱不好维护。Lifecycle组件就是为了解决这种问题的。

android.arch.lifecycle包中提供的类和接口可以帮助我们来更好的维护代码，通过注解，可以让我们自己的组件随着Activity生命周期联动，而无需在Activity生命周期方法中自己写代码维护我们自己组件的生命周期。

## 引入Lifecycle

引用架构组件需要在项目的maven仓库中加入google，

```java
allprojects {
    repositories {
        jcenter()
        google()
    }
}
```

由于Lifecycle经常和LiveData，ViewModel一起使用，所以Google提供了很多组合引入的方法，当然也可以单独引入。如果使用1.x的版本，可以使用下面的方法引入。

```groovy
dependencies {
    def lifecycle_version = "1.1.1"

    // ViewModel and LiveData
    implementation "android.arch.lifecycle:extensions:$lifecycle_version"
    // alternatively - just ViewModel
    implementation "android.arch.lifecycle:viewmodel:$lifecycle_version" // use -ktx for Kotlin
    // alternatively - just LiveData
    implementation "android.arch.lifecycle:livedata:$lifecycle_version"
    // alternatively - Lifecycles only (no ViewModel or LiveData).
    //     Support library depends on this lightweight import
    implementation "android.arch.lifecycle:runtime:$lifecycle_version"

    annotationProcessor "android.arch.lifecycle:compiler:$lifecycle_version"
    // alternately - if using Java8, use the following instead of compiler
    implementation "android.arch.lifecycle:common-java8:$lifecycle_version"

    // optional - ReactiveStreams support for LiveData
    implementation "android.arch.lifecycle:reactivestreams:$lifecycle_version"

    // optional - Test helpers for LiveData
    testImplementation "android.arch.core:core-testing:$lifecycle_version"
}
```

上一篇中提到了，在2018年5月的时候，Google对架构组件进行了一次重构，它成为了AndroidX项目的一部分，因此在架构组件的2.x版本（截止到2018年6月，androidx还没有发布正式版）中，引用的包名发生了改变。

```groovy
dependencies {
    def lifecycle_version = "2.0.0-alpha1"

    // ViewModel and LiveData
    implementation "androidx.lifecycle:lifecycle-extensions:$lifecycle_version"
    // alternatively - just ViewModel
    implementation "androidx.lifecycle:lifecycle-viewmodel:$lifecycle_version" // use -ktx for Kotlin
    // alternatively - just LiveData
    implementation "androidx.lifecycle:lifecycle-livedata:$lifecycle_version"
    // alternatively - Lifecycles only (no ViewModel or LiveData). Some UI
    //     AndroidX libraries use this lightweight import for Lifecycle
    implementation "androidx.lifecycle:lifecycle-runtime:$lifecycle_version"

    annotationProcessor "androidx.lifecycle:lifecycle-compiler:$lifecycle_version"
    // alternately - if using Java8, use the following instead of lifecycle-compiler
    implementation "androidx.lifecycle:lifecycle-common-java8:$lifecycle_version"

    // optional - ReactiveStreams support for LiveData
    implementation "androidx.lifecycle:lifecycle-reactivestreams:$lifecycle_version" // use -ktx for Kotlin

    // optional - Test helpers for LiveData
    testImplementation "androidx.arch.core:core-testing:$lifecycle_version"
}
```

值得注意的是，如果使用了appcompat-v7:27+版本的包，同时会引入lifecycle:runtime包，引入fragment（或者是v4包），会引入LiveData和ViewModel.

```groovy
+--- com.android.support:appcompat-v7:27.1.1
|    +--- com.android.support:support-annotations:27.1.1
|    +--- com.android.support:support-core-utils:27.1.1
|    |    +--- com.android.support:support-annotations:27.1.1
|    |    \--- com.android.support:support-compat:27.1.1
|    |         +--- com.android.support:support-annotations:27.1.1
|    |         \--- android.arch.lifecycle:runtime:1.1.0
|    |              +--- android.arch.lifecycle:common:1.1.0
|    |              \--- android.arch.core:common:1.1.0
|    +--- com.android.support:support-fragment:27.1.1
|    |    +--- com.android.support:support-compat:27.1.1 (*)
|    |    +--- com.android.support:support-core-ui:27.1.1
|    |    |    +--- com.android.support:support-annotations:27.1.1
|    |    |    +--- com.android.support:support-compat:27.1.1 (*)
|    |    |    \--- com.android.support:support-core-utils:27.1.1 (*)
|    |    +--- com.android.support:support-core-utils:27.1.1 (*)
|    |    +--- com.android.support:support-annotations:27.1.1
|    |    +--- android.arch.lifecycle:livedata-core:1.1.0
|    |    |    +--- android.arch.lifecycle:common:1.1.0
|    |    |    +--- android.arch.core:common:1.1.0
|    |    |    \--- android.arch.core:runtime:1.1.0
|    |    |         \--- android.arch.core:common:1.1.0
|    |    \--- android.arch.lifecycle:viewmodel:1.1.0
```

## Lifecycle组件的使用

### 处理与组件生命周期密切相关的类

Android中的四大组件是Android系统的核心元素，应用的运行都必须依赖四大组件进行，四大组件的生命周期是由OS和Android Framework控制，我们的代码都是围绕四大组件的生命周期进行的。如果我们的代码管理不好，没有随着生命周期方法进行适时的资源释放或其他操作，很容易造成内存泄漏等不可预料的后果。

在介绍Lifecycle的使用之前，先看一段代码，通常情况下我们的代码是怎样在Activity的生命周期中写的。比如有一个定位的回调，需要在各个生命周期方法中调用对应的方法。

```java
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_google_map);
        mEmulatedLocationListener = new EmulatedLocationListener(this, new LocationChangeCallback() {
            @Override
            public void onLocationChang(Location location) {
//                do nothing
            }
        });
        mEmulatedLocationListener.init();
    }

    @Override
    protected void onStart() {
        super.onStart();
        mEmulatedLocationListener.start();
    }

    @Override
    protected void onResume() {
        super.onResume();
    }

    @Override
    protected void onStop() {
        super.onStop();
        mEmulatedLocationListener.stop();

    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        mEmulatedLocationListener.destroy();
    }
```

只有定位的时候，这段代码看起来似乎还好，没有太复杂，但是，当一个Activity中需要类似的组件太多的时候，Activity的生命周期方法看起来就会十分的臃肿，不太方便维护。Lifecycle的引入就是为了解决这个问题。

### 使用Lifecycle

上面的代码中，EmulatedLocationListener的多个方法都需要在Activity的生命周期中手动调用，不好维护。使用Lifecycle组件之后，就可以通过下面的方法来使EmulatedLocationListener自动观察Activity的生命周期。

```java
public class EmulatedLocationListenerV2 implements LifecycleObserver {

    private static String TAG = EmulatedLocationListenerV2.class.getCanonicalName();

    private Context mContext;
    private LocationChangeCallback mLocationChangeCallback;
    private boolean mEnabled = false;

    public EmulatedLocationListenerV2(Context mContext, Lifecycle lifecycle, LocationChangeCallback mLocationChangeCallback) {
        this.mContext = mContext;
        this.mLocationChangeCallback = mLocationChangeCallback;
        lifecycle.addObserver(this);
    }

    void enable() {
        this.mEnabled = true;
    }

    void disable() {
        this.mEnabled = false;
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    void init() {
        Log.v(TAG, "init");
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    void stop() {
        Log.v(TAG, "stop");
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    void start() {
        Log.v(TAG, "start");
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    void destroy() {
        Log.v(TAG, "destroy");
    }
}

```
此时，在Activity中只需要实例化EmulatedLocationListener即可，不需要在各个生命周期方法中手动调用方法，代码相对来说，简洁了很多。

```java

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_google_map);
    mEmulatedLocationListener = new EmulatedLocationListenerV2(this, this.getLifecycle(), new LocationChangeCallback() {
        @Override
        public void onLocationChang(Location location) {
//           do nothing
        }
    });
}

```

### 几个重要的类

#### LifeCycle

Lifecyle类是一个抽象类，它持有着Activity等这类组件的生命周期状态，允许其他的组件观察到生命周期状态。

Lifecycle中使用类2个枚举类来跟踪生命周期的状态。

- Event

  生命周期事件由Framework或者Lifecycle类发出，枚举的值和生命周期的回调方法一一对应。
  
- State

 State则标识了当前组件的状态(和Fragment源码中的集中状态类似)。
 
它们的对应关系如图所示：
 
 ![](./img/lifecycle-states.PNG)
 
#### LifecycleOwner 
 
LifecycleOwner是一个接口，只有一个方法（Java 8中这种接口可以叫函数式接口），它表明一个类有生命周期。
 
```java
 public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}

```

26.1.0之后的Spupprt library中，SupportActivity，v4包中的Fragment都实现了LifecyclerOwner接口，封装好了生命周期的分发。LifecycleService也实现了LifecycleOwner接口。

#### LifecycleObserver

LifecycleObserver是一个接口，没有任何方法，它的存在是为了标识某个类可以观察组件（Activity等）的生命周期。

当需要一个类观察组件(Activity等)的生命周期时，只需要实现这个接口，然后把自己添加到LifeCycleOwner的观察者队列中。在这个类中对方法增加注解，就能在对应的生命周期方法中自动调用该注解方法。

```java
//step 1

class MyObserver implements LifecycleObserver {

	@OnLifecycleEvent(Lifecycle.Event.ON_START)
	void method() {
		//do something
	}

}

//step 2
class Activity extends FragmentActivity {

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		...
		MyObserver obs = new Observer();
		this.getLifecycle().addObserver(obs);
	}
}
```

#### LifecycleRegistry

LifecycleRegistry是Lifecycle接口的一个具体实现，Support Library中的Fragmen和Activity都使用了这个类，getLifecycle()方法也是返回的这个类中的实例。如果想自己实现LifecycleOwner，也可以通过这个类来实现生命周期状态的分发。

可以简答看一下这个类中主要的属性和方法。

```java
public class LifecycleRegistry extends Lifecycle {

	/**
	 * 一个Map，支持在遍历的时候编辑元素，它保存了LifecycleObserver和它的状态
	 */
	private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap = new FastSafeIterableMap<>();
	
	/**
     * 组件当前的生命周期状态
     */
    private State mState;
    
    /**
     * 弱引用的LifecycleOwner，可以保证如果Lifecyle有泄漏，不会造成整个Activity和Fragment泄漏
     */
    private final WeakReference<LifecycleOwner> mLifecycleOwner;
	
	//...
    
    /**
     * 把Lifecycle移动到指定的状态，并把状态分发到给观察者
     */
    public void markState(@NonNull State state) {
        moveToState(state);
    }
    
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }

    private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            // we will figure out what to do on upper level.
            return;
        }
        mHandlingEvent = true;
        sync();
        mHandlingEvent = false;
    }
	
	/**
	 * 对Lifecycle做了一层包装，包含了Lifecycle当前的状态
	 */
	static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver; //实现了LifecycleObserver

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.getCallback(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
}
```
