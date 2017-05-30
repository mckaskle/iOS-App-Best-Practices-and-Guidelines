# iOS App Best Practices and Guidelines

## Table of Contents
1. [Threading](#Threading)

## Threading

> Do everything on the main thread. Don’t even think about queues and background threads. Enjoy paradise!
> 
> If, after testing and profiling, you find you do have to move some things to a background queue, pick things that can be perfectly isolated, and make sure they’re perfectly isolated. Use delegates; do not use KVO or notifications.
> 
> If, in the end, you still need to do some tricky things — like updating your model on a background queue — remember that the rest of your app is either running on the main thread or is little isolated things that you don’t need to think about while writing this tricky code. Then: be careful, and don’t be optimistic. (Optimists write crashes.)

Source: [inessential.com: How Not To Crash 4: Threading](http://inessential.com/2015/05/22/how_not_to_crash_4_threading)

If callback needs to be called after some background work is complete, the callback should be dispatched to the main thread. There may exist a scenario where it makes sense to call that callback on another queue. Those rare cases may be handled in two ways:

1. Provide a `queue: DispatchQueue` parameter so the task can dispatch it to the appropriate queue. In this way, the caller always knows which queue the callback is going to be called on.
2. Add a warning to the documentation that explicitly calls out that the callback will not be called on a main thread. Example ([source][warning-source]):

       /// A method that does a thing and calls the handler on completion.
       ///
       /// - Warning:
       /// The handler will be called on a background queue.
       func task(handler: @escaping () -> Void) { … }

[warning-source]: https://developer.apple.com/library/content/documentation/Xcode/Reference/xcode_markup_formatting_ref/Important.html#//apple_ref/doc/uid/TP40016497-CH37-SW1


### Suggested Reading

- [How Not to Crash #4: Threading](http://inessential.com/2015/05/22/how_not_to_crash_4_threading)
- [How Not to Crash #5: Threading, part 2](http://inessential.com/2015/05/26/how_not_to_crash_5_threading_part_2)


## Error Handling

Always proprogate errors up to site that triggered the overarching task. Don't mask the error within the task where the failure occurred.

> Without error propagation:
> 
> - the user-interface may get stuck in a mid-task state
> - earlier stages in the task can’t attempt a retry or recover
> - we’re forcing the task’s presentation of the error, rather than giving the user interface controller that triggered the action a the chance to present errors in a more integrated way
> 
> …
> 
> Swift’s `throws` keyword is one of the best features of the language. You may need to paint a lot of functions with the `throws` attribute (especially if you need to `throw` from deep inside a hierarchy) but it makes your interfaces honest.

Source: [Presenting unanticipated errors to users](https://www.cocoawithlove.com/blog/2016/04/14/error-recovery-attempter.html)

Following this practice not only gives you the option to handle all errors, but it forces you to consider all errors at the UI level. Presenting the error takes more thought and effort than merely logging the error, but the result is a better UX. It is usually good enough to merely show an alert for edge case errors, but errors that are likely (like network calls failing) should be discussed with product/design.


### Note On Error Logging

It is best to log the errors at the site that triggered the overarching task. If you log errors deeper in the stack where the error occurred, it can get complex. For instance, say we try to log the error where it occurred in this example:

```
func doAThing(url: URL) throws {
  do {
    try FileManager.default.removeItem(at: url)
    try FileManager.default.createDirectory(at: url)
  } catch {
    log_error(error) // Assuming we have this global function for logging errors.
    throw error // Rethrow the error so it continue to propogate.
  }
}
```

Here, the assumption is that since this lower level code is logging the error, the caller doesn't need to log the error. That seems fine if this assumption is understood throughout the codebase, at least until the code needs to be modified:

```
func doAThing(url: URL) throws {
  do {
    try FileManager.default.removeItem(at: url)
    try doAnotherThing()
    try FileManager.default.createDirectory(at: url)
  } catch {
    log_error(error) // Assuming we have this global function for logging errors.
    throw error // Rethrow the error so it continue to propogate.
  }
}
```

Now we are potentially logging the error twice if `doAnotherThing()` is the code that triggered the error to be thrown and it too is logging it's own error. A recursive function that can throw an error has the same issue. It keeps the code straightforward if the only logging code is at the topmost level. If the error is going to continue propogating, don't log it.

This is true for errors that are passed into a result handler as well:

```
func doSomething(handler: (Error?) -> Void) { … }
```

It would be up to the caller to log the error:

```
doSomething { error in
  if let error = error {
    log_error(error)
  }
  
  // … continue with the rest of the code
}
```



### Suggested Reading

- [Presenting unanticipated errors to users](https://www.cocoawithlove.com/blog/2016/04/14/error-recovery-attempter.html)


## Handling Server Responses

> … _whoever wrote the server side is your sworn enemy_. He or she hates you very, very much.

Source: [How Not to Crash #7: Dealing with Nothing](http://inessential.com/2015/05/29/how_not_to_crash_7_dealing_with_nothin)

Even if the server API is well documented and there is an understood contract in place, never make the assumption that the server code isn't going to throw in a curve ball (even if the author is really a nice person who loves you very much). Check the type of every single piece of data; ensure all data that you expect to be there is actually there. This is much more natural in Swift then it was in Objective-C. Objective-C's `id` type means that you never have to consider this if you don't want to, but you definitely should.

If there is a piece of data that is not as you expect, then generally speaking the best course of action is to consider the whole task a failure. The callback that is expecting the result of the network request should receive an error. Don't try to cover up the issue with a default value or an empty list. The server is not abiding by the contract set forth in the API and this is an error that needs to be propogated up.


## Working with JSON in Swift

> Converting between representations of the same data in order to communicate between different systems is a tedious, albeit necessary, task for writing software.
> 
> Because the structure of these representations can be quite similar, it may be tempting to create a higher-level abstraction to automatically map between these different representations. For instance, a type might define a mapping between snake_case JSON keys and camelCase property names in order to automatically initialize a model from JSON using the Swift reflection APIs, such as Mirror.
> 
> However, we’ve found that these kinds of abstractions tend not to offer significant benefits over conventional usage of Swift language features, and instead make it more difficult to debug problems or handle edge cases. […] The cost of small amounts of duplication may be significantly less than picking the incorrect abstraction.

Source: [Apple Swift Blog: Working with JSON in Swift](https://developer.apple.com/swift/blog/?id=37)

Reflection and third party libraries that abstract serialization or deserialization of JSON should be avoided.

Is is preferable to throw an error when JSON deserialization fails, as opposed to returning `nil`. See the "Writing a JSON Initializer with Error Handling" section in the previously mentioned [Working with JSON in Swift article](https://developer.apple.com/swift/blog/?id=37) for a good example of handling errors.


## View Controller Flow

According to the [Apple documentation][dismiss-documentation]:

> The presenting view controller is responsible for dismissing the view controller it presented.

Unfortunately, UIKit makes it easy to ignore this directive and have the presented view controller dismiss itself. The next line in the documentation reads:

>  If you call this method on the presented view controller itself, UIKit asks the presenting view controller to handle the dismissal.

In fact, several UIKit-provided view controllers dismiss themselves. However, that is going to lead to tight coupling of view controllers and **should be avoided**.

Let's say there are two view controllers `A` and `B`, where `A` presents `B`. If `B` dismisses itself, then it implicitly knows that it was presented. This tight coupling will lead to unexpected results when a new view controller `C` actually pushes `B` onto a navigation stack instead of presenting it.

Instead, set up a delegate for `B`, so that `B` can let its delegate know when it has finished what it was doing. View controllers `A` and `C` can then be set as the delegate and `A` can dismiss the view controller (calling dismiss on itself, rather than on the `B` view controller) and `C` can pop the navigation stack.


### Note on Popping the Navigation Stack

It is better to use `navigationController?.popToViewController(self, animated: true)` ([documentation][pop-documentation]) rather than `navigationController?.popViewController(animated: true)` so that you can be sure you are popping to the correct view controller. Otherwise, the view controller that did the popping needs to have to have knowledge on how the pushed view controller handles things. It's possible the pushed view controller has pushed view controllers onto the stack too. 

Note that the `dismiss` method already handles a stack in this way. According to the [documentation][dismiss-documentation]:

> If you present several view controllers in succession, thus building a stack of presented view controllers, calling this method on a view controller lower in the stack dismisses its immediate child view controller and all view controllers above that child on the stack. When this happens, only the top-most view is dismissed in an animated fashion; any intermediate view controllers are simply removed from the stack.

[dismiss-documentation]: https://developer.apple.com/reference/uikit/uiviewcontroller/1621505-dismiss
[pop-documentation]: https://developer.apple.com/reference/uikit/uinavigationcontroller/1621871-poptoviewcontroller


### Suggested Reading

The following is a series of posts by Soroush Khanlou on Coordinators. Coordinators are his solution for the tight coupling between view controllers and view controller bloat. Personally, I have tried coordinators out and found that they aren't for me. I'm okay with view controllers doing some of the things he isn't okay with and I find that creating a separate "Coordinator" object doesn't remove coupling, it just moves it to another place. But these articles are worth considering. It was these articles that inspired the view controller flow that is described above.

- [The Coordinator](http://khanlou.com/2015/01/the-coordinator/)
- [Coordinators Redux](http://khanlou.com/2015/10/coordinators-redux/)
- [Series of posts on Advanced Coordinators](http://khanlou.com/tag/advanced-coordinators/) (think of this as extra credit; the basics have already been covered; only read if you are still interested)


## Dependency Injection

Stack Overflow has some good answers on [What is dependency injection and when/why should or shouldn't it be used?](https://stackoverflow.com/q/130794). There are basically two major benefits to DI:

1. [Testability](https://stackoverflow.com/a/140655)
2. [Loose Coupling](https://stackoverflow.com/a/6085922)

Here are a few places that can get overlooked in iOS development when it comes to dependency injection. This list is not exhaustive:

- API clients
- User defaults / user settings
- Logged in user data
- Data model (Core Data persistent container/managed object context)

It will get burdensome to individually inject these items into every object that needs them. The solution is to bundle them all up in a single object that holds a reference to each item. I have seen this object called an [`Environment`](http://news.realm.io/news/try-swift-brandon-williams-writing-testable-code/#code-testing-with-co-effects). I call it a `ScreenContext`. Here is example:

```
class ScreenContext {
  init(apiClient: MyAPIClient, persistentContainer: NSPersistentContainer) {
    self.apiClient = apiClient
    self.persistentContainer = persistentContainer
  }
  
  let apiClient: MyAPIClient
  let persistentContainer: NSPersistentContainer
}
```

All screens might need `ScreenContext`, but say there is a group of screens that are behind a log-in wall. All of those screens will require a `LoggedInUser`. If there are enough screens behind this wall, it makes sense to create another context object, similar to ScreenContext except that includes the user:

```
class LoggedInScreenContext: ScreenContext {
  init(loggedInUser: LoggedInUser, apiClient: MyAPIClient, persistentContainer: NSPersistentContainer) {
    self.loggedInUser = loggedInUser
    super.init(apiClient: apiClient, persistentContainer: persistentContainer)
  }

  let loggedInUser: LoggedInUser
}
```

Don't feel obligated to make a separate screen context for every dependency that needs to be injected. An object should only make it into a screen context if it is used throughout the app or a large component in the app.

Note: subclasses might not always be the best pattern. [Protocol composition][protocol-composition-di] will make more sense in some situations. Subclassing is simpler, but not as flexible. Judge for yourself dependent on the project and then keep with that pattern throughout the project. For new projects, subclasses probably make the most sense. If it makes sense to move to protocol composition, then convert all existing screen contexts away from subclasses to composition.

[protocol-composition-di]: http://merowing.info/2017/04/using-protocol-compositon-for-dependency-injection/


## Access Control

The rule of thumb is to make properties and methods `private` by default and classes are `final` by default. Only open up control as needed. This makes for easier-to-maintain code and can even make the code more performant and compile faster (though this is likely negligible and not the reason for this guideline).


## Localization

Always localize user-facing text. Use `NSLocalizedString` and the `Formatter` subclasses from the beginning. Code should not be merged into the mainline development branch until all user-facing text is localized.


## Nitpicky Stuff

### Style Guide

Follow [The Official raywenderlich.com Swift Style Guide](https://github.com/raywenderlich/swift-style-guide#code-organization). If there are pieces of this document that contradict with the raywenderlich.com Swift Style Guide, then go with this document.

As for code organization, it helps scanning a file for the code you are looking for if the code is organized in a consistent manner across all files. The actual order of things isn't really important. Consistency is the important part. Here is the structure that I typically follow for `classes`/`structs`:

- Enums
- Type Aliases
- Object Lifecycle (`init`, `deinit`, and `awakeFromNib` methods; sometimes a `configure` method if and only if it's used in multiple `init` methods)
- Public Properties
- Public Methods
- Internal Properties
- Internal Methods
- Private Properties
- Actions (these are the `dynamic`/`@IBAction` methods that are triggered by buttons, timers, etc. and they are always private)
- Private Methods
- Subclass overrides (these are broken up by the exact type that defined the original properties/methods that are overridden; for example, a `UITableViewController` subclass might have a `UIViewController` section for `override func viewDidLoad() { … }` and a `UITableViewControllerDataSource` section for `override func numberOfSections(in: UITableView) -> Int { … }`)

Each section should be denoted with a `MARK` comment. For example:

```
final class SubOperation: Operation {

  // MARK: - Object Lifecycle

  init(screenContext: ScreenContext) {
    self.screenContext = screenContext
  }


  // MARK: - Private Properties

  private let screenContext: ScreenContext


  // MARK: - Operation

  override func main() {
    super.main()

    // Do stuff here…
  }

}
```


### Directory Structure

The Xcode Project navigator structure should match the underlying directory structure as much as possible. It helps if the paths of the Groups in the project navigator point to their corresponding directories within the repo.


### No Warnings

If you let the project continue with even just 1 warning, you will start to become blind to the warnings panel that Xcode gives you. No code should be merged to the mainline development branch with any warnings.


## More Suggested Readings

- [How Not To Crash series](http://inessential.com/hownottocrash)

