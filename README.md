# Aaper

Annotated Android Permissions takes care of ensuring Android runtime permissions for an `EnsurePermissions`-annotated method inside either an Activity or a Fragment. The idea is to do so without having to write any code nor override any Activity and/or Fragment method related to runtime permissions.

#### Default behavior example

```kotlin
// Aaper initialization
class MyApplication {
    override fun onCreate() {
        super.onCreate()
        Aaper.init()
    }
}
```
```kotlin
// Aaper usage

class MyActivity/MyFragment {
    override fun onCreate/onViewCreated(...) {
        takePhotoButton.setOnClickListener {
            takePhoto()
        }
    }

    @EnsurePermissions(permissions = [Manifest.permission.CAMERA])
    fun takePhoto() {
        Toast.makeText(this, "Camera permission granted", Toast.LENGTH_SHORT).show()
    }
}
```
Just by adding the `EnsurePermissions` annotation to `takePhoto()`, what will happen (by default) when we run that code and click on the `takePhotoButton` button is:
- Aaper will check if your app has the CAMERA permission already granted.
- If already granted, Aaper will proceed to run `takePhoto()` right away and it will all end there.
- If NOT granted, Aaper will NOT run `takePhoto()` right away, and rather will proceed to run the default permission `RequestStrategy` which is to launch the System's permission dialog asking the user to grant the CAMERA permission.
- If the user approves it, Aaper will proceed to run `takePhoto()`.
- If the user denies it, the default behavior is just not running `takePhoto()`.

Changing the default behavior
---
Aaper's permission requests behavior is fully customizable, you can define what to do before and after a permission request is executed, and even how the request is executed, by creating your own `RequestStrategy` class. The way Aaper works is by delegating the request actions to a `RequestStrategy` instance, you can tell Aaper which strategy to use by:
- Specifying the strategy name on the @EnsurePermissions annotation.
- Defining your own `RequestStrategy` as default.

### Custom Strategy example
We'll create a custom strategy that will finish an activity in the case that at least one of the permissions requested by Aaper is denied.

We'll start by creating our class that extends from `ActivityRequestStrategy`:

```kotlin
class FinishActivityOnDeniedStrategy : ActivityRequestStrategy() {

    override fun onPermissionsRequestResults(
        host: Activity,
        data: PermissionsResult
    ): Boolean {
        TODO("Not yet implemented")
    }

    override fun getName(): String {
        TODO("Not yet implemented")
    }
}
```

There are three types of `RequestStrategy` base classes that we can choose from when creating our custom `RequestStrategy`, those are:
- `ActivityRequestStrategy` - Only supports EnsurePermissions-annotated methods inside Activities.
- `FragmentRequestStrategy` - Only supports EnsurePermissions-annotated methods inside Fragments.
- `AllRequestStrategy` -  Supports both Activities and Fragment classes' EnsurePermissions-annotated methods.


    All three have the same structure and same base methods, the main difference from an implementation point of view, would be the type of `host` provided in its base functions, for example in the method `onPermissionsRequestResults` we see that our host is of type `Activity`, because we extend from `ActivityRequestStrategy`, whereas if we extend from `FragmentRequestStrategy`, the host will be a `Fragment`. For `AllRequestStrategy`, the host is `Any` or `Object` and you'd have to check its type manually in order to verify whether the current request is for an Activity or a Fragment.

In this example, we want to close an Activity if at least one requested permission is denied, therefore `ActivityRequestStrategy` seems to suit better for this case.

We must provide for every custom `RequestStrategy` 2 things, a name (which will serve as an ID for our Strategy) and a boolean as response for `onPermissionsRequestResults` method, depending on what we return there, this is what will happen after a permission request is executed:
- If `onPermissionsRequestResults` retuns TRUE, it means that the request was successful in our Strategy and therefore the EnsurePermissions-annotated method will get executed.
- If `onPermissionsRequestResults` returns FALSE, it means that the request failed in our Strategy and therefore the EnsurePermissions-annotated method will NOT get executed.

For our example, this is what it will end up looking like:
```kotlin
class FinishActivityOnDeniedStrategy : ActivityRequestStrategy() {

    override fun onPermissionsRequestResults(
        host: Activity,
        data: PermissionsResult
    ): Boolean {
        if (data.denied.isNotEmpty()) {
            // At least one permission was denied.
            host.finish()
            return false // So that the annotated method doesn't get called.
        }

        // No permissions were denied, therefore proceed to call the annotated method.
        return true
    }

    override fun getName(): String {
        // We can return anything here, as long as there is no other Strategy with the same name.
        return "FinishActivityOnDenied"
    }
}
```

As we can see in `onPermissionsRequestResults`, we check the `denied` permissions list we get from `data`, and we verify if it is not empty, which would mean that there are some denied permissions, therefore our Strategy will treat the request process as failed and will return `false` so that the annotated method won't get called, and before that, we call `host.finish()`, in order to close our Activity too.

If the `denied` permissions list is empty, it means that all of the requested permissions were approved, therefore our Strategy will treat the request process as successful and will return `true` in order to proceed to call the annotated method.

### Using our custom Strategy
In order to use our new `FinishActivityOnDeniedStrategy` request strategy, we must first register it right after Aaper's initialization:

```kotlin
class MyApp : Application() {

    override fun onCreate() {
        super.onCreate()
        Aaper.init()
        val strategyProvider = Aaper.getRequestStrategyProvider() as DefaultRequestStrategyProvider
        strategyProvider.register(FinishActivityOnDeniedStrategy())
    }
}
```

We can register as many Strategies as we like, as long as they all have unique names. After registering our new `RequestStrategy`, we can either:
- **Use it on our annotation only.** This can be achieved by passing our strategy's name into the `EnsurePermissions` annotation of a method, like so:
    ```kotlin
    @EnsurePermissions(
        permissions = [(PERMISSION NAMES)],
        strategyName = "FinishActivityOnDenied"
    )
    ```
- **Or, Set it as the `default` strategy** for all the annotated methods by doing the following after registering our custom Strategy:
    ```kotlin
    // ...
    strategyProvider.register(FinishActivityOnDeniedStrategy())
    strategyProvider.setDefaultStrategyName("FinishActivityOnDenied")
    ```
    After doing so, you won't have to explicitly pass "FinishActivityOnDenied" to the `EnsurePermissions` annotation in order to use this custom one, as it will be the default one.

License
---
    MIT License

    Copyright (c) 2020 LikeTheSalad.

    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:

    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    SOFTWAR