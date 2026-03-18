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
package com.openautodiag.obd

import android.bluetooth.*
import android.content.Context
import android.util.Log
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import java.io.*
import java.util.*

class ObdManager(context: Context) {
    companion object {
        const val ELM_DEVICE_NAME = "OBDII"
        const val UUID_SPP = "00001101-0000-1000-8000-00805F9B34FB"
        
        // OBD Commands
        const val CMD_ENGINE_RPM = "010C"
        const val CMD_VEHICLE_SPEED = "010D"
        const val CMD_THROTTLE_POS = "0111"
        const val CMD_ENGINE_LOAD = "0104"
        const val CMD_COOLANT_TEMP = "0105"
        const val CMD_READ_DTC = "03"
        const val CMD_CLEAR_DTC = "04"
        const val CMD_SUPPORTED_PIDS = "0100"
    }
    
    private val _connectionState = MutableStateFlow(ConnectionState.DISCONNECTED)
    val connectionState: StateFlow<ConnectionState> = _connectionState
    
    private val _liveData = MutableStateFlow<Map<String, Any>>(emptyMap())
    val liveData: StateFlow<Map<String, Any>> = _liveData
    
    private val _dtcCodes = MutableStateFlow<List<DTC>>(emptyList())
    val dtcCodes: StateFlow<List<DTC>> = _dtcCodes
    
    private var bluetoothSocket: BluetoothSocket? = null
    private var inputStream: InputStream? = null
    private var outputStream: OutputStream? = null
    
    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    private val commandChannel = Channel<String>(Channel.UNLIMITED)
    
    private val pidDecoder = PidDecoder()
    private val dtcDatabase = DtcDatabase(context)
    
    enum class ConnectionState {
        DISCONNECTED, SCANNING, CONNECTING, CONNECTED, ERROR
    }
    
    data class DTC(
        val code: String,
        val description: String,
        val severity: Severity,
        val system: SystemType
    ) {
        enum class Severity { INFO, WARNING, ERROR, CRITICAL }
        enum class SystemType { ENGINE, TRANSMISSION, ABS, AIRBAG, BODY, CHASSIS }
    }
    
    fun connectToDevice(device: BluetoothDevice) {
        scope.launch {
            _connectionState.value = ConnectionState.CONNECTING
            try {
                bluetoothSocket = device.createRfcommSocketToServiceRecord(
                    UUID.fromString(UUID_SPP)
                )
                bluetoothSocket?.connect()
                
                inputStream = bluetoothSocket?.inputStream
                outputStream = bluetoothSocket?.outputStream
                
                // Initialize ELM327
                sendCommand("ATZ") // Reset
                delay(1000)
                sendCommand("ATE0") // Echo off
                sendCommand("ATL0") // Linefeeds off
                sendCommand("ATS0") // Spaces off
                sendCommand("ATSP0") // Auto protocol
                
                startLiveDataStream()
                _connectionState.value = ConnectionState.CONNECTED
                
            } catch (e: Exception) {
                _connectionState.value = ConnectionState.ERROR
                disconnect()
            }
        }
    }
    
    fun readDTCs() {
        scope.launch {
            try {
                val response = sendCommand(CMD_READ_DTC)
                val codes = parseDTCResponse(response)
                val dtcList = codes.map { code ->
                    DTC(
                        code = code,
                        description = dtcDatabase.getDescription(code),
                        severity = dtcDatabase.getSeverity(code),
                        system = dtcDatabase.getSystem(code)
                    )
                }
                _dtcCodes.value = dtcList
            } catch (e: Exception) {
                Log.e("OBD", "Failed to read DTCs", e)
            }
        }
    }
    
    fun clearDTCs() {
        scope.launch {
            sendCommand(CMD_CLEAR_DTC)
            _dtcCodes.value = emptyList()
        }
    }
    
    private fun startLiveDataStream() {
        scope.launch {
            val pidList = listOf(
                CMD_ENGINE_RPM, CMD_VEHICLE_SPEED, CMD_THROTTLE_POS,
                CMD_ENGINE_LOAD, CMD_COOLANT_TEMP
            )
            
            while (_connectionState.value == ConnectionState.CONNECTED) {
                val data = mutableMapOf<String, Any>()
                
                pidList.forEach { pid ->
                    try {
                        val response = sendCommand(pid)
                        val value = pidDecoder.decode(pid, response)
                        data[getPidName(pid)] = value
                    } catch (e: Exception) {
                        // Skip failed PIDs
                    }
                }
                
                _liveData.value = data
                delay(500) // Update every 500ms
            }
        }
    }
    
    private suspend fun sendCommand(command: String): String {
        return withContext(Dispatchers.IO) {
            outputStream?.write("$command\r".toByteArray())
            outputStream?.flush()
            
            val buffer = ByteArray(1024)
            val bytes = inputStream?.read(buffer) ?: 0
            String(buffer, 0, bytes).trim()
        }
    }
    
    private fun parseDTCResponse(response: String): List<String> {
        // Example response: "43 01 33 00 00 00"
        val codes = mutableListOf<String>()
        val hexPairs = response.split(" ").filter { it.isNotEmpty() }
        
        if (hexPairs.size > 2) {
            for (i in 2 until hexPairs.size step 2) {
                if (i + 1 < hexPairs.size) {
                    val firstByte = hexPairs[i].toInt(16)
                    val secondByte = hexPairs[i + 1].toInt(16)
                    
                    val dtc = calculateDTC(firstByte, secondByte)
                    if (dtc.isNotBlank()) {
                        codes.add(dtc)
                    }
                }
            }
        }
        
        return codes
    }
    
    private fun calculateDTC(firstByte: Int, secondByte: Int): String {
        val type = (firstByte shr 6) and 0x03
        val system = when (type) {
            0 -> "P" // Powertrain
            1 -> "C" // Chassis
            2 -> "B" // Body
            3 -> "U" // Network
            else -> "P"
        }
        
        val code1 = ((firstByte shr 4) and 0x03).toString(16).uppercase()
        val code2 = (firstByte and 0x0F).toString(16).uppercase()
        val code3 = ((secondByte shr 4) and 0x0F).toString(16).uppercase()
        val code4 = (secondByte and 0x0F).toString(16).uppercase()
        
        return "$system$code1$code2$code3$code4"
    }
    
    private fun getPidName(pid: String): String {
        return when (pid) {
            CMD_ENGINE_RPM -> "Engine RPM"
            CMD_VEHICLE_SPEED -> "Vehicle Speed"
            CMD_THROTTLE_POS -> "Throttle Position"
            CMD_ENGINE_LOAD -> "Engine Load"
            CMD_COOLANT_TEMP -> "Coolant Temp"
            else -> pid
        }
    }
    
    fun disconnect() {
        scope.launch {
            try {
                inputStream?.close()
                outputStream?.close()
                bluetoothSocket?.close()
            } catch (e: Exception) {
                Log.e("OBD", "Error disconnecting", e)
            } finally {
                _connectionState.value = ConnectionState.DISCONNECTED
                _liveData.value = emptyMap()
            }
        }
    }
}
