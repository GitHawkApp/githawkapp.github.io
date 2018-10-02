---
layout: post
author: ryannystrom
title: Local Notifications with Background Fetch
---

This is how we used [Background Fetch](https://developer.apple.com/documentation/uikit/core_app/managing_your_app_s_life_cycle/preparing_your_app_to_run_in_the_background/updating_your_app_with_background_app_refresh) to show Local Notifications instead of traditional Push Notifications in GitHawk. We used [FMDB](https://github.com/ccgus/fmdb) to track delivered alerts in our [implementation](https://github.com/GitHawkApp/GitHawk/blob/master/Classes/Systems/LocalNotificationsCache.swift).

<p align="center"><img src="https://user-images.githubusercontent.com/739696/46327776-782f4280-c5d1-11e8-8242-561b77e79e54.jpg" width="600" /></p>

One of our values with GitHawk is respecting your privacy, especially when it comes to your GitHub data. We don't use any third-party servers. Instead, we connect directly to GitHub via their API and keep all of your authentication information on your phone.

That poses a huge challenge to adding our most-requested feature for GitHawk: **Push Notifications**.

![App Store Review](https://user-images.githubusercontent.com/739696/46327873-ff7cb600-c5d1-11e8-873c-26ae39bc202a.jpg)

With traditional Apple Push Notification (APN) implementations, you:

- Ask the user permission
- Get a callback with the device token
- Send that token to your server and save it
- Whenever you need to send a notification, send content along with the token to Apple's APN servers

That middle part, having to send a token to our servers, can't happen because we don't want to have access to your private auth data.

However, GitHawk has been using [app background fetch](https://developer.apple.com/documentation/uikit/core_app/managing_your_app_s_life_cycle/preparing_your_app_to_run_in_the_background/updating_your_app_with_background_app_refresh) APIs (glorified polling) to update the badge icon for months now. We decided to piggy-back off of this existing feature with "fake" push notifications.

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

The hard part is making this work well with the background fetch API and not annoying your users.

### Setup

First off, you have to ask for notification permissions!

```swift
UNUserNotificationCenter.current().requestAuthorization(options: [.alert], completionHandler: { (granted, _) in
  // handle if granted or not
})
```

> In GitHawk, we disable notifications **by default** and let the user enable them in a settings screen. This avoids being bombarded with annoying permissions dialogs the first time you open the app.

Next, set the fetch interval. If you're doing this in lieu of actual push notifications, you might as well set this to the minimum time interval.

```
UIApplication.shared.setMinimumBackgroundFetchInterval(UIApplicationBackgroundFetchIntervalMinimum)
```

Next up, handle background fetch system calls by overriding the `UIApplication.application(_:, performFetchWithCompletionHandler:)` function. The `AppDelegate` will be initialized, even if your app was started cold.

```swift
func application(_ application: UIApplication, performFetchWithCompletionHandler completionHandler: @escaping (UIBackgroundFetchResult) -> Void) {
  rootNavigationManager.client?.badge.fetch(application: application, handler: completionHandler)
}
```

> In GitHawk, we lazily initialize our session objects which then become readily available, making networking easy.

### Avoiding Notification Fatigue

If we alerted all content on every fetch, you'd uninstall GitHawk pretty quick! We need to only alert for content that is new, and do it in a performant way.

The naive approach would be to toss IDs and timestamps in `UserDefaults`, but that's gross, and remember that `UserDefaults` are loaded into memory on app start! That's a recipe for slow perf down the road.

Thankfully, this is just what databases are for! SQLite is a wonderfully lightweight database with first-class support on iOS, and [FMDB](https://github.com/ccgus/fmdb) removes all the hairy bits.

Before we jump into code, let's design how this system should work:

- We need an `id: String` for each content that should alert. This will be the key in our table.
- Create the table if it doesn't already exist
- Select all `id`s already in the table and **remove** them from content that will alert.
- Insert all of the new `id`s into the table
- Trim the table so it doesn't grow unbounded

> Trimming the table may be an unnecessary optimization, but the last thing I want is a 100mb SQLite file polluting people's phones just to have notification receipts.

In order to keep this thread-safe, we use `FMDatabaseQueue` and execute all database transactions in the a single closure.

```swift
let queue = FMDatabaseQueue(path: databasePath)
queue.inDatabase { db in
  // queries and stuff
}
```

In GitHawk, the first thing we do is convert an array of content into a mutable `Dictionary<String: Content>` so that we can remove already-alerted content and then lookup content by its `id`.

```swift
var map = [String: Content]()
contents.forEach { map[$0.id] = $0 }
let ids = map.keys.map { $0 }
```

Then inside our database transaction closure, start by creating the table:

```swift
queue.inDatabase { db in 
  do {
    try db.executeUpdate(
      "create table if not exists receipts (id integer primary key autoincrement, content_id text)",
      values: nil
    )
  } catch {
    print("failed: \(error.localizedDescription)")
  }
}
```

> SQLite's `if not exists` makes creating the table once a breeze!

Next we select `id`s that already exist in the table and remove them from the `map`:

```swift
queue.inDatabase { db in 
  do {
    // create table...

    let selectParams = map.keys.map { _ in "?" }.joined(separator: ", ")
    let rs = try db.executeQuery(
      "select content_id from receipts where content_id in (\(selectParams))",
      values: ids
    )
    while rs.next() {
      if let key = rs.string(forColumn: "content_id") {
        map.removeValue(forKey: key)
      }
    }

  } catch {
    print("failed: \(error.localizedDescription)")
  }
}
```

Now the `map` is populated with content that we haven't seen before, so we need to add of these `id`s to the table.

```swift
queue.inDatabase { db in 
  do {
    // create table...
    // select content ids...

    let insertParams = map.keys.map { _ in "(?)" }.joined(separator: ", ")
    try db.executeUpdate(
      "insert into receipts (content_id) values \(insertParams)",
      values: map.keys.map { $0 }
    )

  } catch {
    print("failed: \(error.localizedDescription)")
  }
}
```

Before we're done with the database, let's cap the `receipts` table to 1000 entries. We can do this easily with only the `autoincrement` column:

```swift
queue.inDatabase { db in 
  do {
    // create table...
    // select content ids...
    // insert content ids...

    try db.executeUpdate(
      "delete from receipts where id not in (select id from receipts order by id desc limit 1000)",
      values: nil
    )

  } catch {
    print("failed: \(error.localizedDescription)")
  }
}
```

> GitHub's API only returns 50 notifications at a time, so limiting the history to 1,000 shouldn't have any noticeable repeated notifications.

All that's left to do is iterate through remaining `map.values` and fire off some `UNNotificationRequest`s!

## Wrapping Up

While not as fully featured as traditional Apple Push Notifications (realtime alerts, rich notifications, etc), using Local Notifications and Background Fetch got us an MVP of the most-requested feature in users hands quick. And most importantly, it avoids any need for us to store private authentication data off people's phones.

You can check out our implementation of this system in [LocalNotificationCache.swift](https://github.com/GitHawkApp/GitHawk/blob/7b0746332129d3077f1ba036c9c20ebe45d27751/Classes/Systems/LocalNotificationsCache.swift) and [BadgeNotifications.swift](https://github.com/GitHawkApp/GitHawk/blob/7b0746332129d3077f1ba036c9c20ebe45d27751/Classes/Systems/BadgeNotifications.swift#L125-L144).

> If there's enough interest, I'm happy to extract this functionality into its own Pod!)