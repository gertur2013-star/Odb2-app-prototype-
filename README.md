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
