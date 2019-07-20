---
layout: post
title:  "App Container"
---

When your app launches, the app delegate will create your window, but where does the code belong that decides which screen you should see first? Should you show login, or your apps main contents that require login? UIKit doesn't provide any out of the box solutions to this problem.

## Existing Solutions

The two most common solutions to this problem include presenting your login flow from the apps root view controller or changing your main windows root view controller as state changes.

Here’s what the code might look like to present your login flow from your app's root view controller.

```swift
final class RootTabBarController: UITabBarController {
    var isLoggedIn: Bool {
        // Check if we have a user or auth token cached somewhere
    }
    ...

    func viewDidAppear() {
        super.viewDidAppear()

        if !isLoggedIn {
            presentLoginFlow()
        }
    }

    func presentLoginFlow() {
        // Create our login flow and present it
    }
}
```
Cons:
* Requires `RootTabBarController` to have to know about the high-level state, the user, and have the responsibility of presenting view controllers that it shouldn't need to know about.
* Requires `RootTabBarController` to "refresh" and fetch data when a user logs in.
* Has to listen for logout events and reset state/clear state for the nested view controllers in the tab bar.
* Provides a jarring user experience when the user sees your login flow presenting over top of your apps root view controller.

Changing our windows root view controller is equally gross.

```swift
final class AppDelegate: UIResponder, UIApplicationDelegate {
    var window: UIWindow?

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        window = UIWindow()

        if isLoggedIn {
            window.rootViewController = makeRootTabBarController()
        } else {
            window.rootViewController = makeLoginFlow()
        }

        window?.makeKeyAndVisible()
        
        return true
    }

    func makeRootTabBarController() -> UIViewController {
        // Create tab bar and nested view controllers
    }

    func makeLoginFlow() -> UIViewController {
        // Create login flow
    }
}
```
Cons:
* Logic that shouldn't exist in our app delegate
* It's not easy to animate between root view controllers
* Our app delegate has to observe state and switch between the root tab bar and the login flow as state changes

## View Controller Containment

Around iOS 5, Apple introduced APIs that allow us to nest child view controllers inside of parent view controllers called view controller containment. Many of the core UIViewControllers we use every day are implemented using view controller containment include UINavigationController, UITabBarController, and UISplitViewController.

Let’s take a look at how we might implement our root view controller using view controller containment. A view controller can have many child view controllers, but in this case, we’re going to show one child view controller at a time — either our login flow or our apps root tab bar.

We’ll start with our apps new root view controller and show it in our window. Then we’ll add code to our AppFlowViewController that can cycle between child view controllers. Finally, we'll refactor and clean things up.

## Showing our initial view controller

Lets create our new root view controller: `FlowContainerViewController`. `FlowContainerViewController` will add our login flow or tab bar controller as a child view controller depending on the logged-in state of the user. 

```swift
import UIKit

final class FlowContainerViewController: UIViewController {
    init(initialChildViewController: UIViewController) {
        super.init(nibName: nil, bundle: nil)

        add(initialChild: initialChildViewController)
    }

    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    private func add(initialChild viewController: UIViewController) {
        addChild(viewController)
        viewController.view.frame = self.view.bounds
        view.addSubview(viewController.view)
        viewController.didMove(toParent: self)
    }
}
```
I'm not using storyboards for this example, as a result, I opted to `fatalError` for the `init?(coder:)` implementation. I'm not going to go in depth on a storyboard implementation of this pattern, but you would have to provide a real impementation of `init?(coder:)`, and add the initial child view controller in `awakeFromNib` or `viewDidLoad` as you won't be able to pass an initial view controller into the initializer.

We also provide a custom initializer and private function to embed our first child view controller.

Next, we'll create an instance of `FlowContainerViewController` in our app delegate and set its initial view controller to our splash screen. While managing the splash screen ourselves might seem odd, we will gain the ability to provide custom animations from our splash screen into the app. An excellent example of this would be Twitter's splash screen animation when you launch the app.

```swift
import UIKit

@UIApplicationMain
final class AppDelegate: UIResponder, UIApplicationDelegate {
    private var flowContainerViewController: FlowContainerViewController?
    var window: UIWindow?

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        window = UIWindow()
        let splashViewController = UIStoryboard(name: "LaunchScreen", bundle: nil).instantiateInitialViewController()!
        flowContainerViewController = FlowContainerViewController(initialChildViewController: splashViewController)
        window?.rootViewController = flowContainerViewController        
        window?.makeKeyAndVisible()
        
        return true
    }
}
```

If we have any UI setup in our splash screen storyboard, we should see it persist and stay on screen if we were to run the app. If you’re starting a new project with an empty splash screen an excellent way to test things out would be to add a label with an emoji to the center of your splash screen.

## Showing the first view controller after our splash screen

Next, we’ll add a function to our flow container that lets us switch between different child view controllers with a simple fade animation.

```swift
...

final class FlowContainerViewController: UIViewController {
    ...

    func cycle(from oldViewController: UIViewController, to newViewController: UIViewController) {
        oldViewController.willMove(toParent: nil)
        addChild(newViewController)

        newViewController.view.alpha = 0

        transition(from: oldViewController, to: newViewController, duration: 0.2, options: [], animations: {
            newViewController.view.alpha = 1
        }) { (finished) in
            oldViewController.removeFromParent()
            newViewController.didMove(toParent: self)
        }
    }

    ...
}
```
`transition(from:,to:,duration:,options:,animation:,completion:) ` is provided by Apple for this specific use case and adds/removes our child view controller’s view to and from the view hierarchy for us. We still have to call all of the other view controller containment methods to add our new child view controller and to remove our old view controller. 

We’re also going to add a convenience function that only takes our `newViewController`, and pulls the `oldViewController` out of our array of child view controllers.

```swift
...

final class FlowContainerViewController: UIViewController {
    ...

    func cycle(to newViewController: UIViewController) {
        guard let oldViewController = children.first else {
            assertionFailure("Failed to get first child view controller.")
            return
        }
        cycle(from: oldViewController, to: newViewController)
    }

    ...
}
```

The assertion failure here is purely to warn developers if they’re doing something weird in debug builds like mutating `FlowContainerViewController`’s child outside of these cycle functions. This code will not crash in a release configured build, but would early return and not perform the transition between child view controllers.

Now we're ready to add code to our app delegate to cycle between child view controllers based on whether the user is logged in or not. 

```swift
@UIApplicationMain
final class AppDelegate: UIResponder, UIApplicationDelegate {
    ...

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        ...
        
        if UserStore().currentUser == nil {
            self.flowContainerViewController?.cycle(to: LoginViewController())
        } else {
            self.flowContainerViewController?.cycle(to: makeAppRootViewController())
        }
        
        return true
    }

    func makeAppRootViewController() -> UIViewController {
        let feedViewController = KittyFeedViewController()
        let profileViewController = ProfileViewController()
        let tabBarViewController = UITabBarController()
        tabBarViewController.viewControllers = [feedViewController, profileViewController]
        return tabBarViewController
    }
}
```

In the above code, we use a class called UserStore to check if we have a currentUser. If we don’t, we show the login screen, and if we do, then we show the main feed. In your app, this is where you would implement business logic to check for a current user, or the existence of an auth token to decide where to route users in your app. We’re going to refactor this in a bit because it doesn’t belong in the app delegate. 

## Observing state and cycling between children

Finally, we need to transition the view controller inside of our app container view controller when the user logs in and out. We’ll use closures for this in the example, but in your app, you could use notifications, delegates, or closures.

```swift
@UIApplicationMain
final class AppDelegate: UIResponder, UIApplicationDelegate {
    ...

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        ...
        
        if UserStore().currentUser == nil {
            self.flowContainerViewController?.cycle(to: LoginViewController(loginHandler: handleLogin))
        } else {
            self.flowContainerViewController?.cycle(to: makeAppRootViewController())
        }
        
        ...
    }
    
    private func makeAppRootViewController() -> UIViewController {
        ...
        let profileViewController = ProfileViewController(logoutHandler: handleLogout)
        ...
    }
    
    private func handleLogin() {
        self.flowContainerViewController?.cycle(to: makeAppRootViewController())
    }
    
    private func handleLogout() {
        self.flowContainerViewController?.cycle(to: LoginViewController(loginHandler: handleLogin))
    }
}
```

We pass the `handleLogin` function to the `LoginViewController` to be called when the user has successfully logged in, and the `handleLogout` function to our `ProfileViewController` which displays a logout button that will call `handleLogout` when tapped.

We can now refactor this code further as well because we now have duplicated code in our if statement, and in our `handleLogin` and `handleLogout` functions.

```swift
@UIApplicationMain
final class AppDelegate: UIResponder, UIApplicationDelegate {
    ...

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        ...
        
        if UserStore().currentUser == nil {
            handleLogout()
        } else {
            handleLogin()
        }
        
        ...
    }
    
    private func makeAppRootViewController() -> UIViewController {
        ...
        let profileViewController = ProfileViewController(logoutHandler: handleLogout)
        ...
    }
    
    private func handleLogin() {
        self.flowContainerViewController?.cycle(to: makeAppRootViewController())
    }
    
    private func handleLogout() {
        self.flowContainerViewController?.cycle(to: LoginViewController(loginHandler: handleLogin))
    }
}
```

## Refactoring out AppFlowCoordinator

Now for our biggest refactor. We’re going to take everything out of AppDelegate that doesn’t belong there and move it out to an object called AppFlowCoordinator. AppFlowCoordinator is responsible for listening to events and controlling which child view controller exists in the `FlowContainerViewController`. 

```swift
final class AppFlowCoordinator {
    private var flowContainerViewController: FlowContainerViewController?
    
    func start(with window: UIWindow) {
        let splashViewController = UIStoryboard(name: "LaunchScreen", bundle: nil).instantiateInitialViewController()!
        flowContainerViewController = FlowContainerViewController(initialChildViewController: splashViewController)
        window.rootViewController = flowContainerViewController
        
        if UserStore().currentUser == nil {
            handleLogout()
        } else {
            handleLogin()
        }
    }
    
    private func makeAppRootViewController() -> UIViewController {
        let feedViewController = KittyFeedViewController()
        let profileViewController = ProfileViewController(logoutHandler: handleLogout)
        let tabBarViewController = UITabBarController()
        tabBarViewController.viewControllers = [feedViewController, profileViewController]
        return tabBarViewController
    }
    
    private func handleLogin() {
        self.flowContainerViewController?.cycle(to: makeAppRootViewController())
    }
    
    private func handleLogout() {
        self.flowContainerViewController?.cycle(to: LoginViewController(loginHandler: handleLogin))
    }
}
```

```swift
@UIApplicationMain
final class AppDelegate: UIResponder, UIApplicationDelegate {
    var window: UIWindow?
    private let flowCoordinator = AppFlowCoordinator()

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        let appWindow = UIWindow()
        window = appWindow
        appWindow.makeKeyAndVisible()
        flowCoordinator.start(with: appWindow)
        
        return true
    }
}
```

## Conclusion

Now we have a simple AppDelegate and a class that exists explicitly to manage our high-level app state and change what the users see based on their authentication status.

Check out the full project on [Github](https://github.com/pmick/app-container-example).