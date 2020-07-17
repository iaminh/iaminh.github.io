---
title: "Deeplinking with coordinators"
tags:
  - architecture
  - coordinator
header:
  teaser: "assets/images/hackerimg.jpg"
---

Countless of articles about **Coordinator** pattern and how great it is to manage deeplinking, but not many really goes in depth on how to actually use it in real production apps.



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

Let's see how we can improve this :)

## Coordinators

The coordinator pattern is wildly known and there are many articles about that. That's why I will not describe how to actually implement them.

You can also use my [implementation](https://github.com/iaminh/CoordinatorExample) to follow the guide.

## App structure

{% raw %}
<img src="../../assets/images/appStructure.png" alt="">
{% endraw %}

We will implement deeplinks for this app structure. Our main purpose is to be able to deeplink to any part of the app. There are multiple layers of coordinators/childCoordinators and our job is to load the correct screen

