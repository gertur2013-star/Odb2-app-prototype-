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
package com.openautodiag.obd

class PidDecoder {
    fun decode(pid: String, response: String): Any {
        return when (pid) {
            ObdManager.CMD_ENGINE_RPM -> decodeRPM(response)
            ObdManager.CMD_VEHICLE_SPEED -> decodeSpeed(response)
            ObdManager.CMD_THROTTLE_POS -> decodeThrottle(response)
            ObdManager.CMD_ENGINE_LOAD -> decodeEngineLoad(response)
            ObdManager.CMD_COOLANT_TEMP -> decodeTemperature(response)
            else -> decodeGeneric(response)
        }
    }
    
    private fun decodeRPM(response: String): Int {
        val bytes = extractDataBytes(response)
        if (bytes.size >= 2) {
            return ((bytes[0] * 256 + bytes[1]) / 4).toInt()
        }
        return 0
    }
    
    private fun decodeSpeed(response: String): Int {
        val bytes = extractDataBytes(response)
        if (bytes.isNotEmpty()) {
            return bytes[0].toInt()
        }
        return 0
    }
    
    private fun decodeThrottle(response: String): Float {
        val bytes = extractDataBytes(response)
        if (bytes.isNotEmpty()) {
            return (bytes[0] * 100 / 255).toFloat()
        }
        return 0f
    }
    
    private fun decodeEngineLoad(response: String): Float {
        val bytes = extractDataBytes(response)
        if (bytes.isNotEmpty()) {
            return (bytes[0] * 100 / 255).toFloat()
        }
        return 0f
    }
    
    private fun decodeTemperature(response: String): Int {
        val bytes = extractDataBytes(response)
        if (bytes.isNotEmpty()) {
            return bytes[0].toInt() - 40
        }
        return 0
    }
    
    private fun extractDataBytes(response: String): ByteArray {
        // Remove "41 XX" header and any spaces
        val clean = response.replace("41 ", "").replace(" ", "")
        return if (clean.length % 2 == 0) {
            ByteArray(clean.length / 2) {
                clean.substring(it * 2, it * 2 + 2).toInt(16).toByte()
            }
        } else {
            byteArrayOf()
        }
    }
    
    private fun decodeGeneric(response: String): String {
        return response
    }
}
package com.openautodiag.obd

import android.content.Context
import com.google.gson.Gson
import com.google.gson.reflect.TypeToken
import org.json.JSONObject

class DtcDatabase(context: Context) {
    private val dtcMap = mutableMapOf<String, DTCInfo>()
    
    init {
        loadDatabase(context)
    }
    
    data class DTCInfo(
        val description: String,
        val severity: ObdManager.DTC.Severity,
        val system: ObdManager.DTC.SystemType,
        val causes: List<String> = emptyList(),
        val solutions: List<String> = emptyList()
    )
    
    private fun loadDatabase(context: Context) {
        try {
            val inputStream = context.assets.open("dtc_codes.json")
            val json = inputStream.bufferedReader().use { it.readText() }
            val type = object : TypeToken<Map<String, DTCInfo>>() {}.type
            val loaded = Gson().fromJson<Map<String, DTCInfo>>(json, type)
            dtcMap.putAll(loaded)
        } catch (e: Exception) {
            // Load default hardcoded codes
            loadDefaultCodes()
        }
    }
    
    private fun loadDefaultCodes() {
        // Common OBD2 codes
        val commonCodes = mapOf(
            "P0300" to DTCInfo(
                "Random/Multiple Cylinder Misfire Detected",
                ObdManager.DTC.Severity.ERROR,
                ObdManager.DTC.SystemType.ENGINE,
                listOf("Ignition system", "Fuel system", "Compression"),
                listOf("Check spark plugs", "Check fuel injectors", "Compression test")
            ),
            "P0420" to DTCInfo(
                "Catalyst System Efficiency Below Threshold",
                ObdManager.DTC.Severity.WARNING,
                ObdManager.DTC.SystemType.ENGINE,
                listOf("Catalytic converter", "O2 sensors", "Exhaust leak"),
                listOf("Check catalytic converter", "Test O2 sensors")
            ),
            "P0171" to DTCInfo(
                "System Too Lean (Bank 1)",
                ObdManager.DTC.Severity.WARNING,
                ObdManager.DTC.SystemType.ENGINE,
                listOf("Vacuum leak", "Fuel pressure", "MAF sensor"),
                listOf("Check for vacuum leaks", "Test fuel pressure", "Clean MAF sensor")
            )
        )
        dtcMap.putAll(commonCodes)
    }
    
    fun getDescription(code: String): String {
        return dtcMap[code]?.description ?: "Unknown DTC"
    }
    
    fun getSeverity(code: String): ObdManager.DTC.Severity {
        return dtcMap[code]?.severity ?: ObdManager.DTC.Severity.WARNING
    }
    
    fun getSystem(code: String): ObdManager.DTC.SystemType {
        return dtcMap[code]?.system ?: ObdManager.DTC.SystemType.ENGINE
    }
    
    fun getCauses(code: String): List<String> {
        return dtcMap[code]?.causes ?: emptyList()
    }
}
{
  "P0100": {
    "description": "Mass or Volume Air Flow Circuit Malfunction",
    "severity": "ERROR",
    "system": "ENGINE",
    "causes": ["MAF sensor failure", "Wiring issue", "ECU problem"],
    "solutions": ["Test MAF sensor", "Check wiring harness", "Scan ECU"]
  },
  "P0700": {
    "description": "Transmission Control System Malfunction",
    "severity": "ERROR",
    "system": "TRANSMISSION",
    "causes": ["TCM failure", "Transmission solenoid", "Wiring issue"],
    "solutions": ["Check TCM codes", "Test solenoids", "Inspect wiring"]
  },
  "U0100": {
    "description": "Lost Communication With ECM/PCM",
    "severity": "CRITICAL",
    "system": "CHASSIS",
    "causes": ["CAN bus failure", "ECU power loss", "Network issue"],
    "solutions": ["Check CAN bus", "Test ECU power", "Scan network"]
  }
}
{
  "supportedPids": [
    {"pid": "0100", "name": "Supported PIDs 01-20", "unit": "bits"},
    {"pid": "0101", "name": "Monitor Status", "unit": "bits"},
    {"pid": "010C", "name": "Engine RPM", "unit": "RPM"},
    {"pid": "010D", "name": "Vehicle Speed", "unit": "km/h"},
    {"pid": "0111", "name": "Throttle Position", "unit": "%"},
    {"pid": "0105", "name": "Coolant Temperature", "unit": "°C"},
    {"pid": "0104", "name": "Engine Load", "unit": "%"},
    {"pid": "010B", "name": "Intake Manifold Pressure", "unit": "kPa"},
    {"pid": "010F", "name": "Intake Air Temperature", "unit": "°C"},
    {"pid": "0110", "name": "MAF Air Flow Rate", "unit": "g/s"}
  ]
}
package com.openautodiag

import android.Manifest
import android.bluetooth.BluetoothAdapter
import android.bluetooth.BluetoothDevice
import android.content.pm.PackageManager
import android.os.Build
import android.os.Bundle
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import androidx.fragment.app.Fragment
import com.openautodiag.databinding.ActivityMainBinding
import com.openautodiag.obd.ObdManager

class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding
    private lateinit var obdManager: ObdManager
    
    companion object {
        private const val REQUEST_BLUETOOTH_PERMISSIONS = 100
        private val REQUIRED_PERMISSIONS = arrayOf(
            Manifest.permission.BLUETOOTH_SCAN,
            Manifest.permission.BLUETOOTH_CONNECT,
            Manifest.permission.ACCESS_FINE_LOCATION
        )
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
        
        obdManager = ObdManager(this)
        
        checkPermissions()
        setupNavigation()
        setupObservers()
    }
    
    private fun checkPermissions() {
        val missingPermissions = REQUIRED_PERMISSIONS.filter {
            ContextCompat.checkSelfPermission(this, it) != PackageManager.PERMISSION_GRANTED
        }
        
        if (missingPermissions.isNotEmpty()) {
            ActivityCompat.requestPermissions(
                this,
                missingPermissions.toTypedArray(),
                REQUEST_BLUETOOTH_PERMISSIONS
            )
        }
    }
    
    private fun setupNavigation() {
        binding.bottomNavigation.setOnItemSelectedListener { item ->
            when (item.itemId) {
                R.id.nav_dashboard -> {
                    replaceFragment(DashboardFragment.newInstance(obdManager))
                    true
                }
                R.id.nav_codes -> {
                    replaceFragment(CodeReaderFragment.newInstance(obdManager))
                    true
                }
                R.id.nav_live -> {
                    replaceFragment(LiveDataFragment.newInstance(obdManager))
                    true
                }
                else -> false
            }
        }
        
        replaceFragment(DashboardFragment.newInstance(obdManager))
    }
    
    private fun replaceFragment(fragment: Fragment) {
        supportFragmentManager.beginTransaction()
            .replace(R.id.fragment_container, fragment)
            .commit()
    }
    
    private fun setupObservers() {
        obdManager.connectionState.observe(this) { state ->
            when (state) {
                ObdManager.ConnectionState.CONNECTED -> {
                    Toast.makeText(this, "Connected to OBD2", Toast.LENGTH_SHORT).show()
                }
                ObdManager.ConnectionState.ERROR -> {
                    Toast.makeText(this, "Connection error", Toast.LENGTH_SHORT).show()
                }
                else -> {}
            }
        }
    }
    
    override fun onDestroy() {
        super.onDestroy()
        obdManager.disconnect()
    }
}
package com.openautodiag.ui

import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.fragment.app.Fragment
import androidx.lifecycle.lifecycleScope
import com.openautodiag.databinding.FragmentDashboardBinding
import com.openautodiag.obd.ObdManager
import kotlinx.coroutines.launch

class DashboardFragment : Fragment() {
    private var _binding: FragmentDashboardBinding? = null
    private val binding get() = _binding!!
    
    private lateinit var obdManager: ObdManager
    
    companion object {
        fun newInstance(manager: ObdManager): DashboardFragment {
            return DashboardFragment().apply {
                obdManager = manager
            }
        }
    }
    
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentDashboardBinding.inflate(inflater, container, false)
        return binding.root
    }
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        setupUI()
        observeLiveData()
    }
    
    private fun setupUI() {
        binding.btnScanDevices.setOnClickListener {
            // Open device scan dialog
        }
        
        binding.btnReadCodes.setOnClickListener {
            lifecycleScope.launch {
                obdManager.readDTCs()
            }
        }
        
        binding.btnClearCodes.setOnClickListener {
            obdManager.clearDTCs()
        }
    }
    
    private fun observeLiveData() {
        obdManager.liveData.observe(viewLifecycleOwner) { data ->
            data.forEach { (key, value) ->
                when (key) {
                    "Engine RPM" -> binding.tvRpm.text = "$value RPM"
                    "Vehicle Speed" -> binding.tvSpeed.text = "$value km/h"
                    "Throttle Position" -> binding.tvThrottle.text = "$value%"
                    "Engine Load" -> binding.tvLoad.text = "$value%"
                    "Coolant Temp" -> binding.tvTemp.text = "${value}°C"
                }
            }
        }
        
        obdManager.dtcCodes.observe(viewLifecycleOwner) { codes ->
            binding.tvCodeCount.text = "Codes: ${codes.size}"
            if (codes.isNotEmpty()) {
                binding.tvLastCode.text = codes.first().code
            }
        }
    }
    
    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
package com.openautodiag.advanced

// WARNING: Actual programming requires licensed access and safety measures
// This is only a read-only example

class EcuInfoReader {
    fun readEcuInfo(protocol: String): Map<String, String> {
        return mapOf(
            "ECU Serial" to readSerialNumber(protocol),
            "Software Version" to readSoftwareVersion(protocol),
            "Calibration ID" to readCalibrationId(protocol)
        )
    }
    
    private fun readSerialNumber(protocol: String): String {
        // Implementation depends on specific manufacturer protocol
        return when (protocol) {
            "KWP2000" -> readKwpSerial()
            "CAN" -> readCanSerial()
            else -> "Not Supported"
        }
    }
    
    private fun readSoftwareVersion(protocol: String): String {
        // Placeholder - real implementation requires protocol-specific commands
        return "V1.0.0"
    }
    
    private fun readCalibrationId(protocol: String): String {
        // Placeholder - real implementation requires protocol-specific commands
        return "CAL123456"
    }
}
