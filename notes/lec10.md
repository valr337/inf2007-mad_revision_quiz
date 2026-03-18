# INF2007 Lecture 10 Notes
## Background Work, Broadcast Receiver, Content Provider

## 1. Lecture scope
This lecture covers three major areas:
- Background work
- Broadcast receivers
- Content providers

## 2. Services
A **Service** is one of Android's big four components. It is used for work that:
- does not need or directly affect the UI
- is long running
- may need to continue even after the Activity is no longer visible or has shut down

### Typical service use cases
- Network transactions
- Continuous database transactions, including local ones
- Intensive long-running operations

### Real-world examples mentioned
- Email service
- Text or messaging service
- Calendar service

Example idea:
If a user taps **Send** in an email app and leaves the compose screen, the email can still continue sending through a service.

## 3. Ways to use services
The lecture distinguishes between two direct service patterns:
- **Started service**
- **Bound service**

For scheduled background work, the lecture recommends **WorkManager** instead.

## 4. Started services
A **started service** is launched by something like an Activity using `startService(intent)`.

### Started service characteristics
- Usually there is **no direct communication channel** back to the Activity
- The service can continue doing work independently
- Since services run on the **main UI thread by default**, long-running work should usually be moved to a **background thread**

### Started service lifecycle flow
1. A `Context` such as an Activity calls `startService(intent)`
2. `onCreate()` runs for setup
3. `onStartCommand()` runs and contains the main service task logic
4. When work is complete, the service calls `stopSelf()`
5. `onDestroy()` runs when the service is destroyed

### Key exam point
`onStartCommand()` is where the actual service work is typically implemented.

## 5. Bound services
A **bound service** allows components such as Activities to **communicate with the service**.

Example from the lecture:
A music player UI Activity can be bound to a music playing service.
- Service to Activity: "played n seconds of music"
- Activity to Service: "user clicked pause"

### Bound service lifecycle flow
1. A `Context` calls `bindService(...)`
2. `onCreate()` runs for setup
3. Each client calls `onBind()` to get an `IBinder`
4. The client uses public methods from the service through that binder
5. When done, the client calls `onUnbind()`
6. When all clients are unbound, `onDestroy()` is triggered

### Local bound service pattern
If the service is only used **within the same app**, a common pattern is:
- define an inner `Binder` class
- expose a method like `getService()` to return the service instance
- let the client call the service's public methods directly

### Activity-side binding pattern
Typical Activity flow:
- Create a `ServiceConnection`
- In `onServiceConnected()`, cast the `IBinder` and get the service instance
- Bind in `onStart()`
- Unbind in `onStop()`

### Cross-process service access
If the service is used by **other apps or processes**, the lecture mentions:
- **Messenger**: use when requests can be handled sequentially
- **AIDL**: use when simultaneous processing of client requests is needed

## 6. Notifications and services
Notifications help the user know what a background service is doing.

### Notification points from the lecture
- Notifications are managed by **NotificationManager**
- Notifications can include **actions**
- Notifications can support wearable-compatible app interactions
- Do not overuse them

### PendingIntent
A `PendingIntent` is used when another component, such as `NotificationManager`, should execute your app's intent later.

The lecture highlights that PendingIntent can:
- transfer your app's permissions to another app or system component for that action
- launch an **Activity**, **Service**, or **BroadcastReceiver** later

### Common notification launch flow
1. Create an `Intent` for the target Activity
2. Wrap it in `PendingIntent.getActivity(...)`
3. Build a `Notification.Action` using that pending intent

### Notification actions
The lecture notes that a notification can have **up to 3 actions**.

### Notification channels and runtime permission
The slides mention:
- **Notification Channels** were introduced in Android Oreo
- **Notification runtime permission** is new in Android Tiramisu

## 7. Foreground services
A **foreground service** is a service the user should be actively aware of.

### Foreground service properties
- Shows a **non-dismissible ongoing notification**
- Android gives it higher priority, especially under memory pressure

Typical call:
```kotlin
startForeground(NOTIFICATION_ID, notification)
```

## 8. WorkManager
The lecture presents **WorkManager** as the recommended solution for persistent background work.

### Why WorkManager is recommended
- It is the primary recommended API for persistent background processing
- Scheduled work can remain through:
  - app restarts
  - system reboots
- It supports:
  - work constraints
  - robust scheduling
  - work chaining
  - immediate, long-running, and deferrable persistent work

### Basic steps for WorkManager
1. Define a `Worker` subclass
2. Implement the task inside `doWork()`
3. Create a `WorkRequest`
4. Submit it using `WorkManager.getInstance(context).enqueue(request)`

### Example pattern
```kotlin
class SimpleWorker(context: Context, params: WorkerParameters) : Worker(context, params) {
    override fun doWork(): Result {
        Log.d("SimpleWorker", "do work in SimpleWorker")
        return Result.success()
    }
}

val request = OneTimeWorkRequest.Builder(SimpleWorker::class.java)
    .setInitialDelay(5, TimeUnit.MINUTES)
    .build()

WorkManager.getInstance(context).enqueue(request)
```

### WorkManager usage guidance from the lecture
- **Coroutines**: for asynchronous work
- **WorkManager**: for persistent background work
- **JobScheduler**: mentioned as another background option

### Special WorkManager calls in the slides
- `setExpedited()`
  - for important requests
  - usually short tasks that should start immediately
- `setForegroundAsync()` / `setForeground()`
  - for important or long-running requests

## 9. Broadcast receivers
A **BroadcastReceiver** is another one of Android's big four components.

### Purpose
Broadcast receivers implement a **publish-subscribe pattern**.
They allow apps to receive broadcasts from:
- the Android system
- other apps through custom broadcasts

### Important property
A broadcast receiver can be triggered **even when the app is not running**.

## 10. System broadcasts
System broadcasts are special intents sent by Android when system events occur.

Examples listed in the lecture:
- `ACTION_LOCKED_BOOT_COMPLETED`
- `ACTION_NEW_OUTGOING_CALL`
- `ACTION_USB_ACCESSORY_ATTACHED`
- `ACTION_USB_ACCESSORY_DETACHED`
- `ACTION_LOCALE_CHANGED`
- `SMS_RECEIVED_ACTION`

### Key point
Most system broadcasts are **implicit broadcasts**.

## 11. Intent filters
An **IntentFilter** declares which implicit intents should trigger a component.

The lecture says intent filters can be:
- **statically declared** in the manifest
- **dynamically coded** in the app

### Example concept
The launcher Activity is associated with an intent filter containing:
- `android.intent.action.MAIN`
- `android.intent.category.LAUNCHER`

## 12. Static broadcast receivers
A **static** broadcast receiver is declared in the manifest.

### Main idea
It can react to a matching implicit intent **even when the app is closed**.

### But there are restrictions
The lecture warns that this mechanism is powerful and easily abused, so Android has made the rules stricter.

Examples mentioned:
- Some intents can only be registered **dynamically** in code
- Android N removed some implicit broadcasts such as:
  - `CONNECTIVITY_ACTION`
  - `ACTION_NEW_PICTURE`
  - `ACTION_NEW_VIDEO`
- Android O removed most implicit broadcasts except a limited set of exceptions

### Exam caution
A static receiver example in the slides ends with:
> Q: is the above code valid?

This is a hint that older manifest-declared patterns may no longer be valid depending on the API level and broadcast type.

## 13. Dynamic broadcast receivers
A **dynamic** broadcast receiver exists only during the app lifecycle.

### Lecture recommendation
Use dynamic receivers **whenever possible**.

### Example lifecycle pattern
```kotlin
class NewBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // do stuff
    }
}

class SomeActivity {
    private val receiver = NewBroadcastReceiver()

    override fun onResume() {
        super.onResume()
        registerReceiver(receiver, IntentFilter(Intent.ACTION_HEADSET_PLUG))
    }

    override fun onPause() {
        super.onPause()
        unregisterReceiver(receiver)
    }
}
```

### Key point
Dynamic receivers are tied to the Activity or app lifecycle, unlike static receivers declared in the manifest.

## 14. Custom broadcasts
Apps can also define and send their own broadcasts.

Example custom action from the lecture:
```kotlin
val intent = Intent("com.mypackage.CUSTOM_INTENT")
```

### Ways to send custom broadcasts
- `sendBroadcast(intent)`
  - receivers get it in undefined order
- `sendOrderedBroadcast(intent, permission)`
  - receivers get it one at a time
  - receivers can pass cumulative results along
  - ordering can be affected by `android:priority`

### Broadcast permissions
The lecture also shows that you can restrict who receives a broadcast by requiring a permission.

Sender side example idea:
```kotlin
sendBroadcast(notiIntent, Manifest.permission.SEND_SMS)
```

Receiver side requirements:
- request the same permission
- register/unregister the receiver in the intended lifecycle
- implement `onReceive()`

## 15. Content providers
A **ContentProvider** is also one of Android's big four components.

### Main purpose
Content providers manage access to data stored by:
- the app itself
- other apps

They also provide a structured way to **share data across app boundaries**.

### Why content providers matter
They:
- encapsulate data
- provide mechanisms for defining data security
- allow controlled access from outside the app

## 16. Benefits of content providers
The lecture lists these benefits:
- granular control over and enforcement of permissions for accessing data
- abstraction over different data sources
- encapsulation of datasets
- inter-application data sharing across app boundaries

## 17. Common content provider operations
The diagram in the lecture highlights the typical operations:
- `insert()`
- `update()`
- `delete()`
- `query()`

These operations are typically implemented by the provider and operate on the app's data source, such as a database.

## 18. Typical content provider scenario
The lecture shows a common architecture:
- a **UI app** has an Activity
- a separate **Data app** exposes a content provider
- the content provider accesses the underlying database

This allows one app to use another app's structured data safely and consistently.

## 19. Android examples of content providers
Built-in Android examples mentioned in the lecture:
- Browser: bookmarks and history
- Call log: telephone usage
- Contacts: contact data
- Media: media database
- UserDictionary: predictive spelling database

## 20. High-yield comparisons

### Service vs WorkManager
- **Service**: good for ongoing or active app work, especially when immediate execution and direct app-controlled behavior are needed
- **WorkManager**: recommended for persistent scheduled background work that should survive restarts and reboots

### Started service vs Bound service
- **Started service**: launched to do work independently, usually no direct communication
- **Bound service**: allows clients to communicate with the service using a binder

### Static vs Dynamic broadcast receiver
- **Static**: declared in manifest, may work when app is closed, but heavily restricted in newer Android versions
- **Dynamic**: registered in code, exists only during app lifecycle, generally preferred whenever possible

### Broadcast receiver vs Content provider
- **Broadcast receiver**: receives event-style intent messages
- **Content provider**: exposes structured data operations like query, insert, update, delete

## 21. Fast revision checklist
Before exams or quizzes, make sure you can explain:
- why services exist beyond the Activity lifecycle
- why services should avoid blocking the UI thread
- difference between started and bound services
- when `onStartCommand()` and `onBind()` are used
- why PendingIntent is needed in notifications
- what makes a service a foreground service
- why WorkManager is preferred for persistent background work
- difference between Coroutines and WorkManager
- what broadcast receivers listen for
- difference between static and dynamic receivers
- difference between `sendBroadcast()` and `sendOrderedBroadcast()`
- when content providers should be used
- benefits and common operations of content providers

## 22. One-line memory aid
- **Service** = background work
- **WorkManager** = persistent scheduled work
- **BroadcastReceiver** = listen for system/app events
- **ContentProvider** = share structured data safely across apps
