[logsene]: https://sematext.com/logsene/
[register]: https://apps.sematext.com/ui/registration
[hosted-kibana]: https://sematext.com/docs/logs/kibana/
[video-tutorials]: https://www.elastic.co/blog/kibana-4-video-tutorials-part-1

Logsene for Android Applications
================================

[![](https://jitpack.io/v/sematext/sematext-logsene-android.svg)](https://jitpack.io/#sematext/sematext-logsene-android)
[![License](https://img.shields.io/github/license/sematext/sematext-logsene-android.svg)](https://img.shields.io/github/license/sematext/sematext-logsene-android.svg)

[Logsene is ELK as a Service][logsene]. This library lets you collect **mobile analytics** and **log data** from your Android applications using Logsene. If you don't have a Logsene account, you can [register for free][register] to get your app token.

Getting Started
===============

If you haven't already, register for a free account. Create a new Logsene app to get the app token, you will need it later.

Add the following gradle dependency to your android application:

```
allprojects {
 repositories {
    jcenter()
    maven { url "https://jitpack.io" }
 }
}

dependencies {
    compile 'com.github.sematext:sematext-logsene-android:2.1.0'
}
```

The library sends data to Logsene servers, so you will need to add the `INTERNET` and `ACCESS_NETWORK_STATE` permissions to your application manifest.

```xml
<uses-permission android:name="android.permission.INTERNET"></uses-permission>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"></uses-permission>
```

The library sends data in batches to preserve battery (every 15 minutes), or if
there are more than 10 events queued up. Events are saved while the device is
offline so you don't have to worry about losing any data. By default the
library keeps up to 5,000 events while offline.

Add the following inside the application manifest (inside `<application>`):

```xml
 <meta-data
   android:name="LogseneAppToken"
   android:value="yourtoken" />
 <meta-data
   android:name="LogseneType"
   android:value="example" />

 <!-- For EU region use https://logsene-receiver.eu.sematext.com as the receiverUrl -->
 <meta-data
   android:name="LogseneReceiverUrl"
   android:value="https://logsene-receiver.sematext.com" />
```

 * **LogseneAppToken (required)**: This is your Logsene application token, you should have received one after registering and creating your Logsene app.
 We **highly recommend** creating a write-only token in your app settings to prevent any unauthorized access to your logs.
 * **LogseneType (required)**: Type to be used for all events (Logsene uses Elasticsearch compatible API)
 * **LogseneMaxOfflineMessages**: Maximum number of offline stored events. Events are stored on the device while it's offline, or if the library is unable to send them to Logsene for some reason.
 * **LogseneReceiverUrl**: If you are using Logsene On Premises, you can put your Logsene Receiver URL here. For EU region please use https://logsene-receiver.eu.sematext.com as the `receiverUrl`.
 * **LogseneMinTimeDelay**: Minimum amount of time to wait between sending logs while application is running and creating new log messages
 * **LogseneInterval**: time interval for sending logs regardless of app being active (minimum 15 minutes)
 * **LogseneRequiresUnmeteredNetwork**: if logs should be shipped only on unmetered network connection
 * **LogseneRequiresDeviceIdle**: if logs should be shipped only when device is idle
 * **LogseneRequiresBatteryNotLow**: if logs should be shipped only when battery is not low

Example Application
-------------------

To see how some basic use cases are actually implemented, checkout the bundled `TestApp` android application. Make sure to set your own application token in the android manifest.

**Note** that it's highly recommended that you use one instance of Logsene at any time in your app.

Mobile Application Analytics
----------------------------

With Logsene you get Elasticsearch and Kibana out of the box, which makes it great for mobile analytics. Once you've setup the Logsene service, it's trivial to start sending custom events. For example, you may want to send an event every time a user starts an activity. In that case you could put the following inside the `onCreate()` method:

```java
try {
    JSONObject event = new JSONObject();
    event.put("activity", this.getClass().getSimpleName());
    event.put("action", "started");
    Logsene logsene = new Logsene(this);
    logsene.event(event);
} catch (JSONException e) {
    Log.e("myapp", "Unable to construct json", e);
}
```

To visualize the collected data, you would use the [integrated Kibana dashboard][hosted-kibana].

If you don't see the events in the dashboard immediately, note that data are sent in batches to preserve the battery (every 60s), or if there are more than 10 events queued up. Events are saved while the device is offline, so you don't have to worry about losing any data.

When it comes to the structure of your events, you are free to choose your own, the above is just an example. You can use any number of fields, and you can use nested fields. Basically, any valid JSON object will work fine. Note that the `meta` field is reserved for meta information (see [Meta Fields](#meta-fields) below). If you set a value for this field when sending an event, the meta information will not be included for that event.

Meta Fields
-----------

Meta data are added to each event, all stored within the "meta" field:

 * versionName (as defined in your build.gradle)
 * versionCode (as defined in your build.gradle)
 * osRelease (android version)
 * uuid (unique identifier for this app installation)

You can set your own meta fields with `Logsene.setDefaultMeta()`. For example:

```java
try {
    JSONObject meta = new JSONObject();
    meta.put("user", "user@example.com");
    Logsene.setDefaultMeta(meta);
} catch (JSONException e) {
    Log.e("myapp", "Unable to construct json", e);
}
```

Note that these meta fields are global, and will be attached to every event sent to Logsene.

Centralized Logging
-------------------

The library offers some basic functions for centralized logging:

- Logsene.debug(String)
- Logsene.info(String)
- Logsene.warn(String)
- Logsene.warn(Throwable)
- Logsene.error(String)
- Logsene.error(Throwable)

For integrating with existing logging frameworks, see below.

### JUL

If your application uses JUL (java.util.logging) loggers, you can use the provided custom Handler for Logsene. You will need to configure it through code, since we need a reference to the `Context` object. If you configure your loggers to use the `LogseneHandler`, all log messages will be sent to Logsene for centralized logging.

```java
Logsene logsene = new Logsene(context);
Logger logger = Logger.getLogger("mylogger");
logger.addHandler(new LogseneHandler(logsene));
```

### Logging exceptions

If you use JUL and the `LogseneHandler`, all logged exceptions will be sent to Logsene, no further configuration is needed. However, if you don't use JUL, the library provides a helper method to log exceptions:

```java
Logsene logsene = new Logsene(context);
try {
    trySomeOperation();
} catch (IOException e) {
    logsene.error(e);
}
```

### How to log uncaught exceptions (crashes)

You can log any uncaught exceptions by defining your own uncaught exception handler. You will need to define a custom `Application` class for this to work. For example:

```java
public class TestApplication extends Application {
    private Thread.UncaughtExceptionHandler defaultExceptionHandler;
    private Thread.UncaughtExceptionHandler exceptionHandler = new Thread.UncaughtExceptionHandler() {
        @Override
        public void uncaughtException(Thread thread, Throwable ex) {
            // Send uncaught exception to Logsene.
            Logsene logsene = new Logsene(TestApplication.this);
            logsene.error(ex);

            // Run the default android handler if one is set
            if (defaultExceptionHandler != null) {
                defaultExceptionHandler.uncaughtException(thread, ex);
            }
        }
    };

    public TestApplication() {
        defaultExceptionHandler = Thread.getDefaultUncaughtExceptionHandler();
        Thread.setDefaultUncaughtExceptionHandler(exceptionHandler);
    }
}
```

Don't forget to declare the custom application class in your manifest (with `android:name` on `application` element).
