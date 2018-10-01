# Local Notifications with Background Fetch

* `application(_:, performFetchWithCompletionHandler:)`
* do network fetch
* launches app, disable networking in `viewDidLoad` by checking `UIApplication.shared.applicationState` 
* FMDB
  - queue
  - create, select, insert, delete
* return filtered ids
* send unfiltered as notifications

---
layout: post
author: ryannystrom
title: Local Notifications with Background Fetch
---

One of our values with GitHawk is respecting your privacy, especially when it comes to your GitHub data. We don't use any third-party servers. Instead, we connect directly to GitHub via their API and keep all of your authentication information on your phone.

That poses a huge challenge to adding our most-requested feature for GitHawk: **Push Notifications**.

![App Store Review](https://user-images.githubusercontent.com/739696/46262026-927df900-c4c9-11e8-9d15-7c65825f1ddf.jpg)

With traditional Apple Push Notification (APN) implementations, you:

- Ask the user permission
- Get a callback with the device token
- Send that token to your server and save it
- Whenever you need to send a notification, send content along with the token to Apple's APN servers

That middle part, having to send a token to our servers, can't happen because we don't want to have access to your private auth data.

However, GitHawk has been using [app background fetch](https://developer.apple.com/documentation/uikit/core_app/managing_your_app_s_life_cycle/preparing_your_app_to_run_in_the_background/updating_your_app_with_background_app_refresh) APIs to update the badge icon for months now. We decided to piggy-back off of this existing feature with "fake" push notifications.

## Local Notifications

Sending local notifications is _easy_ with the `UserNotifications` framework:

```swift
let content = UNMutableNotificationContent()
content.title = "Alert!"
content.body = "Something happened"

let request = UNNotificationRequest(
  identifier: UUID(),
  content: content,
  trigger: UNTimeIntervalNotificationTrigger(timeInterval: 0, repeats: false)
  )
UNUserNotificationCenter.current().add(request)
```

The hard part is making this work well with the background fetch API, and not annoying your users.

### Setup

First off, you have to ask for notification permissions!

```swift
UNUserNotificationCenter.current().requestAuthorization(options: [.alert], completionHandler: { (granted, _) in
  // handle if granted or not
})
```

> In GitHawk, we disable notifications **by default** and let the user enable them in a settings screen. This avoids being bombarded with annoying permissions dialogs the first time you open the app.

Next, set the fetch interval. If you're doing this in lieu of actual push notifications, might as well set this to a minimum time interval.

```
UIApplication.shared.setMinimumBackgroundFetchInterval(UIApplicationBackgroundFetchIntervalMinimum)
```


```swift
func application(_ application: UIApplication, performFetchWithCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
  rootNavigationManager.client?.badge.fetch(application: application, handler: completionHandler)
}
```