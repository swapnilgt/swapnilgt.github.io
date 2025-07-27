---
layout:	post
title:	"Handle exceptions like a pro with Coroutines"
date:	2023-04-16
---

---

Handling exceptions in any mobile app is very critical. Improper handling of exceptions leads to the following major problems:

* Dead ends in the user flow.
* Application crashes.
* Lack of observability on the health of the app.

As soon as the code base grows in size, it becomes incrementally tough to handle exceptions. Additionally, if the app starts growing to a large user base, it becomes really critical to identify the correct and high-priority issues dwelling in the app with a low turnaround time.

Leaving **dead ends in the code** leads to users’ frustration resulting in user dropout and bad reviews. Figuring and fixing these dead ends also become expensive with the time.

**Application crashes** though easily discoverable have a high cost with user dropouts and bad reviews.

> How Google Play Store [calculates rating](https://learn.apptentive.com/knowledge-base/google-play-store-ratings-reporting/).

> How Apple Store [calculates rating](https://blog.apptopia.com/app-store-rank-explained-what-is-an-apps-apple-google-store-ranking-and-what-impacts-it).

Given that the positioning of a mobile app on app stores depends directly on the rating of the store, developers should strive to manage exceptions perfectly from the very start.

### What are dead ends and how to manage them?

Before we see how to manage dead ends, let’s first try to understand a dead end with an example.

```
suspend fun submitData(userData: UserData) {  
        try {  
            userDao.save(userData)  
        } catch (e: Exception) {  
            Log.e(TAG, "Exception in saving the data locally")  
        }  
    }
```

The code snippet shown above is one of the most common mistakes that developers tend to make when handling an exception. Even, if developers log the exception to a remote exception logger system like [Firebase Crashlytics](https://firebase.google.com/docs/crashlytics/customize-crash-reports?platform=android#log-excepts), [Sentry](https://forum.sentry.io/t/non-fatal-reporting/11191), etc...

```
suspend fun submitData(userData: UserData) {  
        try {  
            userDao.save(userData)  
        } catch (e: Exception) {  
            Log.e(TAG, "Exception in saving the data locally")  
            // Logging exception to remote logging system.  
            FirebaseCrashlytics.getInstance().recordException(e)  
        }  
    }
```

There are two drawbacks to the above approach:

1. The reporting framework object needs to be injected at different locations.
2. The devs need to take care of exception handling across different layers in their code.

### Coroutines at rescue

With [Coroutines](https://www.google.com/search?q=kotlin+coroutines&oq=kotlin+coroutines&aqs=chrome..69i57j69i59l2j69i60l2j69i65.2776j0j9&sourceid=chrome&ie=UTF-8), Kotlin enables developers to handle exceptions in a very convenient way. Here is a gist of exception handling in Koltin:

> Coroutine builders come in two flavors: **propagating exceptions** automatically ([launch](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) and [actor](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html)) or exposing them to users ([async](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) and [produce](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html)). When these builders are used to create a root coroutine, that is not a child of another coroutine, the former builders treat exceptions as **uncaught** exceptions, similar to Java’s `Thread.uncaughtExceptionHandler`, while the latter are relying on the user to consume the final exception, for example via [await](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/await.html) or [receive](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html) ([produce](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html) and [receive](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html) are covered in [Channels](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/channels.md) section).

Exception propagation makes life really easy and enables developers to avoid any exception leakages.

Usually, there are two types of coroutine use cases in Android.

* Coroutines triggered and bound to the scope of Activity/Fragment or ViewModel.
* Coroutines triggered in a global scope and can execute beyond the lifecycle of the containing component (Activity/Fragment)

#### Coroutines scoped to Activity/Fragments/ViewModel

The usual flow, in this case, is the following:

> Activity/Fragment → ViewModel → Usecase → Repository (Network/Local cache)

During this execution, if there is any exception at one of the above layers, the exception is propagated to the scope which might be [LifecycleCoroutineScope](https://developer.android.com/reference/kotlin/androidx/lifecycle/LifecycleCoroutineScope) or [ViewModelScope](https://developer.android.com/kotlin/coroutines/coroutines-best-practices#viewmodel-coroutines). We can capture the exception originating from any layer and handle it elegantly.

```
private fun renderPopup() {  
    viewLifecycleOwner.lifecycleScope.launch {  
        try {  
            .......  
        } catch (e: Exception) {  
            FirebaseCrashlytics.getInstance().recordException(e)  
            Timber.e(e, "Exception in running the coroutine")  
        }  
    }  
}
```

#### Coroutines scoped globally

There are cases when you might want to have a coroutine execution run beyond the scope of an Activity/Fragment/ViewModel. In such cases, a scope can be defined and can be injected inside your code for usage. You can define a global scope like:

```
fun provideCoroutineScope(): CoroutineScope {  
    return CoroutineScope(Dispatchers.IO + SupervisorJob())  
}
```

If we trigger any coroutine from the above scope and there are any exceptions originating in the downstream code, the application will crash. We definitely wouldn’t want to be in that state. To solve this, we can hook a [CoroutineExceptionHandler](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/) in the custom scope.

```
fun provideCoroutineScope(): CoroutineScope {  
    return CoroutineScope(Dispatchers.IO + SupervisorJob()  
            + CoroutineExceptionHandler { _, throwable ->  
        Timber.e(throwable, "Exception inside my Coroutine execution")  
        FirebaseCrashlytics.getInstance().recordException(throwable)  
    })  
}
```

> A detailed explanation of the internals of the same is mentioned here: <https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c>

Having a [CoroutineExceptionHandler](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/) hooked to the custom scope helps us in avoiding crashes for the app and record the exception cases.

### Conclusion

We have demonstrated how coroutines enable developers to handle exceptions elegantly in the code. This helps us with:

* Avoid **dead ends** in the code.
* Avoid **application crashes.**
* Discovering the **exceptions** users are witnessing inside the app.

I hope this article helped in understanding the power of exception handling through coroutines. Do follow me on my social handles to get updates on my posts:

* [Linkedin](https://www.linkedin.com/in/swapnil-gupta-6968a824/)
* [Twitter](https://twitter.com/swapnilgupta20)