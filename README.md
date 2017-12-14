## Summary

This is the Android SDK of Adjust™. You can read more about Adjust™ at [adjust.com].

If your app is an app which uses web views and you would like to use Adjust tracking from Javascript code, please consult
our [Android web views SDK guide](doc/english/web_views.md).

## Table of contents

* [Example app](#example-app)
* [Basic integration](#basic-integration)
   * [Add the SDK to your project](#sdk-add)
   * [Add Google Play Services](#sdk-gps)
   * [Add permissions](#sdk-permissions)
   * [Proguard settings](#sdk-proguard)
   * [Install referrer](#install-referrer)
      * [Google Play Referrer API](#gpr-api)
      * [Google Play Store intent](#gps-intent)
   * [Integrate the SDK into your app](#sdk-integrate)
   * [Basic setup](#basic-setup)
   * [Session tracking](#session-tracking)
      * [API level 14 and higher](#session-tracking-api14)
      * [API level between 9 and 13](#session-tracking-api9)
   * [Adjust logging](#adjust-logging)
   * [Build your app](#build-the-app)
* [Additional features](#additional-features)
   * [Event tracking](#event-tracking)
      * [Track revenue](#revenue-tracking)
      * [Revenue deduplication](#revenue-deduplication)
      * [In-App Purchase verification](#iap-verification)
      * [Callback parameters](#callback-parameters)
      * [Partner parameters](#partner-parameters)
   * [Session parameters](#session-parameters)
      * [Session callback parameters](#session-callback-parameters)
      * [Session partner parameters](#session-partner-parameters)
      * [Delay start](#delay-start)
   * [Attribution callback](#attribution-callback)
   * [Session and event callbacks](#session-event-callbacks)
   * [Disable tracking](#disable-tracking)
   * [Offline mode](#offline-mode)
   * [Event buffering](#event-buffering)
   * [Background tracking](#background-tracking)
   * [Device IDs](#device-ids)
      * [Google Play Services advertising identifier](#di-gps-adid)
      * [Amazon advertising identifier](#di-amz-adid)
      * [Adjust device identifier](#di-adid)
   * [User attribution](#user-attribution)
   * [Push token](#push-token)
   * [Pre-installed trackers](#pre-installed-trackers)
   * [Deep linking](#deeplinking)
      * [Standard deep linking scenario](#deeplinking-standard)
      * [Deferred deep linking scenario](#deeplinking-deferred)
      * [Reattribution via deep links](#deeplinking-reattribution)
* [Troubleshooting](#troubleshooting)
   * [I'm seeing the "Session failed (Ignoring too frequent session. ...)" error](#ts-session-failed)
   * [Is my broadcast receiver capturing the install referrer?](#ts-broadcast-receiver)
   * [Can I trigger an event at application launch?](#ts-event-at-launch)
* [License](#license)

## <a id="example-app"></a>Example app

There is an example app inside the [`example` directory][example]. You can open the Android project to see an example on how the Adjust SDK can be integrated.

## <a id="basic-integration"></a>Basic integration

These are the minimal steps required to integrate the Adjust SDK into your Android project. We are going to assume that you use Android Studio for your Android development and target an Android API level 9 (Gingerbread) or later.

### <a id="sdk-add"></a>Add the SDK to your project

If you are using Maven, add the following to your `build.gradle` file:

```
compile 'com.adjust.sdk:adjust-android:4.12.0'
compile 'com.android.installreferrer:installreferrer:1.0'
```

**Note**: If you are using `Gradle 3.0.0 or above`, make sure to use the `implementation` keyword instead of `compile` as follows:

```
implementation 'com.adjust.sdk:adjust-android:4.12.0'
implementation 'com.android.installreferrer:installreferrer:1.0'
```

This applies when adding the Google Play Services dependency to your `build.gradle` file.

---

You can also add the Adjust SDK to your project as a JAR library. The latest SDK version's JAR library can be found on our [releases page][releases].

### <a id="sdk-gps"></a>Add Google Play Services

Since the 1st of August of 2014, apps in the Google Play Store must use the [Google Advertising ID][google_ad_id] to uniquely identify devices. To allow the Adjust SDK to use the Google Advertising ID, you must integrate the [Google Play Services][google_play_services]. If you haven't done this yet, follow these steps:

1. Open the `build.gradle` file of your app and find the `dependencies` block. Add the following line:

    ```
    compile 'com.google.android.gms:play-services-analytics:10.2.1'
    ```

    ![][gradle_gps]
    
    **Note**: The Adjust SDK is not tied to any specific version of the `play-services-analytics` part of the Google Play Services 
    library, so feel free to always use the latest version of it (or whichever you might need)

2. **Skip this step if you are using version 7 or later of Google Play Services**:
   In the Package Explorer open the `AndroidManifest.xml` of your Android
   project.  Add the following `meta-data` tag inside the `<application>`
   element.

    ```xml
    <meta-data android:name="com.google.android.gms.version"
               android:value="@integer/google_play_services_version" />
    ```

### <a id="sdk-permissions"></a>Add permissions

Please add the following permissions, which the Adjust SDK needs, if they are not already present in your `AndroidManifest.xml` file:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

If you are **not targeting the Google Play Store**, please also add the following permission:

```xml
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
```

### <a id="sdk-proguard"></a>Proguard settings

If you are using Proguard, add these lines to your Proguard file:

```
-keep public class com.adjust.sdk.** { *; }
-keep class com.google.android.gms.common.ConnectionResult {
    int SUCCESS;
}
-keep class com.google.android.gms.ads.identifier.AdvertisingIdClient {
    com.google.android.gms.ads.identifier.AdvertisingIdClient$Info getAdvertisingIdInfo(android.content.Context);
}
-keep class com.google.android.gms.ads.identifier.AdvertisingIdClient$Info {
    java.lang.String getId();
    boolean isLimitAdTrackingEnabled();
}
-keep class dalvik.system.VMRuntime {
    java.lang.String getRuntime();
}
-keep class android.os.Build {
    java.lang.String[] SUPPORTED_ABIS;
    java.lang.String CPU_ABI;
}
-keep class android.content.res.Configuration {
    android.os.LocaledList getLocales();
    java.util.Locale locale;
}
-keep class android.os.LocaledList {
    java.util.Locale get(int);
}
-keep public class com.android.installreferrer.** { *; }
```

If you are **not targeting the Google Play Store**, you can remove the `com.google.android.gms` rules.

### <a id="install-referrer"></a>Install referrer

In order to correctly attribute an install of your app to its source, Adjust needs information about the **install referrer**. This can be obtained by using the **Google Play Referrer API** or by catching the **Google Play Store intent** with a broadcast receiver.

**Important**: The Google Play Referrer API is newly introduced by Google with the express purpose of providing a more reliable and secure way of obtaining install referrer information and to aid attribution providers in the fight against click injection. It is **strongly advised** that you support this in your application. The Google Play Store intent is a less secure way of obtaining install referrer information. It will continue to exist in parallel with the new Google Play Referrer API temporarily, but it is set to be deprecated in future.

#### <a id="gpr-api"></a>Google Play Referrer API

In order to support this in your app, please make sure that you have followed the [Add the SDK to your project](#sdk-add) chapter properly and that you have following line added to your `build.gradle` file:

```
compile 'com.android.installreferrer:installreferrer:1.0'
```

Also, make sure that you have paid attention to the [Proguard settings](#sdk-proguard) chapter and that you have added all the rules mentioned in it, especially the one needed for this feature:

```
-keep public class com.android.installreferrer.** { *; }
```

This feature is supported if you are using **Adjust SDK v4.12.0 or above**.

#### <a id="gps-intent"></a>Google Play Store intent

The Google Play Store `INSTALL_REFERRER` intent should be captured with a broadcast receiver. If you are **not using your own broadcast receiver** to receive the `INSTALL_REFERRER` intent, add the following `receiver` tag inside the `application` tag in your `AndroidManifest.xml`.

```xml
<receiver
    android:name="com.adjust.sdk.AdjustReferrerReceiver"
    android:exported="true" >
    <intent-filter>
        <action android:name="com.android.vending.INSTALL_REFERRER" />
    </intent-filter>
</receiver>
```

![][receiver]

We use this broadcast receiver to retrieve the install referrer and pass it to our backend.

If you are already using a different broadcast receiver for the `INSTALL_REFERRER` intent, follow [these instructions][referrer] to add the Adjust broadcast receiver.

### <a id="sdk-integrate"></a>Integrate the SDK into your app

To start with, we'll set up basic session tracking.

### <a id="basic-setup"></a>Basic setup

We recommend using a global android [Application][android_application] class to initialize the SDK. If you don't have one in your app already, follow these steps:

1. Create a class that extends `Application`.
    ![][application_class]

2. Open the `AndroidManifest.xml` file of your app and locate the `<application>` element.
3. Add the attribute `android:name` and set it to the name of your new application class pefixed by a dot.

    In our example app we use an `Application` class named `GlobalApplication`, so the manifest file is configured as:
    ```xml
     <application
       android:name=".GlobalApplication"
       ... >
         ...
    </application>
    ```

    ![][manifest_application]

4. In your `Application` class find or create the `onCreate` method and add the following code to initialize the Adjust SDK:

    ```java
    import com.adjust.sdk.Adjust;
    import com.adjust.sdk.AdjustConfig;

    public class GlobalApplication extends Application {
        @Override
        public void onCreate() {
            super.onCreate();

            String appToken = "{YourAppToken}";
            String environment = AdjustConfig.ENVIRONMENT_SANDBOX;
            AdjustConfig config = new AdjustConfig(this, appToken, environment);
            Adjust.onCreate(config);
        }
    }
    ```

    ![][application_config]

    Replace `{YourAppToken}` with your app token. You can find this in your [dashboard].

    Depending on whether you are building your app for testing or for production, you must set `environment` with one of these values:

    ```java
    String environment = AdjustConfig.ENVIRONMENT_SANDBOX;
    String environment = AdjustConfig.ENVIRONMENT_PRODUCTION;
    ```

    **Important:** This value should be set to `AdjustConfig.ENVIRONMENT_SANDBOX` if and only if you or someone else is testing your app. Make sure to set the environment to `AdjustConfig.ENVIRONMENT_PRODUCTION` before you publish the app. Set it back to `AdjustConfig.ENVIRONMENT_SANDBOX` when you start developing and testing it again.

    We use this environment to distinguish between real traffic and test traffic from test devices. It is imperative that you keep this value meaningful at all times, especially if you are tracking revenue.

### <a id="session-tracking"></a>Session tracking

**Note**: This step is **really important** and please **make sure that you implement it properly in your app**. By implementing it, you will enable proper session tracking by the Adjust SDK in your app.

### <a id="session-tracking-api14"></a>API level 14 and higher

1. Add a private class that implements the `ActivityLifecycleCallbacks` interface. If you don't have access to this interface, your app is targeting an Android API level inferior to 14. You will have to update manually each Activity by following these [instructions](#session-tracking-api9). If you had `Adjust.onResume` and `Adjust.onPause` calls on each Activity of your app before, you should remove them.

    ![][activity_lifecycle_class]

2. Edit the `onActivityResumed(Activity activity)` method and add a call to `Adjust.onResume()`. Edit the
`onActivityPaused(Activity activity)` method and add a call to `Adjust.onPause()`.

    ![][activity_lifecycle_methods]

3. Add on the `onCreate()` method where the Adjust SDK is configured and add call  `registerActivityLifecycleCallbacks` with an instance of the created `ActivityLifecycleCallbacks` class.

    ```java
    import com.adjust.sdk.Adjust;
    import com.adjust.sdk.AdjustConfig;

    public class GlobalApplication extends Application {
        @Override
        public void onCreate() {
            super.onCreate();

            String appToken = "{YourAppToken}";
            String environment = AdjustConfig.ENVIRONMENT_SANDBOX;
            AdjustConfig config = new AdjustConfig(this, appToken, environment);
            Adjust.onCreate(config);

            registerActivityLifecycleCallbacks(new AdjustLifecycleCallbacks());

            //...
        }

         private static final class AdjustLifecycleCallbacks implements ActivityLifecycleCallbacks {
             @Override
             public void onActivityResumed(Activity activity) {
                 Adjust.onResume();
             }

             @Override
             public void onActivityPaused(Activity activity) {
                 Adjust.onPause();
             }

             //...
         }
      }
    ```

    ![][activity_lifecycle_register]

### <a id="session-tracking-api9"></a>API level between 9 and 13

If your app `minSdkVersion` in gradle is between `9` and `13`, consider updating it to at least `14` to simplify the integration process in the long term. Consult the official Android [dashboard][android-dashboard] to know the latest market share of the major versions.

To provide proper session tracking it is required to call certain Adjust SDK methods every time any Activity resumes or pauses. Otherwise the SDK might miss a session start or session end. In order to do so you should **follow these steps for each Activity of your app**:

1. Open the source file of your Activity.
2. Add the `import` statement at the top of the file.
3. In your Activity's `onResume` method call `Adjust.onResume()`. Create the method if needed.
4. In your Activity's `onPause` method call `Adjust.onPause()`. Create the method if needed.

After these steps your activity should look like this:

```java
import com.adjust.sdk.Adjust;
// ...
public class YourActivity extends Activity {
    protected void onResume() {
        super.onResume();
        Adjust.onResume();
    }
    protected void onPause() {
        super.onPause();
        Adjust.onPause();
    }
    // ...
}
```

![][activity]

Repeat these steps for **every** Activity of your app. Don't forget these steps when you create new Activities in the future. Depending on your coding style you might want to implement this in a common superclass of all your Activities.

### <a id="adjust-logging"></a>Adjust Logging

You can increase or decrease the amount of logs that you see during testing by calling `setLogLevel` on your `AdjustConfig` instance with one of the following parameters:

```java
config.setLogLevel(LogLevel.VERBOSE);   // enable all logging
config.setLogLevel(LogLevel.DEBUG);     // enable more logging
config.setLogLevel(LogLevel.INFO);      // the default
config.setLogLevel(LogLevel.WARN);      // disable info logging
config.setLogLevel(LogLevel.ERROR);     // disable warnings as well
config.setLogLevel(LogLevel.ASSERT);    // disable errors as well
config.setLogLevel(LogLevel.SUPRESS);   // disable all log output
```

In case you want all your log output to be disabled, beside setting the log level to suppress, you should also use  constructor for `AdjustConfig` object which gets boolean parameter indicating whether suppress log level should be supported  or not:

```java
String appToken = "{YourAppToken}";
String environment = AdjustConfig.ENVIRONMENT_SANDBOX;

AdjustConfig config = new AdjustConfig(this, appToken, environment, true);
config.setLogLevel(LogLevel.SUPRESS);

Adjust.onCreate(config);
```

### <a id="build-the-app"></a>Build your app

Build and run your Android app. In your `LogCat` viewer you can set the filter `tag:Adjust` to hide all other logs. After your app has launched you should see the following Adjust log: `Install tracked`

![][log_message]

## Additional Features

Once you have integrated the Adjust SDK into your project, you can take advantage of the following features.

### <a id="event-tracking"></a>Event tracking

You can use adjust to track any event in your app. Suppose you want to track every tap on a button. You would have to create a new event token in your [dashboard]. Let's say that event token is `abc123`. In your button's `onClick` method you could then add the following lines to track the click:

```java
AdjustEvent event = new AdjustEvent("abc123");
Adjust.trackEvent(event);
```

### <a id="revenue-tracking"></a>Revenue tracking

If your users can generate revenue by tapping on advertisements or making in-app purchases you can track those revenues with events. Lets say a tap is worth one Euro cent. You could then track the revenue event like this:

```java
AdjustEvent event = new AdjustEvent("abc123");
event.setRevenue(0.01, "EUR");
Adjust.trackEvent(event);
```

This can be combined with callback parameters of course.

When you set a currency token, Adjust will automatically convert the incoming revenues into a reporting revenue of your choice. Read more about [currency conversion here][currency-conversion].

You can read more about revenue and event tracking in the [event tracking guide][event-tracking].

### <a id="revenue-deduplication"></a>Revenue deduplication

You can also add an optional order ID to avoid tracking duplicate revenues. The last ten order IDs are remembered, and revenue events with duplicate order IDs are skipped. This is especially useful for in-app purchase tracking. You can see an  example below.

If you want to track in-app purchases, please make sure to call `trackEvent` only if the purchase is finished and item is purchased. That way you can avoid tracking revenue that is not actually being generated.

```java
AdjustEvent event = new AdjustEvent("abc123");

event.setRevenue(0.01, "EUR");
event.setOrderId("{OrderId}");

Adjust.trackEvent(event);
```

### <a id="iap-verification"></a>In-App Purchase verification

If you want to check the validity of In-App Purchases made in your app using Purchase Verification, Adjust's server side receipt verification tool, then check out our Android purchase SDK to read more about it [here][android-purchase-verification].

### <a id="callback-parameters"></a>Callback parameters

You can register a callback URL for your events in your [dashboard]. We will send a GET request to that URL whenever the event is tracked. You can add callback parameters to that event by calling `addCallbackParameter` to the event instance before tracking it. We will then append these parameters to your callback URL.

For example, suppose you have registered the URL `http://www.adjust.com/callback` then track an event like this:

```java
AdjustEvent event = new AdjustEvent("abc123");

event.addCallbackParameter("key", "value");
event.addCallbackParameter("foo", "bar");

Adjust.trackEvent(event);
```

In that case we would track the event and send a request to:

```
http://www.adjust.com/callback?key=value&foo=bar
```

It should be mentioned that we support a variety of placeholders like `{gps_adid}` that can be used as parameter values. In the resulting callback this placeholder would be replaced with the Google Play Services ID of the current device. Also note that we don't store any of your custom parameters, but only append them to your callbacks. If you haven't registered a callback for an event, these parameters won't even be read.

You can read more about using URL callbacks, including a full list of available values, in our [callbacks guide][callbacks-guide].

### <a id="partner-parameters"></a>Partner parameters

You can also add parameters to be transmitted to network partners, which have been activated in your Adjust dashboard.

This works similarly to the callback parameters mentioned above, but can be added by calling the `addPartnerParameter` method on your `AdjustEvent` instance.

```java
AdjustEvent event = new AdjustEvent("abc123");

event.addPartnerParameter("key", "value");
event.addPartnerParameter("foo", "bar");

Adjust.trackEvent(event);
```

You can read more about special partners and these integrations in our [guide to special partners][special-partners].

### <a id="session-parameters"></a>Set up session parameters

Some parameters are saved to be sent in every **event** and **session** of the Adjust SDK. Once you have added any of these parameters, you don't need to add them every time, since they will be saved locally. If you add the same parameter twice, there will be no effect.

These session parameters can be called before the Adjust SDK is launched to make sure they are sent even on install. If you need to send them with an install, but can only obtain the needed values after launch, it's possible to [delay](#delay-start) the first launch of the Adjust SDK to allow this behaviour.

### <a id="session-callback-parameters"></a>Session callback parameters

The same callback parameters that are registered for [events](#callback-parameters) can be also saved to be sent in every  event or session of the Adjust SDK.

The session callback parameters have a similar interface to the event callback parameters. Instead of adding the key and it's value to an event, it's added through a call to `Adjust.addSessionCallbackParameter(String key, String value)`:

```java
Adjust.addSessionCallbackParameter("foo", "bar");
```

The session callback parameters will be merged with the callback parameters added to an event. The callback parameters added to an event have precedence over the session callback parameters. Meaning that, when adding a callback parameter to an event with the same key to one added from the session, the value that prevails is the callback parameter added to the event.

It's possible to remove a specific session callback parameter by passing the desiring key to the method `Adjust.removeSessionCallbackParameter(String key)`.

```java
Adjust.removeSessionCallbackParameter("foo");
```

If you wish to remove all keys and their corresponding values from the session callback parameters, you can reset it with the method `Adjust.resetSessionCallbackParameters()`.

```java
Adjust.resetSessionCallbackParameters();
```

### <a id="session-partner-parameters"></a>Session partner parameters

In the same way that there are [session callback parameters](#session-callback-parameters) sent in every event or session of the Adjust SDK, there is also session partner parameters.

These will be transmitted to network partners, for the integrations that have been activated in your Adjust [dashboard].

The session partner parameters have a similar interface to the event partner parameters. Instead of adding the key and it's value to an event, it's added through a call to `Adjust.addSessionPartnerParameter(String key, String value)`:

```java
Adjust.addSessionPartnerParameter("foo", "bar");
```

The session partner parameters will be merged with the partner parameters added to an event. The partner parameters added to an event have precedence over the session partner parameters. Meaning that, when adding a partner parameter to an event with the same key to one added from the session, the value that prevails is the partner parameter added to the event.

It's possible to remove a specific session partner parameter by passing the desiring key to the method `Adjust.removeSessionPartnerParameter(String key)`.

```java
Adjust.removeSessionPartnerParameter("foo");
```

If you wish to remove all keys and their corresponding values from the session partner parameters, you can reset it with the method `Adjust.resetSessionPartnerParameters()`.

```java
Adjust.resetSessionPartnerParameters();
```

### <a id="delay-start"></a>Delay start

Delaying the start of the Adjust SDK allows your app some time to obtain session parameters, such as unique identifiers, to be sent on install.

Set the initial delay time in seconds with the method `setDelayStart` in the `AdjustConfig` instance:

```java
adjustConfig.setDelayStart(5.5);
```

In this case, this will make the Adjust SDK not send the initial install session and any event created for 5.5 seconds. After this time is expired or if you call `Adjust.sendFirstPackages()` in the meanwhile, every session parameter will be added to the delayed install session and events and the Adjust SDK will resume as usual.

**The maximum delay start time of the adjust SDK is 10 seconds**.

### <a id="attribution-callback"></a>Attribution callback

You can register a listener to be notified of tracker attribution changes. Due to the different sources considered for attribution, this information can not be provided synchronously. The simplest way is to create a single anonymous listener:

Please make sure to consider our [applicable attribution data policies][attribution-data].

With the `AdjustConfig` instance, before starting the SDK, add the anonymous listener:

```java
AdjustConfig config = new AdjustConfig(this, appToken, environment);

config.setOnAttributionChangedListener(new OnAttributionChangedListener() {
    @Override
    public void onAttributionChanged(AdjustAttribution attribution) {
    }
});

Adjust.onCreate(config);
```

Alternatively, you could implement the `OnAttributionChangedListener` interface in your `Application` class and set it as listener:

```java
AdjustConfig config = new AdjustConfig(this, appToken, environment);
config.setOnAttributionChangedListener(this);
Adjust.onCreate(config);
```

The listener function will be called after the SDK receives the final attribution data. Within the listener function you have access to the `attribution` parameter. Here is a quick summary of its properties:

- `String trackerToken` the tracker token of the current attribution.
- `String trackerName` the tracker name of the current attribution.
- `String network` the network grouping level of the current attribution.
- `String campaign` the campaign grouping level of the current attribution.
- `String adgroup` the ad group grouping level of the current attribution.
- `String creative` the creative grouping level of the current attribution.
- `String clickLabel` the click label of the current attribution.
- `String adid` the Adjust device identifier.

If any value is unavailable, it will default to `null`.

### <a id="session-event-callbacks"></a>Session and event callbacks

You can register a listener to be notified when events or sessions are tracked. There are four listeners: one for tracking successful events, one for tracking failed events, one for tracking successful sessions and one for tracking failed sessions. You can add any number of listeners after creating the `AdjustConfig` object:

```java
AdjustConfig config = new AdjustConfig(this, appToken, environment);

// Set event success tracking delegate.
config.setOnEventTrackingSucceededListener(new OnEventTrackingSucceededListener() {
    @Override
    public void onFinishedEventTrackingSucceeded(AdjustEventSuccess eventSuccessResponseData) {
        // ...
    }
});

// Set event failure tracking delegate.
config.setOnEventTrackingFailedListener(new OnEventTrackingFailedListener() {
    @Override
    public void onFinishedEventTrackingFailed(AdjustEventFailure eventFailureResponseData) {
        // ...
    }
});

// Set session success tracking delegate.
config.setOnSessionTrackingSucceededListener(new OnSessionTrackingSucceededListener() {
    @Override
    public void onFinishedSessionTrackingSucceeded(AdjustSessionSuccess sessionSuccessResponseData) {
        // ...
    }
});

// Set session failure tracking delegate.
config.setOnSessionTrackingFailedListener(new OnSessionTrackingFailedListener() {
    @Override
    public void onFinishedSessionTrackingFailed(AdjustSessionFailure sessionFailureResponseData) {
        // ...
    }
});

Adjust.onCreate(config);
```

The listener function will be called after the SDK tries to send a package to the server. Within the listener function you have access to a response data object specifically for the listener. Here is a quick summary of the success session response data object fields:

- `String message` the message from the server or the error logged by the SDK.
- `String timestamp` timestamp from the server.
- `String adid` a unique device identifier provided by Adjust.
- `JSONObject jsonResponse` the JSON object with the reponse from the server.

Both event response data objects contain:

- `String eventToken` the event token, if the package tracked was an event.

If any value is unavailable, it will default to `null`.

And both event and session failed objects also contain:

- `boolean willRetry` indicates that will be an attempt to resend the package at a later time.

### <a id="disable-tracking"></a>Disable tracking

You can disable the Adjust SDK from tracking any activities of the current device by calling `setEnabled` with parameter `false`. **This setting is remembered between sessions**.

```java
Adjust.setEnabled(false);
```

You can check if the Adjust SDK is currently enabled by calling the function `isEnabled`. It is always possible to activatе the Adjust SDK by invoking `setEnabled` with the enabled parameter as `true`.

### <a id="offline-mode"></a>Offline mode

You can put the Adjust SDK in offline mode to suspend transmission to our servers, while retaining tracked data to be sent later. While in offline mode, all information is saved in a file, so be careful not to trigger too many events while in offline mode.

You can activate offline mode by calling `setOfflineMode` with the parameter `true`.

```java
Adjust.setOfflineMode(true);
```

Conversely, you can deactivate offline mode by calling `setOfflineMode` with `false`. When the Adjust SDK is put back in online mode, all saved information is sent to our servers with the correct time information.

Unlike disabling tracking, this setting is **not remembered** between sessions. This means that the SDK is in online mode whenever it is started, even if the app was terminated in offline mode.

### <a id="event-buffering"></a>Event buffering

If your app makes heavy use of event tracking, you might want to delay some HTTP requests in order to send them in one batch every minute. You can enable event buffering with your `AdjustConfig` instance:

```java
AdjustConfig config = new AdjustConfig(this, appToken, environment);

config.setEventBufferingEnabled(true);

Adjust.onCreate(config);
```

### <a id="background-tracking"></a>Background tracking

The default behaviour of the Adjust SDK is to pause sending HTTP requests while the app is in the background. You can change this in your `AdjustConfig` instance:

```java
AdjustConfig config = new AdjustConfig(this, appToken, environment);

config.setSendInBackground(true);

Adjust.onCreate(config);
```

### <a id="device-ids"></a>Device IDs

The Adjust SDK offers you possibility to obtain some of the device identifiers.

### <a id="di-gps-adid"></a>Google Play Services advertising identifier

Certain services (such as Google Analytics) require you to coordinate Device and Client IDs in order to prevent duplicate reporting.

If you need to obtain the Google Advertising ID, there is a restriction that only allows it to be read in a background thread. If you call the function `getGoogleAdId` with the context and a `OnDeviceIdsRead` instance, it will work in any situation:

```java
Adjust.getGoogleAdId(this, new OnDeviceIdsRead() {
    @Override
    public void onGoogleAdIdRead(String googleAdId) {
        // ...
    }
});
```

Inside the method `onGoogleAdIdRead` of the `OnDeviceIdsRead` instance, you will have access to Google Advertising ID as the variable `googleAdId`.

### <a id="di-amz-adid"></a>Amazon advertising identifier

If you need to obtain the Amazon Advertising ID, you can make a call to following method on `Adjust` instance:

```java
String amazonAdId = Adjust.getAmazonAdId(context);
```

### <a id="di-adid"></a>Adjust device identifier

For each device with your app installed on it, Adjust backend generates unique **Adjust device identifier** (**adid**). In order to obtain this identifier, you can make a call to following method on `Adjust` instance:

```java
String adid = Adjust.getAdid();
```

**Note**: You can only make this call in the Adjust SDK v4.11.0 and above.

**Note**: Information about **adid** is available after app installation has been tracked by the Adjust backend. From that moment on, Adjust SDK has information about your device **adid** and you can access it with this method. So, **it is not possible** to access **adid** value before the SDK has been initialised and installation of your app was tracked successfully.

### <a id="user-attribution"></a>User attribution

Like described in [attribution callback section](#attribution-callback), this callback get triggered providing you info about new attribution when ever it changes. In case you want to access info about your user's current attribution when ever you need it, you can make a call to following method of the `Adjust` instance:

```java
AdjustAttribution attribution = Adjust.getAttribution();
```

**Note**: You can only make this call in the Adjust SDK v4.11.0 and above.

**Note**: Information about current attribution is available after app installation has been tracked by the Adjust backend and attribution callback has been initially triggered. From that moment on, Adjust SDK has information about your user's attribution and you can access it with this method. So, **it is not possible** to access user's attribution value before the SDK has been initialized and attribution callback has been initially triggered.

### <a id="push-token"></a>Push token

To send us the push notification token, add the following call to Adjust once you have obtained your token or when ever it's value is changed:

```java
Adjust.setPushToken(pushNotificationsToken, context);
```

This updated signature with `context` added allows the SDK to cover more scenarios to make sure that the push token is sent, and it is advised that you use the signature method above.

We still support the previous signature of the same method:

```java
Adjust.setPushToken(pushNotificationsToken);
```

Push tokens are used for Audience Builder and client callbacks, and they are required for the upcoming uninstall tracking feature.

### <a id="pre-installed-trackers"></a>Pre-installed trackers

If you want to use the Adjust SDK to recognize users whose devices came with your app pre-installed, follow these steps.

1. Create a new tracker in your [dashboard].
2. Open your app delegate and add set the default tracker of your `AdjustConfig`:

  ```objc
  AdjustConfig config = new AdjustConfig(this, appToken, environment);
  config.setDefaultTracker("{TrackerToken}");
  Adjust.onCreate(config);
  ```

  Replace `{TrackerToken}` with the tracker token you created in step 1. Please note that the Dashboard displays a tracker URL (including `http://app.adjust.com/`). In your source code, you should specify only the six-character token and not the 
  entire URL.

3. Build and run your app. You should see a line like the following in your LogCat:

    ```
    Default tracker: 'abc123'
    ```

### <a id="deeplinking"></a>Deep linking

If you are using an Adjust tracker URL with the option to deep link into your app, there is the possibility to get information about the deep link URL and its content. Hitting the URL can happen when the user has your app already installed (standard deep linking scenario) or if they don't have the app on their device (deferred deep linking scenario). In the standard deep linking scenario, the Android platform natively offers the possibility for you to get the information about the deep link content. The deferred deep linking scenario is something which the Android platform doesn't support out of the box, and, in this case, the Adjust SDK will offer you the mechanism you need to get the information about the deep link content.

### <a id="deeplinking-standard"></a>Standard deep linking scenario

If a user has your app installed and you want it to launch after hitting an Adjust tracker URL with the `deep_link` parameter in it, you need to enable deep linking in your app. This is done by choosing a desired **unique scheme name** and assigning it to the Activity you want to launch once your app opens following a user clicking on the tracker URL. This is set in the `AndroidManifest.xml`. You need to add the `intent-filter` section to your desired Activity definition in the manifest file and assign an `android:scheme` property value with the desired scheme name:

```xml
<activity
    android:name=".MainActivity"
    android:configChanges="orientation|keyboardHidden"
    android:label="@string/app_name"
    android:screenOrientation="portrait">

    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>

    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="adjustExample" />
    </intent-filter>
</activity>
```

With this now set, you need to use the assigned scheme name in the Adjust tracker URL's `deep_link` parameter if you want your app to launch once the tracker URL is clicked. A tracker URL without any information added to the deep link can be built to look something like this:

```
https://app.adjust.com/abc123?deep_link=adjustExample%3A%2F%2F
```

Please, have in mind that the `deep_link` parameter value in the URL **must be URL encoded**.

After clicking this tracker URL, and with the app set as described above, your app will launch along with the `MainActivity` intent. Inside the `MainActivity` class, you will automatically be provided with the information about the `deep_link` parameter content. Once this content is delivered to you, it **will not be encoded**, although it was encoded in the URL.

Depending on the `android:launchMode` setting of your Activity in the `AndroidManifest.xml` file, information about the `deep_link` parameter content will be delivered to the appropriate place in the Activity file. For more information about the possible values of the `android:launchMode` property, check [the official Android documentation][android-launch-modes].

There are two places within your desired Activity where information about the deep link content will be delivered via the `Intent` object--either in the Activity's `onCreate` or `onNewIntent` methods. Once your app has launched and one of these methods has been triggered, you will be able to get the actual deep link passed in the `deep_link` parameter in the click URL. You can then use this information to conduct some additional logic in your app.

You can extract the deep link content from these two methods like this:

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Intent intent = getIntent();
    Uri data = intent.getData();

    // data.toString() -> This is your deep_link parameter value.
}
```

```java
@Override
protected void onNewIntent(Intent intent) {
    super.onNewIntent(intent);

    Uri data = intent.getData();

    // data.toString() -> This is your deep_link parameter value.
}
```

### <a id="deeplinking-deferred"></a>Deferred deep linking scenario

The deferred deep linking scenario occurs when a user clicks on an Adjust tracker URL with a `deep_link` parameter in it, but does not have the app installed on the device at click time. After that, the user will be redirected to the Play Store to download and install your app. After opening it for the first time, the `deep_link` parameter content will be delivered to your app.

In order to get information about the `deep_link` parameter content in a deferred deep linking scenario, you should set a listener method on the `AdjustConfig` object. This will be triggered once the Adjust SDK gets the information about the deep link content from the Adjust backend.

```java
AdjustConfig config = new AdjustConfig(this, appToken, environment);

// Evaluate the deeplink to be launched.
config.setOnDeeplinkResponseListener(new OnDeeplinkResponseListener() {
    @Override
    public boolean launchReceivedDeeplink(Uri deeplink) {
        // ...
        if (shouldAdjustSdkLaunchTheDeeplink(deeplink)) {
            return true;
        } else {
            return false;
        }
    }
});

Adjust.onCreate(config);
```

Once the Adjust SDK receives the information about the deep link content from the Adjust backend, it will deliver you the information about its content in this listener and expect the `boolean` return value from you. This return value represents your decision on whether the Adjust SDK should launch the Activity to which you have assigned the scheme name from the deep link (like in the standard deep linking scenario) or not.

If you return `true`, we will launch it and the exact same scenario which is described in the [Standard deep linking scenario chapter](#deeplinking-standard) will happen. If you do not want the SDK to launch the Activity, you can return `false` from this listener, and, based on the deep link content, decide on your own what to do next in your app.

### <a id="deeplinking-reattribution"></a>Reattribution via deep links

Adjust enables you to run re-engagement campaigns through deep links. For more information on how to do that, please check our [official docs][reattribution-with-deeplinks].

If you are using this feature, in order for your user to be properly reattributed, you need to make one additional call to the Adjust SDK in your app.

Once you have received deep link content information in your app, add a call to the `Adjust.appWillOpenUrl` method. By making this call, the Adjust SDK will try to find if there is any new attribution information inside of the deep link. If there is any, it will be sent to the Adjust backend. If your user should be reattributed due to a click on the adjust tracker URL with deep link content, you will see the [attribution callback](#attribution-callback) in your app being triggered with new attribution info for this user.

The call to `Adjust.appWillOpenUrl` should be done like this:

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Intent intent = getIntent();
    Uri data = intent.getData();

    Adjust.appWillOpenUrl(data);
}
```

```java
@Override
protected void onNewIntent(Intent intent) {
    super.onNewIntent(intent);

    Uri data = intent.getData();

    Adjust.appWillOpenUrl(data);
}
```

## <a id="troubleshooting"></a>Troubleshooting

### <a id="ts-session-failed"></a>I'm seeing the "Session failed (Ignoring too frequent session. ...)" error.

This error typically occurs when testing installs. Uninstalling and reinstalling the app is not enough to trigger a new install. The servers will determine that the SDK has lost its locally aggregated session data and ignore the erroneous message, given the information available on the servers about the device.

This behaviour can be cumbersome during tests, but is necessary in order to have the sandbox behaviour match production as much as possible.

You can reset the session data of the device in our servers. Check the error message in the logs:

```
Session failed (Ignoring too frequent session. Last session: YYYY-MM-DDTHH:mm:ss, this session: YYYY-MM-DDTHH:mm:ss, interval: XXs, min interval: 20m) (app_token: {yourAppToken}, adid: {adidValue})
```

With the `{yourAppToken}` and `{adidValue}`/`{gps_adidValue}`/`{androidIDValue}` values filled in below, open one of the 
following links:


```
http://app.adjust.com/forget_device?app_token={yourAppToken}&adid={adidValue}
```

```
http://app.adjust.com/forget_device?app_token={yourAppToken}&gps_adid={gps_adidValue}
```

```
http://app.adjust.com/forget_device?app_token={yourAppToken}&android_id={androidIDValue}
```

When the device is forgotten, the link just returns `Forgot device`. If the device was already forgotten or the values were incorrect, the link returns `Device not found`.

### <a id="ts-broadcast-receiver"></a>Is my broadcast receiver capturing the install referrer?

If you followed the instructions in the [guide](#broadcast_receiver), the broadcast receiver should be configured to send the install referrer to our SDK and to our servers.

You can test this by triggering a test install referrer manually. Replace `com.your.appid` with your app ID and run the following command with the [adb](http://developer.android.com/tools/help/adb.html) tool that comes with Android Studio:

```
adb shell am broadcast -a com.android.vending.INSTALL_REFERRER -n com.your.appid/com.adjust.sdk.AdjustReferrerReceiver --es "referrer" "adjust_reftag%3Dabc1234%26tracking_id%3D123456789%26utm_source%3Dnetwork%26utm_medium%3Dbanner%26utm_campaign%3Dcampaign"
```

If you are already using a different broadcast receiver for the `INSTALL_REFERRER` intent and followed this [guide][referrer], replace `com.adjust.sdk.AdjustReferrerReceiver` with your broadcast receiver.

You can also remove the `-n com.your.appid/com.adjust.sdk.AdjustReferrerReceiver` parameter so that all the apps in the device will receive the `INSTALL_REFERRER` intent.

If you set the log level to `verbose`, you should be able to see the log from reading the referrer:

```
V/Adjust: Reading query string (adjust_reftag=abc1234&tracking_id=123456789&utm_source=network&utm_medium=banner&utm_campaign=campaign) from reftag
```

And a click package added to the SDK's package handler:

```
V/Adjust: Path:      /sdk_click
    ClientSdk: android4.6.0
    Parameters:
      app_token        abc123abc123
      click_time       yyyy-MM-dd'T'HH:mm:ss.SSS'Z'Z
      created_at       yyyy-MM-dd'T'HH:mm:ss.SSS'Z'Z
      environment      sandbox
      gps_adid         12345678-0abc-de12-3456-7890abcdef12
      needs_attribution_data 1
      referrer         adjust_reftag=abc1234&tracking_id=123456789&utm_source=network&utm_medium=banner&utm_campaign=campaign
      reftag           abc1234
      source           reftag
      tracking_enabled 1
```

If you perform this test before launching the app, you won't see the package being sent. The package will be sent once the app is launched.

### <a id="ts-event-at-launch"></a>Can I trigger an event at application launch?

Not how you might intuitively think. The `onCreate` method on the global `Application` class is called not only at application launch, but also when a system or application event is captured by the app.

Our SDK is prepared for initialization at this time, but not actually started. This will only happen when an activity is started, i.e., when a user actually launches the app.

That's why triggering an event at this time will not do what you would expect. Such calls will start the Adjust SDK and send the events, even when the app was not launched by the user - at a time that depends on external factors of the app.

Triggering events at application launch will thus result in inaccuracies in the number of installs and sessions tracked.

If you want to trigger an event after the install, use the [attribution callback](#attribution-callback).

If you want to trigger an event when the app is launched, use the `onCreate` method of the Activity which is started.

[dashboard]:                      http://adjust.com
[adjust.com]:                     http://adjust.com

[maven]:                          http://maven.org
[example]:                        https://github.com/adjust/android_sdk/tree/master/Adjust/example
[releases]:                       https://github.com/adjust/adjust_android_sdk/releases
[referrer]:                       doc/english/referrer.md
[google_ad_id]:                   https://support.google.com/googleplay/android-developer/answer/6048248?hl=en
[event-tracking]:                 https://docs.adjust.com/en/event-tracking
[callbacks-guide]:                https://docs.adjust.com/en/callbacks
[new-referrer-api]:               https://developer.android.com/google/play/installreferrer/library.html
[application_name]:               http://developer.android.com/guide/topics/manifest/application-element.html#nm
[special-partners]:               https://docs.adjust.com/en/special-partners
[attribution-data]:               https://github.com/adjust/sdks/blob/master/doc/attribution-data.md
[android-dashboard]:              http://developer.android.com/about/dashboards/index.html
[currency-conversion]:            https://docs.adjust.com/en/event-tracking/#tracking-purchases-in-different-currencies
[android_application]:            http://developer.android.com/reference/android/app/Application.html
[android-launch-modes]:           https://developer.android.com/guide/topics/manifest/activity-element.html
[google_play_services]:           http://developer.android.com/google/play-services/setup.html
[activity_resume_pause]:          doc/activity_resume_pause.md
[reattribution-with-deeplinks]:   https://docs.adjust.com/en/deeplinking/#manually-appending-attribution-data-to-a-deep-link
[android-purchase-verification]:  https://github.com/adjust/android_purchase_sdk

[activity]:                     https://raw.github.com/adjust/sdks/master/Resources/android/v4/14_activity.png
[proguard]:                     https://raw.github.com/adjust/sdks/master/Resources/android/v4/08_proguard_new.png
[receiver]:                     https://raw.github.com/adjust/sdks/master/Resources/android/v4/09_receiver.png
[gradle_gps]:                   https://raw.github.com/adjust/sdks/master/Resources/android/v4/05_gradle_gps.png
[log_message]:                  https://raw.github.com/adjust/sdks/master/Resources/android/v4/15_log_message.png
[manifest_gps]:                 https://raw.github.com/adjust/sdks/master/Resources/android/v4/06_manifest_gps.png
[gradle_adjust]:                https://raw.github.com/adjust/sdks/master/Resources/android/v4/04_gradle_adjust.png
[import_module]:                https://raw.github.com/adjust/sdks/master/Resources/android/v4/01_import_module.png
[select_module]:                https://raw.github.com/adjust/sdks/master/Resources/android/v4/02_select_module.png
[imported_module]:              https://raw.github.com/adjust/sdks/master/Resources/android/v4/03_imported_module.png
[application_class]:            https://raw.github.com/adjust/sdks/master/Resources/android/v4/11_application_class.png
[application_config]:           https://raw.github.com/adjust/sdks/master/Resources/android/v4/13_application_config.png
[manifest_permissions]:         https://raw.github.com/adjust/sdks/master/Resources/android/v4/07_manifest_permissions.png
[manifest_application]:         https://raw.github.com/adjust/sdks/master/Resources/android/v4/12_manifest_application.png
[activity_lifecycle_class]:     https://raw.github.com/adjust/sdks/master/Resources/android/v4/16_activity_lifecycle_class.png
[activity_lifecycle_methods]:   https://raw.github.com/adjust/sdks/master/Resources/android/v4/17_activity_lifecycle_methods.png
[activity_lifecycle_register]:  https://raw.github.com/adjust/sdks/master/Resources/android/v4/18_activity_lifecycle_register.png

## <a id="license"></a>License

The Adjust SDK is licensed under the MIT License.

Copyright (c) 2012-2017 Adjust GmbH, http://www.adjust.com

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
of the Software, and to permit persons to whom the Software is furnished to do
so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
