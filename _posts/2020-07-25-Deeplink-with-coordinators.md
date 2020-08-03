---
title: "Deeplink with coordinators"
tags:
  - coordinators
  - deeplinks
  - combine
header:
  teaser: "assets/images/coordinator-teaser.jpg"
---
There are countless articles about **Coordinator** pattern and how great it is to manage deeplinking, but not many go in-depth on how to use them in real production apps.

{% raw %}
<img src="../../assets/images/coordinator-teaser.jpg" alt="">
{% endraw %}

## Coordinators

The coordinator pattern is wildly known and there are many articles about that. 
That's why I will not describe how to implement them.

*So why coordinators? You don't want to end up with something like this...*
```swift
guard let rootVC = AppDelegate.shared.window?.rootViewController else { return }
if viewController is UIAlertController {
    rootVC.present(viewController, animated: true, completion: nil)
    return
}

if viewController is WebViewVC {
    (viewController as? WebViewVC)?.onCloseTap = {
        rootVC.dismiss(animated: true, completion: nil)
    }
    let navCtrl = BaseNavigationController(rootViewController: viewController)
    rootVC.present(navCtrl, animated: true, completion: nil)
    return
}

if let tabBarCtrl = rootVC as? UITabBarController,
    let navCtrl = tabBarCtrl.selectedViewController as? UINavigationController
        ?? tabBarCtrl.selectedViewController?.navigationController {
    navCtrl.pushViewController(viewController, animated: true)
    return
}

if let navCtrl = rootVC as? UINavigationController ?? rootVC.navigationController {
    navCtrl.pushViewController(viewController, animated: true)
    return
}

```

I strongly encourage using *Reactive* approach with *MVVM*. For the sake of simplicity, I will skip the *ViewModel* part and only continue using Coordinators and ViewControllers.

## App structure

{% raw %}
<img src="../../assets/images/appStructure.png" alt="">
{% endraw %}

We will implement deeplinks for the above app structure. Our main goal is to be able to deeplink to any part of the app. There are multiple layers of coordinators/childCoordinators and our job is to load the correct screen and keep the same navigation hierarchy.

## Go with the flow

Let's start by separating our app into flows. We can create one flow per coordinator.
So our `AppFlow` should look like this:

```swift
enum AppFlow: String {
    case home
    case login
}
```
In this case we either navigate the user to the login screen or the main dashboard.

And our *Home* and *Login* parts should look like this. The other flows should be implemented in the same manner.

```swift
enum HomeFlow: String {
    case dashboard
    case profile
    case settings
} 

enum LoginFlow: String {
    case login
    case registration
}

enum DashboardFlow: String {
    case card
}

enum CardFlow: String {
    case cardDetail
}

```

### The fun part
Now that we had defined several flows, how can we pass the deeplink from `AppFlow` to other child flows? How do we now that *cardDetail* belongs to `CardFlow`, or that we have to go through  

`AppFlow → HomeFlow → DashboardFlow → CardFlow` to show `CardDetailVC`?

### Recursion

{% raw %}
<img src="../../assets/images/recursive-gru.jpg" alt="">
{% endraw %}

We can use a simple recursion with custom inits for the flows. Something like this
```swift

enum AppFlow: String {
    case home
    case login

    init?(deeplink: String) {
        if let flow = AppFlow(rawValue: deeplink) {
            self = flow
            return
        }

        if HomeFlow(deeplink: deeplink) != nil {
            self = .home
        } else if LoginFlow(deeplink: deeplink) != nil {
            self = .login
        } else {
            return nil
        }
    }
}

enum HomeFlow: String {
    case dashboard
    case profile
    case settings

    init?(deeplink: String) {
        if let flow = HomeFlow(rawValue: deeplink) {
            self = flow
            return
        }

        if DashboardFlow(deeplink: deeplink) != nil {
            self = .dashboard
        } else if ProfileFlow(deeplink: deeplink) != nil {
            self = .profile
        } else if SettingsFlow(deeplink: deeplink) != nil {
            self = .settings
        } else {
            return nil
        }
    }
}

...

enum CardFlow: String {
    case cardDetail

    init?(deeplink: String) {
        if let flow = CardFlow(rawValue: deeplink) {
            self = flow
        } else {
            return nil
        }
    }
}
```
Pay attention to init parts, there are differences between *rawValue* and *deeplink*

Let's walk through how our flow logic should work right now.

Now given the route `AppFlow → HomeFlow → DashboardFlow → CardFlow`, we can go through each corresponding coordinator and handle it separately.

In `AppCoordinator` we will initialize `Appflow(deeplink: "cardDetail")`. By the above-defined recursion, it should return `.home` case, so we know that we should push the `HomeCoordinator`.

We will do the same in `HomeFlow(deeplink: "cardDetail")` and it should return `.dashboard` case. Our `HomeCoordinator` has a `UITabBarController` so it will select the Dashboard. After that, we pass the deeplink deeper into the `DashboardCoordinator`

We will repeat the above step for the Dashboard and now finally in `CardCoordinator` we initialize `CardFlow(deeplink: "cardDetail")` and push the corresponding `CardDetailVC`

## Real application

I am going to try this out with *reactive* approach using *Combine*. So our base *Coordinator* part with deeplink handling now looks like this

```swift
class Coordinator {
    let deeplinkSubject = CurrentValueSubject<String?, Never>(nil)
    var deeplinkDisposeBag = Set<AnyCancellable>()

    func resetDeeplink() {
        for child in childCoordinators {
            child.deeplinkDisposeBag = Set<AnyCancellable>()
        }
        deeplinkSubject.send(nil)
    }

    func addChild(_ coordinator: Coordinator) {
        deeplinkSubject
            .subscribe(coordinator.deeplinkSubject)
            .store(in: &coordinator.deeplinkDisposeBag)

        childCoordinators.append(coordinator)
    }
}
```
We are passing deeplinks deeper into child coordinators by binding deeplinkSubjects in
`addChild` function.  

Note that we used `CurrentValueSubject` (`BehaviorSubject` in RxSwift) instead of `PassthroughSubject` (`PublishSubject` in RxSwift) because in the time the deeplinkSubject gets emitted, we may not have our child Coordinators initialized.  

We have to `resetDeeplink` after we process the deeplink. That should solve the cases when the deeplink gets emitted again and we are already deeper in the navigation.

In this implementation, we will always reset the navigation when the deeplink gets emitted.

The actual implementation in the coordinators:
```swift
// AppCoordinator
private func bindDeeplink() {
    deeplinkSubject
        .unwrap()
        .map(AppFlow.init(deeplink:))
        .unwrap()
        .receive(on: DispatchQueue.main)
        .sink { [weak self] deeplink in
            guard let self = self else { return }
            switch deeplink {
            case .home:
                self.setHome()
            case .login:
                self.setLogin()
            }
            self.resetDeeplink()
        }.store(in: &disposeBag)
}

private func setHome() {
    let homeCoordinator = HomeCoordinator(router: router, navigationType: .newFlow(hideBar: true))
    setRootChild(coordinator: homeCoordinator, hideBar: true)
}

private func setLogin() {
    let loginCoordinator = LoginCoordinator(router: router, navigationType: .newFlow(hideBar: false), userManager: userManager)
    setRootChild(coordinator: loginCoordinator, hideBar: false)
}

// HomeCoordinator
private func bindDeeplink() {
    deeplinkSubject
        .unwrap()
        .map(HomeFlow.init(deeplink:))
        .unwrap()
        .receive(on: DispatchQueue.main)
        .sink { [weak self] deeplink in
            guard let self = self else { return }
            switch deeplink {
            case .dashboard:
                self.tabBarController.selectedViewController = self.dashboardCoordinator.toPresentable()
            case .profile:
                self.tabBarController.selectedViewController = self.profileCoordinator.toPresentable()
            case .settings:
                self.tabBarController.selectedViewController = self.settingsCoordinator.toPresentable()
            }
            self.resetDeeplink()
        }.store(in: &disposeBag)
}

// DashboardCoordinator
private func bindDeeplink() {
    deeplinkSubject
        .unwrap()
        .map(DashboardFlow.init(deeplink:))
        .unwrap()
        .receive(on: DispatchQueue.main)
        .sink { [weak self] deeplink in
            guard let self = self else { return }
            switch deeplink {
            case .card:
                self.showCard(animated: false)
            }
            self.resetDeeplink()
        }.store(in: &disposeBag)
}

private func showCard(animated: Bool = true) {
    let cardCoordinator = CardCoordinator(router: router, navigationType: .currentFlow)
    pushChild(coordinator: cardCoordinator, animated: animated)
}
```

## Other dependencies

There will often be times when you need to fetch and load some data to continue or that you need to wait for some other asynchronous operations (eg. wait when the user logs in). That is where the *reactive* approach comes in handy. You can apply all sorts of operations (combineLatest, zip, withLatestFrom, flatMap, etc.) to those signals and then process them when everything is loaded.

You can find a complete example [here](https://github.com/iaminh/CoordinatorExample).
