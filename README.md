# M.E.P.P. Android SDK - Setup Guide

This guide will take you through the process of adding the SDK to your Android application. 


### Features

 * (Automatic) Scanning for beacons and receiving corresponding content from the backend CMS
 * Fetching content objects from backend via ID


## Setup

### Requirements

It assumes that you have an Android project using Android Studio. The SDK builds from Android 4.3 (18) onwards. The SDK will be provided as an aar package.


### Step 1: Adding the SDK to your project

In Android Studio (under edit) File -> New Module -> Import .JAR/.AAR, import the .AAR.

In your project build.gradle (not the top level one, the one under 'app' module) add the following (in the dependencies section):

```xml
dependencies {
    compile project(':mepplibrary-release_1.3.0')
    compile 'com.google.android.gms:play-services-analytics:10.0.1'
    compile 'com.google.android.gms:play-services-awareness:10.0.1'
    compile 'com.squareup.okhttp3:okhttp:3.2.0'
    compile 'com.squareup:otto:1.3.8'
    compile 'org.altbeacon:android-beacon-library:2.8.1'
    compile 'com.google.code.gson:gson:2.8.0'
    }
```

Synchronize and check if settings.gradle now contains the include.

#### Alternative route

Put the library on your local maven directory and use as a dependency.


### Step 2: Permissions

Interacting with beacons requires following permissions for different aspects of the library:
```xml
<uses-permission android:name="android.permission.BLUETOOTH"/>
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/>
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

With the release of Android Marshmallow, the permission system has changed significantly. Starting a Bluetooth Low Energy scan requires permission from Location group.

As a result of that one of the following permissions is required:

```xml
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
```


### Step 3: Adding Meta Data

Your AndroidManifest.xml needs some additional meta information for configuration purposes (in the application section of the manifest). Please replace the placeholders with your App Token, server endpoint and Google Analytics UA ID.

```xml
<meta-data
    android:name="mepp.apptoken"
    android:value="$your_app_token_here$" />
<meta-data
    android:name="mepp.cms.endpoint"
    android:value="$your_server_endpoint_here$" />
<meta-data
    android:name="mepp.analytics.id"
    android:value="$your_google_analytics_ua_here$" />
```

Optional you can set a beacon scan interval in seconds as meta data. If you don't set an interval in the manifest, the default scan interval of 120 seconds will be used.

```xml
<meta-data
    android:name="mepp.beacon.scan.interval"
    android:value="60" />
```

### Step 4: Registering Service

Enable the SDK Scanning service in the application section of the manifest.

```xml
<service
  android:name="com.mopius.mepplibrary.controller.MeppService"
  android:exported="false" />
```


### Step 5: Adding Broadcast Receiver

To receive log events and (beacon or geofence) content from the SDK, it's necessary to add a Broadcast receiver to the application section in your app's manifest. The action filter for log output is `your.package.name` + `.log`, for example `com.mopius.mepp.demo.log`. For receiving content it's `your.package.name` + `.content`, for example `com.mopius.mepp.demo.content`

```xml
<receiver
  android:name=".BroadcastReceiver">
  <intent-filter>
    <action android:name="your.package.name.log" />
    <action android:name="your.package.name.content" />
  </intent-filter>
</receiver>
```


### Step 6: Handle device reboots

The SDK takes care of device reboots. If you have started scanning for beacons or geofences before the user is rebooting his/her device, the SDK will automatically restart scanning after the reboot is finished.  

## Usage


### Starting the SDK

To start the SDK you simply have to (e.g. in your MainActivity) instantiate a `MeppManager`. The constructor of the `MeppManager` takes two parameters. The first one is the context and the second one is a boolean with which you can state, if the `MeppManager` should start scanning for beacons right away. Please mind the permission requesting on Android Marshmallow and onwards.

To stop the scanning for beacons, just call `stopScanning()` on the `MeppManager` instance. Later on you can restart the scan procedure with `startScanning()`.

When the App gets closed, please don't forget to call the MeppManager's `shutDownManager(boolean)` method. With the boolean parameter you can select, if the SDK should continue scanning for beacons or not.


### Receiving Broadcasts (Logs & Content)

Like in Step 5 of the guide stated, the SDK delivers beacon or geofence content via a broadcast. The following code snipped shows, how to extract a `MeppContent` object out of a broadcast within the `onReceive` Method. The intent bundle holds the SDK's log information as a String object, you can get the name of it with `Broadcasts.getLogFilter(context)` which is `your.package.name.log`. The content object is a serializable extra which can be deserialized by the name `your.package.name.content` or simply call `Broadcasts.getContentFilter(context)`.

```java
@Override
public void onReceive(Context context, Intent intent) {
	if (intent.getAction().equals(Broadcasts.getLogFilter(context))) {
		Bundle bundle = intent.getExtras();
		String str = (String) bundle.get(MeppConstants.MEPP_INTENT_EXTRA_LOG);

        if(str != null)
			Log.d(TAG, "Received log: " + str);

	} else if (intent.getAction().equals(Broadcasts.getContentFilter(context))) {
        MeppContent contentModel = (MeppContent) intent.getSerializableExtra(MeppConstants.MEPP_INTENT_EXTRA_CONTENT);
        if (contentModel != null) {
        	Log.d(TAG, "Received content: " + contentModel.getId());
        	// go on...
        }
    }
}
```

The `MeppContent` Object holds all the information provided by the CMS and is filled depending on the individual use case scenario.

#### Alternative

You can also receive the information via local broadcasts. To do so, create a `LocalBroadcastManager` like this:

```java
LocalBroadcastManager.getInstance(this).registerReceiver(mContentReceiver,
                        new IntentFilter(Broadcasts.getContentFilter(context));
```
```java
BroadcastReceiver mContentReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            // Get extra data included in the Intent
            MeppContent contentModel = (MeppContent) intent.getSerializableExtra(MeppConstants.MEPP_INTENT_EXTRA_CONTENT);

            if (contentModel != null) {
				// go on...
            }
        }
    };
```


### Fetching specific content

The SDK supports requesting a specific content entry by it's ID. To do so, get an instance of the `MeppManager` and call the `fetchContentByID([ContentListener], [ID])` method. This method takes two arguments. The first one is a listener which gets called when the webrequest finished, and the second one is the ID of the desired content:

```java
mMeppManager.fetchContentByID(this, "313");
```
The callback method of the `ContentListener` looks like this:

```java
IContentListener() {
    @Override
    public void onContentReceived(MeppContent _content, String _errorMessage) {
        // do something
    }
}
```

The callback method of the `ContentListener` gets called after every request, no mather if it was successful or not. On a successful request the `MeppContent` object is prefilled, not null and the `ErrorMessage` String is empty. If something went wrong, the `MeppContent` object is null and an error message was generated.

### Handle NFC content

To handle NFC content the NFC permission is required:

```xml
<uses-permission android:name="android.permission.NFC" />
```
To handle incoming NFC intents you need to define an intent filter in your manifest file:
```xml
<intent-filter>
    <action android:name="android.nfc.action.NDEF_DISCOVERED" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:scheme="https" android:host="mepp.at" android:path="/nfc"/>
    <data android:scheme="http" android:host="mepp.at" android:path="/nfc"/>
</intent-filter>
```

The SDK supports handling NFC content. To do so, get an instance of the `MeppManager` and call the `handleNfcIntent([Intent], [HardwareListener])` method in `onCreate()` and `onNewIntent()` of the activity where you want to handle the NFC content. This method takes two arguments. The first one is an intent, and the second one is a listener which gets called when the webrequest finished:

```java
mMeppManager.handleNfcIntent(getIntent(), this);
```
The callback method of the `HardwareListener` looks like this:

```java

IWSRGetHardwareListener() {
    @Override
    public void onHardwareReceived(HardwareWSRResponse _response, String _errorMessage) {
        // do something
    }
}
```

The callback method of the `HardwareListener` gets called after every request, no mather if it was successful or not. On a successful request the `HardwareWSRResponse` object is prefilled, not null and the `ErrorMessage` String is empty. If something went wrong, the `HardwareWSRResponse` object is null and an error message was generated.

To handle NFC content in foreground you have to get an instance of the `MeppManager` and call the `setupForegroundDispatch([Context])` in  `onResume()` and `stopForegroundDispatch([Context])` in `onPause()` of the activity where you want to handle NFC content in foreground:

```java
mMeppManager.setupForegroundDispatch(this);
```

```java
mMeppManager.stopForegroundDispatch(this);
```

### Handle QR content

To handle incoming QR intents you need to define an intent filter in your manifest file:
```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <category android:name="android.intent.category.BROWSABLE"/>
    <data android:scheme="https" android:host="mepp.at" android:path="/qr"/>
    <data android:scheme="http" android:host="mepp.at" android:path="/qr"/>
</intent-filter>
```
The SDK supports requesting content by a specific QR-Code URI. To do so, get an instance of the `MeppManager` and call the `fetchContentByHardwareQr([HardwareListener], [URI])` method. This method takes two arguments. The first one is a listener which gets called when the webrequest finished, and the second one is the URI saved on the QR-Code:

```java
mMeppManager.fetchContentByHardwareQr(this, "https://mepp.at/qr?id=1234asdf");
```

The callback method of the `HardwareListener` looks like this:

```java

IWSRGetHardwareListener() {
    @Override
    public void onHardwareReceived(HardwareWSRResponse _response, String _errorMessage) {
        // do something
    }
}
```

The callback method of the `HardwareListener` gets called after every request, no mather if it was successful or not. On a successful request the `HardwareWSRResponse` object is prefilled, not null and the `ErrorMessage` String is empty. If something went wrong, the `HardwareWSRResponse` object is null and an error message was generated.

### Handle Geofence content

To use geofences in your application add the following (in the dependencies section)
in your project build.gradle (not the top level one, the one under 'app' module):

```xml
dependencies {
         compile 'com.squareup:otto:1.3.8'
         compile 'com.google.android.gms:play-services-awareness:9.8.0'
    }
```

To handle geofences, enable the transition service in the application section of the manifest.

```xml
<service
 android:name="com.mopius.mepplibrary.controller.MeppGeofenceTransitionsIntentService"
 android:exported="false" />
```

To start scanning for geofences, just call `startGeofencing()` on the `MeppManager` instance. Later on you can stop the scan procedure with `stopGeofencing()`. Or you can just use the `startGeofencing` parameter of the `MeppManager` constructor to start scanning for geofences.

Like in Step 5 of the guide stated, the SDK delivers a geofence's content via a broadcast.
You gonna receive a content broadcast on a geofence transition if you have set enter or exit content for the geofence.
See "Receiving Broadcasts (Logs & Content)" for more information about how to receive SDK broadcasts.

## Data structure


### MeppContent


 * **int id;**

	This is the internal content ID. Use this to fetch content object again later on.

 * **boolean active;**

	This boolean states if the content is currently acitve or not

 * **String contentType;**

	This defines the type of the content (a.k.a use case)

 * **String displayType;**

	This defines if the content gets notified on entry or on exit of a beacon region (handled by SDK)

 * **int delayTime;**

	This is the delay time in seconds after which a user-visible notification should be triggered

 * **int coolDownTime;**

	This is the cooldown time in seconds after which the SDK notifies for a content again (handled by SDK)

 * **Date activeDateStart;**

	This indicates the date from which a content is active

 * **Date activeDateStop;**

	This indicates the date until a content is active

 * **String activeTimeStart;**

	This delivers the time of the day from which a content is active

 * **String activeTimeStop;**

	This delivers the time of the day until a content is active

 * **MetaInfo metaInfo;**

	This object holds the image uri and the link uri as strings

 * **TextRecord textRecord;**

	This object holds a name and a text (for notification)

 * **Map<String, String> extraInfo;**

	This object holds the extra infos as a map (e.g. the legal text)

 * **SeenWithBeacon beaconSeenWith;**

	This object holds the information of the beacon the content was fetched with
    

### MetaInfo

 * **String imageUri;**

	This is the image uri

 * **String linkUri;**

	This is the link uri


### TextRecord
 * **String name;**

	This is the notification title

 * **String text;**

	This is the notification text


### SeenWithBeacon
 * **String uuid;**

	This is the beacons UUID
    
 * **int major;**

	This is the beacons major ID
    
 * **int minor;**

	This is the beacons minor ID
    
### Changelog

v1.3.0
- Set background scan interval in manifest
- Restart scanning after device reboot 
- Removed Kontakt.io SDK

v1.2.0
- Geofence and QR support
- Clear SDK cache method added (`mMeppManager.clearSDKCaches()`)
- Kontakt.io SDK update (update your build.gradle (in the dependencies section) `compile 'com.kontaktio:sdk:3.2.1'`)

v1.1.0
- NFC support
- Action filters for Broadcast Receiver changed to `your.package.name.log` and `your.package.name.content` (see step 5)
- Methods to get the correct action filters added (see "Receiving Broadcasts (Logs & Content)")
- If you use an application object you no longer need to extend MEPPApplication

v1.0.0
- Initial release

<br><br>
**Thanks for using M.E.P.P.**<br>
If you have any troubles integrating or running the SDK, please feel free to contact us!<br>

© [Mopius Mobile GmbH](https://www.mopius.com) 2017 - Created on 2017-03-07
