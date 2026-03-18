OpenAutoDiag/
├── app/
│   ├── src/main/java/com/openautodiag/
│   │   ├── MainActivity.kt
│   │   ├── obd/
│   │   │   ├── ObdManager.kt
│   │   │   ├── BluetoothManager.kt
│   │   │   └── ProtocolDecoder.kt
│   │   ├── diag/
│   │   │   ├── CodeDatabase.kt
│   │   │   ├── LiveDataViewer.kt
│   │   │   └── DataLogger.kt
│   │   ├── ui/
│   │   │   ├── DashboardFragment.kt
│   │   │   ├── CodeReaderFragment.kt
│   │   │   └── LiveDataFragment.kt
│   │   └── utils/
│   │       ├── FileManager.kt
│   │       └── SafetyCheck.kt
│   └── res/
│       ├── layout/
│       └── values/
├── assets/
│   ├── dtc_codes.json
│   └── pid_list.json
└── build.gradle.kts

<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    
    <uses-permission android:name="android.permission.BLUETOOTH" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
    <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
    <uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
    <uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    
    <uses-feature android:name="android.hardware.bluetooth" android:required="true" />
    
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/Theme.OpenAutoDiag">
        
        <activity android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        
    </application>
</manifest>
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
}

android {
    compileSdk 34
    
    defaultConfig {
        applicationId "com.openautodiag"
        minSdk 21
        targetSdk 34
        versionCode 1
        versionName "1.0"
    }
    
    buildFeatures {
        viewBinding true
    }
}

dependencies {
    implementation 'androidx.core:core-ktx:1.12.0'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.11.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
    implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
    
    // Bluetooth
    implementation 'com.github.PhilJay:MPAndroidChart:v3.1.0'
    implementation 'com.github.anastr:speedviewlib:1.6.0'
    
    // JSON parsing
    implementation 'com.google.code.gson:gson:2.10.1'
}
