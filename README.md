![Liferay Mobile SDK logo](https://github.com/liferay/liferay-mobile-sdk/raw/master/logo.png)

# Liferay Push for Android

[![Build Status](https://travis-ci.org/liferay-mobile/liferay-push-android.svg?branch=master)](https://travis-ci.org/liferay-mobile/liferay-push-android)
[![Coverage Status](https://coveralls.io/repos/liferay-mobile/liferay-push-android/badge.svg?branch=master&t=1)](https://coveralls.io/r/liferay-mobile/liferay-push-android?branch=master)

* [Setup](#setup)
* [Use](#use)
	* [Registering a device](#registering-a-device)
	* [Receiving push notifications](#receiving-push-notifications)
	* [Sending push notifications](#sending-push-notifications)
	* [Unregistering a device](#unregistering-a-device)

## Setup

Add the library as a dependency to your project's build.gradle file:

```groovy
dependencies {
	implementation 'com.liferay.mobile:liferay-push:1.3.0'
}
```

If you are using **liferay-mobile-sdk version <= 7.1.3** or **liferay-screens version <= 5.0.0**
You should use this version instead

```groovy
dependencies {
	implementation 'com.liferay.mobile:liferay-push:1.2.0.1'
}
```


### Breaking changes in version 1.2.0

Since version 1.2.0 you have to add a new property when declaring the PushService:

```xml
<service android:name=".PushService"
	android:permission="android.permission.BIND_JOB_SERVICE" /> <!--You have to add this-->
```

## Use

### Registering a device

To receive push notifications, your app must register itself to the portal first. On the portal side, each
device is tied to a user. Each user can have multiple registered devices. A device is represented by a device token string. Google calls this the `registrationId`.

To register a device, we need to configure our project with firebase, for doing so, we need to add the application to our firebase project settings:

<img src="docs/images/Register Android Application.png">

Click on adding the android application and follow the instructions.


After registering the app in firebase, it's easy to register a device with *Liferay Push for Android*, you just have to call to the following method:

```java
import com.liferay.mobile.push.Push;

Session session = new SessionImpl("http://localhost:8080", new BasicAuthentication("test@liferay.com", "test"));

Push.with(session).register();
```

**If you want to use Liferay 7.x you should manually specify the version** with a call like this:

```java
push.withPortalVersion(70)
```

Since all operations are asynchronous, you can set callbacks to check if the registration succeeded or an error occurred on the server side:

```java
Push.with(session)
	.onSuccess(new Push.OnSuccess() {

		@Override
		public void onSuccess(JSONObject jsonObject) {
			System.out.println("Device was registered!");
			registrationId = jsonObject.getString("token");
		}

	})
	.onFailure(new Push.OnFailure() {

		@Override
		public void onFailure(Exception e) {
			System.out.println("Some error occurred!");
		}

	})
	.register();
```

The `onSuccess` and `onFailure` callbacks are optional, but it's good practice to implement both. By doing so, your app can persist the registrationId device token or tell the user that an error occurred.

*Liferay Push for Android* is calling the *GCM server*, retrieving the results and storing your `registrationId` in the Liferay Portal instance for later use.

Don't forget to add in your app the *Internet* permission if you haven't done it already:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```
		
And, if you are using Liferay 7, you will have to add permissions to be able to register the device in the portal:

<img src="docs/images/Liferay Permissions.png">

All set! If everything went well, you should see a new device registered under the *Push Notifications* menu in *Configuration*. Next step is [Receiving push notifications](#receiving-push-notifications)


#### Using the registrationId directly without registering against Liferay Portal

If you obtain the token manually, you can register the device to the portal by calling the following method:

```java
Push.with(session).register(registrationId);
```

Now each time the portal wants to send a push notification to the user `test@liferay.com`, it looks up all registered devices for the user (including the one just registered) and sends the push notification for each `registrationId` found.

You should note that the [Push](src/main/java/com/liferay/mobile/push/Push.java) class is a wrapper for the Mobile SDK generated services. Internally, it calls the Mobile SDK's `PushNotificationsDeviceService` class. While you can still use `PushNotificationsDeviceService` directly, using the wrapper class is easier.

### Receiving push notifications

Once your device is registered, you have to configure both the server and the client to be able to receive push messages.

To send notifications from Liferay you should configure the `API_KEY` inside *System Settings*, *Notifications* and *Firebase*. To obtain the `API_KEY` you should, again, access your firebase project settings and under *Cloud Messaging*, use the **Server Key**. 

<img src="docs/images/Firebase Console Server Key.png">
 
Then you have to configure your project to be able to listen for notifications:

* You should implement a `BroadcastReceiver` instance in your app. [Android's developer documentation](http://developer.android.com/google/gcm/client.html#sample-receive) shows you how to do this. Specifically, you should:

	* Register a `<uses-permission android:name="com.google.android.c2dm.permission.RECEIVE" />` in your *AndroidManifest.xml* file.
	* Register a WAKE_LOCK permission and Internet if you had not used those permissions before:

		```xml
		<uses-permission android:name="android.permission.INTERNET" />
		<uses-permission android:name="android.permission.WAKE_LOCK" />
		```
	
	* Add a *BroadcastReceiver* and a *IntentService* to your project:

		```xml
		<receiver
	        android:name=".PushReceiver"
	        android:permission="com.google.android.c2dm.permission.SEND">
	        <intent-filter>
	            <action android:name="com.google.android.c2dm.intent.RECEIVE" />
	            <category android:name="com.liferay.mobile.push" />
	        </intent-filter>
	    </receiver>
	
	   <service android:name=".PushService" 
	   		android:permission="android.permission.BIND_JOB_SERVICE"/>
		```
	
	* The code implementing those classes is really simple:

		```java
		public class PushReceiver extends PushNotificationsReceiver {
		    @Override
		    public String getServiceClassName() {
		        return PushService.class.getName();
		    }
		}
		
		public class PushService extends PushNotificationsService {
			 @Override
		    public void onPushNotification(JSONObject jsonObject) {
		        super.onPushNotification(jsonObject);
		        
		        // Your own code to deal with the push notification
		    }
		}
		```

* If you want to execute an action or show a notification only if the application is active, you could register a callback:

```java
Push.with(session).onPushNotification(new Push.OnPushNotification() {

	@Override
	public void onPushNotification(JSONObject jsonObject) {
	}

});
```
	
This method only works if you have already registered against Liferay Portal using the previous instructions.

### Sending push notifications

There are many ways to send push notifications from Liferay Portal. See the [Liferay Push documentation](../README.md) for more details. Alternatively, you can send push notifications from your Android app. Just make sure the user has the proper permissions in the portal to send push notifications.

```java
JSONObject notification = new JSONObject();
notification.put("message", "Hello!");

Push.with(session).send(toUserId, notification);
```

In this code, the push notification is sent to the user specified by `toUserId`. Upon receiving the notification, the portal looks up all the user's registered devices (both Android and iOS devices) and sends `notification` as the body of the push notification.

### Unregistering a device

If you want to stop receiving push notifications on a device, you can unregister it from from the portal with the following code:

```java
Push.with(session).unregister(registrationId);
```

Users can only unregister devices they own and they need to have the MANAGE_DEVICES permission.
