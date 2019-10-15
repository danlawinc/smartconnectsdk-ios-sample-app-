# smartconnectsdk-iOS-sample-app
This sample app acts as a template to quickstart your iOS app development. 

To build the project, just **copy the SDK (.framework file) to Frameworks folder** of your project and **replace apiKey with your key** that was issued to you by Danlaw in ```LaunchScreenVC.swift``` file.

# Table of Contents  

1. [Features](#features)
2. [Requirements](#requirements)
3. [Installation](#installation)
4. [Authentication](#authentication)
5. [Connecting to Datalogger](#connecting-to-datalogger)
6. [Auto-Connect](#auto-connect)
7. [Basic PIDs](#basic-pids)
8. [Advanced PIDs](#advanced-pids)
9. [Data PIDs](#data-pids)
10. [UDP Events](#udp-events)
11. [FAQ](#faq)
12. [Credits](#credits)



# Features
- [x] Scan Devices
- [x] Enable Bluetooth
- [x] Connect/Disconnect to Devices
- [x] Set Favorite/Forget Devices
- [x] Autoconnect to Favorite Device
- [x] Firmware Update
- [x] Read basic and data pids 
- [x] Register/unregister data pids for continuous updates
- [x] Register/unregister for event updates

# Requirements
- iOS 10.0+
- Xcode 11.0+
- Swift 5.0+


# Installation
- Create Framework directory inside your Project's root folder. Copy-paste iOS SDK provided by Danlaw. 
- Go to Project's Target settings and import framework from the directory.
- Replace ApiKey issued by Danlaw in 'LaunchScreenVC'

```import SmartConnectSDK``` in swift file to access framework.

# Component Library
- DLAuthInterface: This class AuthInterface provides the entry point for the SDK. The hosting application should call the interface method validateToken to get the SDK authenticated. If the SDK is not authenticated, then none of the services that are offered by the SDK will be available.
- DLGatewayInterface: This class handles communication between the app and the SDK. It provides the outward facing methods for interacting with the Danlaw iOS SDK.
- DLGatewayDelegate: This `protocol DLGatewayDelegate` defines the callbacks required for the delegate on the `DLGatewayInterface`.
- DLBleapInterface: This class handles communication between the app and SDK for UDP events. This interface has to be implemented only if mobile app uses Bleap Datalogger
- DLBleapUDPDataDelegate: This `protocol DLBleapUDPDataDelegate` defines the callbacks required for the delegate on the `DLBleapInterface`.

# Authentication
After installing the SDK, **you MUST authenticate it before you can use all the interfaces**. 
To authenticate the SDK, use the following method:

```
/**
 - parameter apiKey: ApiKey issued by Danlaw Inc.
 - parameter issuedTo: App's main bundle
 - parameter delegate: Instance of implementing class of DLAuthDelegate
*/
// Request to SDK
DLAuthInterface.sharedInstance.validateToken(apiKey: self.apiKey, issuedTo: Bundle.main, delegate: self)
```

SDK uses `DLAuthDelegate` method in response of `validateToken:`

```
// Response from SDK
/**
    - parameter authenticationResult: 200 in case of Success
    - parameter message: Message from SDK success/error
*/
func onAuthenticationResult(authenticationResult: Int, message: String)
``` 

# Connecting to Datalogger
1. Get an instance:

```
/**
 - parameter delegate: Instance of implementing class of DLGatewayDelegate
*/
do {
    var gateway:DLGatewayInterface
    try gateway = DLGatewayInterface.getInstance()
    gateway.setDelegate(delegate: self)
}catch DLException.SdkNotAuthenticatedException(let error) {
    print(error)
}
``` 

2. Scan for devices, scan results are received in delegate method of DLGatewayDelegate `onOBDDeviceFound`:

```gateway.startScan(start: true)```

3. Connect to device using the `deviceName` returned in `func onOBDDeviceFound(deviceName: String, identifier: String)`:

```gateway.connect(name: deviceName)```

# Auto-Connect

SmartConnect SDK allows mobile app to connect to Datalogger automatically. To enable this feature, device must be set as favorite. SDK uses iBeacon Services and Location services to search for favorite datalogger. 

1. Set a device as favorite:

```gateway.setFavoriteDevice(name: deviceName, identifier: identifier)```

2. To remove favorite device, use following method:

```gateway.forgetDevice()```

Auto-connect requires Bluetooth access and Location service enabled to "Always".<br /> Auto connect works in all states of mobile app(Active, InActive, Background, Suspended, Non-Running)<br />
Add ```gateway.enableiBeaconServices(isBeaconMonitoring: true)``` and ```gateway.startBackgroundScan(start: true)``` in ```func applicationDidEnterBackground(_ application: UIApplication)``` method to enable background wakeups<br />
Add required Privacy permission property keys in app's info.plist(Refer Page.13 of Danlaw SmartConnect Installation guide)


# Basic Channel

Datalogger uses Basic channel to send data pids and custom data pids. Datalogger sends PID data once for each request made by Datalogger. Baic channel can be used to request data that does not require frequent update<br />

- Data that can be requested using Basic channel:<br />
 1. Standard Pids (id: 0-255) (Refer Page.55-56 of Danlaw SmartConnect Installation guide)<br />
 2. Danlaw's Custom PIDs (id: 256 and over) (Refer Page.57 of Danlaw SmartConnect Installation guide)<br />

Request custom PID(VIN Number) using Basic Channel method:

```let isPidAvailable = gateway.readBasicPidData(pid: DLCommandPId.basic.VIN)```

SDK uses following method to respond with received custom PID data:

```
func onBasicDataReceived(responseCode: Int, pid: Int, object: DLBasicPIDObject?){
    if responseCode != 0 {
       print("Response failed for pid:", " \(pid)")
    } else {
      switch pid {
        case DLCommandPId.basic.VIN: //Vin number
            guard let vin = object as? DLVin else { return }
                if let vinNo = vin.value {
                    print("\nVIN: \(vinNo)")
                 }
         default:
             print("Basic data")
        }
     }
}
```

Request Data PID(MAF Rate) using Basic Channel method:

```let isPidAvailable = gateway.readBasicPidData(pid: DLCommandPId.basic.MAFRate)```

SDK uses following method to respond with received Data Pid's data:

```
func onDataPidDataReceived(responseCode: Int, DPid: Int, hashmap: [Int : DLBasicPIDObject]) {
        if responseCode != 0 {
            print("Received Failing Response Code")
            return }
        for (pid, object) in hashmap {
            switch pid {
            case DLCommandPId.basic.MAFRate:
               guard let maf = object as? DLMAFRate else {return}
                if let mafValue = maf.value {
                    print("\nMAFRate: \(mafValue)")}
            default:
                print("Data Pid received")
            }
        }
    }
```

# Advanced Channel

Datalogger uses Advanced channel to send Event pids and Data pids. Mobile app has to register pids once and device will keep sending events for registered pids in real time.<br /> 

- Data that can be requested using Basic channel:<br />
 1. Standard Pids (id: 0-255) (Refer Page.55-56 of Danlaw SmartConnect Installation guide)<br />
 2. Event PIDs (id: 256 and over) (Refer Page.57 of Danlaw SmartConnect Installation guide)<br />
 Note: Event PIDs have to be preconfigured in datalogger to receive real time events.

<br />Request Event Pids using Advanced channel.

```
let isEPidsRegistered = gateway.registerEventPid(pids: [DLEventID.hardBraking, DLEventID.hardAcceleration, DLEventID.idling, DLEventID.tripStart, DLEventID.tripEnd])
```

SDK uses following method to respond with received Event data:

```
/**
 - parameter responseCode: 0 if success
 - parameter EPid: Pid Id
 - parameter object: DLDataObject (Super class of all event Object. Refer Page.57 of Installation guide to see Object class for each event)
*/
func onEventPidDataReceived(responseCode: Int, EPid: Int, object: DLDataObject?) {
    if responseCode != 0 {
        print("Received Failing Response Code")
        return
     }else {
        switch EPid {
            case DLEventID.hardBraking: // Hard Braking
                guard let obj = object as? DLHardBrakingData else { return }
                if let messageHeader = obj.header {
                    print("Latitude: \(String(describing: messageHeader.lattitude)), Longitude: \(String(describing: messageHeader.longitude))")}
                if let startSpeedInMph = obj.initialSpeed, let endSpeedInMph = obj.finalSpeed, let maxDecelInMphrSec = obj.maxBraking {
                print("\nHard Brake Event:- StartSpeed(MpHr): \(startSpeedInMph), EndSpeed(MpHr): \(endSpeedInMph), MaxDecel: \(maxDecelInMphrSec)")}
            default:
                print("Event Pids")
        }}
}
```
At a time, app can request maximum 5 pids in one array. <br />

Unregister Event Pids to stop receiving updates:
```
/**
 - parameter pids: Array of Int(where Int is Pid Id. Eg. DLEventID.hardBraking)
 - returns: true or false
*/
unregisterEventPid(pids: [Int])-> Bool
```
<br />

Request Data Pids(Vehicle Speed) using Advanced channel for continous update.

```
/**
 - parameter DPid: constant Int value
 - parameter pids: Array of Int(where Int is Pid Id. Eg. DLCommandPId.basic.vehicleSpeed)
*/

gateway.registerDataPid(DPid: 1, pids: [DLCommandPId.basic.vehicleSpeed])
```

NOTE: App can register multiple Data pids at a same time in array. But if one Data pid fails to register, then all the other pids are failed to register.<br /><br />

SDK uses following method to respond with received Data Pid's data:

```
func onDataPidDataReceived(responseCode: Int, DPid: Int, hashmap: [Int : DLBasicPIDObject]) {
        if responseCode != 0 {
            print("Received Failing Response Code")
            return }
        for (pid, object) in hashmap {
            switch pid {
            case DLCommandPId.basic.vehicleSpeed:
                guard let speed = object as? DLVehicleSpeed else {
                    return}
                if let value = speed.value {
                    print("\nVehicle Speed: \(value)")}
            default:
                print("Data Pid received")
            }
        }
    }
```

Unregister Data Pids to stop receiving updates:
```
/**
 - parameter DPid: constant Int value that is used to register Data Pids
 - parameter pids: Array of Int(where Int is Pid Id. Eg. DLCommandPId.basic.vehicleSpeed)
 - returns: true or false
*/
unregisterDataPid(DPid: Int, pids: [Int]) -> Bool
```

# UDP Channel

Datalogger logs every event occured in realtime and sends it via UDP channel when datalogger is connected to mobile app. 
App cannot register or unregister for UDP events.<br />
Bydefault mobile app acts as pass thru to send UDP events to Danlaw Server. But datalogger with BLEAP configuration, UDP Events are delivered to mobile app.

- Data that can be received using UDP channel:<br />
 1. Event PIDs (id: 256 and over) (Refer Page.57 of Danlaw SmartConnect Installation guide)<br />
 Note: Event PIDs have to be preconfigured in datalogger to receive udp events.
 
- No registration is required to receive UDP events. Implement `DLBleapInterface` and `DLBleapUDPDataDelegate` to receive UDP events sent by Device:
<br />

1. Get instance of Bleap Interface

```
do {
    var bleapInterface:DLBleapInterface
    try self.bleapInterface = DLBleapInterface.getInstance()
    bleapInterface.setDelegate(delegate: 'instance of implementing class of DLBleapUDPDataDelegate')
}catch DLException.SdkNotAuthenticatedException(let error) {
    print(error)
}
 ```

2. Set this flag to “false” to send acknowledgement manually.(Default value: true)

```bleapInterface.onAutoSendAcknowledgement(onSendAcknowledgement: false)```

DLBleapUDPDataDelegate method to receive Parsed UDP Data:

```
func onBleapUDPDataParsed(udpMessages: [UDPMessage], acknowledgementId: Data) {
    
    /// Send acknowledgement here if "onSendAcknowledgement" is "false"
    
    bleapInterface.udpPacketReceivedAcknowledgement(acknowledgementId: acknowledgementId)
    
    for message in udpMessages {
       let messagePayload = message.messagePayload
       let messageId = message.messageType
       switch messageId {
          case DLEventID.GPSMessage:
              guard let obj = messagePayload as? DLGPSMessage else { return }
              guard let messageHeader = obj.header else { return }
              if let time = messageHeader.time, let lat = messageHeader.lattitude, let long = messageHeader.longitude {                         print(“Latitude: \(lat), Longitude:\(long), time: \(time)”) }
               if let satellite = obj.noOfSatellite {
                  print(“No of Satellites: \(satellite)”)}
           // Handle more events here
          default:
            return       
        }              
     } 
}

```
If app fails to send acknowledgement to datalogger, datalogger will keep sending same data again.<br />
Once acknowledgement is sent to datalogger, datalogger will erase that data from its memory


# FAQ
-	**No such module ‘SmartConnectSDK’**  

    <br />Go to project's target settings. Make sure SDK is imported in General-> Framework, Libraries and Embedded content as well as in Build Phases-> Link Binary with Library and Build Phases-> Embed frameworks.
  
-	**Module compiled with Swift 5.1 cannot be imported by the Swift compiler 5.0.1**

    <br />Please make sure you are using Xcode11+ to compile and build this app. If you need support for older verison of Xcode, please contact mobile@danlawinc.com.
  
-	**“App” requires provisioning profile.**
    
    <br />Please make sure you have provided Developer team and valid code signing info in project's settings. You can try 'Automatically manage signing'. SDK is not code signed. Select 'Embed & Sign' in imported Framework's settings.


# Credits
SmartConnect sample app and SmartConnectSDK is owned by Danlaw Inc. A valid license is required to use Danlaw’s Smart Connect products. Licenses are issued by Danlaw on an annual basis for a rolling twelve-month effective time period. License fees established by Danlaw are comprised of a baseline minimum fee, plus a per device fee for each active device at the time of the annual Smart Connect license renewal. Please contact mobile@danlawinc.com for the Key and Licensing information.
