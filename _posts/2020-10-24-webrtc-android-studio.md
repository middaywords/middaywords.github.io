---
title: Setup WebRTC android app in Android Studio
date: 2020-10-24
permalink: /posts/2020/10/android-webrtc/
tags:
  - webrtc
  - android
typora-root-url: ../../middaywords.github.io
---

repo: https://github.com/Forsworns/webrtc_android



### Setup guide

1. import the android project in `src/examples/androidapp/` to Android Studio.

2. Copy the `src/examples/androidapp/third_part/autobanh/lib/autobanh.jar` to `libs/` in Android project.

3. Add build.gradle at application level (This is example for reference)

   ```sh
   plugins {
       id 'com.android.application'
   }
   
   android {
       compileSdkVersion 30
       buildToolsVersion "30.0.0"
   
       defaultConfig {
           applicationId "com.example.webrtcapp"
           minSdkVersion 21
           targetSdkVersion 30
           versionCode 1
           versionName "1.0"
   
           testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
       }
   
       buildTypes {
           release {
               minifyEnabled false
               proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
           }
       }
       compileOptions {
           sourceCompatibility JavaVersion.VERSION_1_8
           targetCompatibility JavaVersion.VERSION_1_8
       }
   }
   
   dependencies {
       implementation fileTree(dir: 'libs', include: ['*.jar'])
       implementation 'androidx.appcompat:appcompat:1.0.0'
       implementation 'com.google.code.findbugs:jsr305:3.0.2'
       implementation 'org.webrtc:google-webrtc:1.0.+'
       implementation 'androidx.annotation:annotation:1.0.0'
       testImplementation 'junit:junit:4.12'
       androidTestImplementation 'androidx.test.ext:junit:1.1.1'
       androidTestImplementation 'androidx.test.espresso:espresso-core:3.1.0'
   }
   ```

   To support development in Android studio, the easiest way to use webrtc library in android is using the [official prebuilt libraries](https://bintray.com/google/webrtc/google-webrtc) available at JCenter. 

   ```
   implementation 'org.webrtc:google-webrtc:1.0.+'
   ```

4. Then you can start developing the AppRTC in Android Studio.



---

#### If you want to use your own webrtc library

1. Firstly, you can follow the [official guide](https://webrtc.googlesource.com/src/+/refs/heads/master/docs/native-code/android/index.md) to compile the source code, you can directly get the AppRTC app.

2. Or you can only compile the library and support debugging the app in Android studio.

   1. open Application Level `build.gradle` file and add `*.aar` file such as:

      ```
      		dependencies {
              compile(name:'libwebrtc', ext:'aar')
          }
      ```

3. To compile the AppRTC example android app in webrtc, we can follow the [official guide](https://webrtc.googlesource.com/src/+/refs/heads/master/docs/native-code/android/index.md).



#### More about AppRTC

1. Defaultly, apprtc android client will use `https://appr.tc` as signaling server and ICE server. however it is quite unstable. You can change the default signaling sever and ICE server in app settings.

2. You can also enable peer connection by setting the `room name` in apprtc.

   1. For AppRTC server in p2p connection, if the `room name` is local IP address, the app will server as a server and listen on 8888 port. 
   2. For AppRTC client in p2p connection, you should set the `room name` as `{server-ip}:{server-port}` , then it will connects to the server and starts the telephony.
   3. For other `room name`,  you should rely on other rely on ICE server.

3. In implementation, the p2p telephony(does not rely on ICE sever) is implemented in this way: only provide sdp information in signaling message.

   ```java
   				SignalingParameters parameters = new SignalingParameters(
               // Ice servers are not needed for direct connections.
               new ArrayList<>(),
               false, // isServer
               null, // clientId
               null, // wssUrl
               null, // wssPostUrl
               sdp, // offerSdp
               null // iceCandidates
           );
   ```

4. You can also start the app in adb

   You can also run application from a command line to connect to the first room in a list:

   `adb shell am start -n org.appspot.apprtc/.ConnectActivity -a android.intent.action.VIEW`

   This should result in the app launching on Android and connecting to the 3-dot-apprtc page displayed in the desktop browser.

   To run loopback test execute following command:

   `adb shell am start -n org.appspot.apprtc/.ConnectActivity -a android.intent.action.VIEW --ez "org.appspot.apprtc.LOOPBACK" true`

   

   