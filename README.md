```mermaid
flowchart TD
    subgraph Smartphone ["USER'S SMARTPHONE"]
        direction TB
        
        subgraph App ["AURA ANDROID APP"]
            subgraph UI ["UI LAYER (Jetpack Compose)"]
                ScanScreen
                ConnectedScreen
                BoschScreen
                MediaScreen
                
                ScanScreen <-->|NavGraph| ConnectedScreen
                ConnectedScreen --> BoschScreen
                ConnectedScreen --> MediaScreen
            end
            
            subgraph ViewModel ["VIEWMODEL LAYER (Hilt)"]
                ControlViewModel
                ScanViewModel
                MediaViewModel
            end
            
            subgraph Data ["DATA LAYER (@Singleton)"]
                HeadUnitGattManager
                BluetoothScanManager
                BluetoothSPPManager
                IdentificationEngine
                UserVolumePreferences
                LocalMediaRepository
                ModuleRegistry
                MediaSessionController
            end
            
            subgraph Services ["ANDROID SERVICES (Foreground)"]
                BluetoothConnectionService
                SmartPlayPlayerSvc
            end
            
            UI --> ViewModel
            ViewModel --> Data
            Data --> Services
        end
    end
    
    subgraph Bluetooth ["Android Bluetooth Stack"]
        BLE["BLE GATT (Volume control)"]
        BTClassic["BT Classic (AVRCP / A2DP / SPP)"]
    end
    
    App --> Bluetooth
    
    subgraph HeadUnit ["BOSCH HEAD UNIT (In-Car)"]
        subgraph GATTServer ["BLE GATT Server"]
            HandshakeSvc["Handshake Svc"]
            VolumeRW["Volume R/W Char"]
            CCCD["CCCD (notify)"]
        end
        subgraph BTClassicHU ["BT Classic"]
            A2DP["A2DP (audio)"]
            AVRCP["AVRCP (media)"]
            SPP["SPP (telemetry)"]
        end
    end
    
    BLE --> GATTServer
    BTClassic --> BTClassicHU
```




```mermaid
stateDiagram-v2
    [*] --> Screen.Scan : App Launch
    
    Screen.Scan --> Screen.Connected : User taps Connect\n(connectionState = Identifying or Connected)
    
    state Screen.Connected {
        [*] --> AppTab.HOME
        AppTab.HOME --> AppTab.MEDIA : Tab Navigation\n(State-based)
        AppTab.MEDIA --> AppTab.HOME
    }
    
    Screen.Connected --> Screen.Scan : Disconnect button\n(GATT terminated,\npopUpTo(0))
    
    note right of Screen.Connected
        Back Button (while connected):
        moveTaskToBack(true)
        GATT stays alive, app backgrounded
        (NOT disconnect)
    end note
```


```mermaid
stateDiagram-v2
    [*] --> IDLE : App start
    
    IDLE --> SCANNING_DEVICES : connectToHeadUnit()
    SCANNING_DEVICES --> CONNECTING : tryNextDevice() [connectGatt()]
    
    CONNECTING --> CONNECTING : currentDeviceIndex++\ntryNextDevice() (Not HU)
    CONNECTING --> CONNECTED : CCCD confirmed (status=0)\nInitial volume written
    
    state CONNECTED {
        [*] --> BidirectionalSync
        BidirectionalSync : Volume notifications active
    }
    
    CONNECTED --> RetryState : CCCD status=133\n(HU session reset)
    
    state RetryState {
        scheduleRetryForCurrentDevice
        isRetrying_true : isRetrying = true
        bluetoothGatt_close : bluetoothGatt.close()
        wait_2500ms : wait 2500ms
        
        scheduleRetryForCurrentDevice --> isRetrying_true
        isRetrying_true --> bluetoothGatt_close
        bluetoothGatt_close --> wait_2500ms
    }
    
    RetryState --> CONNECTING : tryNextDevice() [same idx]
    RetryState --> ERROR : All retries failed
    CONNECTING --> ERROR : No bonded devices found
    
    SCANNING_DEVICES --> IDLE : disconnect() called
    CONNECTING --> IDLE : disconnect() called
    CONNECTED --> IDLE : disconnect() called
    ERROR --> IDLE : disconnect() called
```



```mermaid
classDiagram
    class BoschHeadUnitGATTServer {
        <<Device>>
    }
    class HandshakeService {
        UUID: 9781b0a6-...
    }
    class HandshakeCharacteristic {
        UUID: 9781b0a7-...
        WRITE 0x01 to authenticate
    }
    class VolumeService {
        UUID: [Volume Service]
    }
    class VolumeReadCharacteristic {
        UUID: 11d66fa7-...
        READ | NOTIFY
    }
    class CCCDDescriptor {
        UUID: 00002902-...
        Write ENABLE_NOTIFICATION
    }
    class VolumeWriteCharacteristic {
        UUID: 11d66fa8-...
        WRITE [0x00, volumePercent]
    }
    
    BoschHeadUnitGATTServer *-- HandshakeService
    HandshakeService *-- HandshakeCharacteristic
    BoschHeadUnitGATTServer *-- VolumeService
    VolumeService *-- VolumeReadCharacteristic
    VolumeReadCharacteristic *-- CCCDDescriptor
    VolumeService *-- VolumeWriteCharacteristic
```



```mermaid
sequenceDiagram
    participant Phone
    participant HU as Head Unit
    
    Note over Phone: t=0ms
    Phone->>HU: connectGatt(TRANSPORT_LE)
    
    HU-->>Phone: onConnectionStateChange(STATE_CONNECTED)
    Note over Phone: gatt.refresh() (clears cache)
    
    Note over Phone: t=+500ms
    Phone->>HU: discoverServices()
    HU-->>Phone: onServicesDiscovered()
    
    Note over Phone: check HANDSHAKE_SERVICE_UUID
    Phone->>HU: sendHandshake(0x01)
    HU-->>Phone: onCharacteristicWrite(HANDSHAKE, status=0)
    
    Note over Phone: t=+600ms
    Phone->>HU: write(CCCD, ENABLE_NOTIFICATION_VALUE)
    
    alt status = 0 (Success)
        HU-->>Phone: onDescriptorWrite success
        Note over Phone: t=+1000ms
        Phone->>HU: writeInitialVolumePayload(savedVolume%)
        Note over Phone: state → CONNECTED
    else status = 133 (HU terminated link / session reset)
        HU-->>Phone: status=133
        Note over Phone: isRetrying = true
        Note over Phone: gatt.close()
        Note over Phone: t=+2500ms
        Phone->>HU: tryNextDevice() (repeat from connectGatt)
    end
```



```mermaid
sequenceDiagram
    participant A as Phone A
    participant Android
    participant B as Phone B
    participant HU as Head Unit
    
    rect rgb(255, 200, 200)
        Note right of A: PROBLEM (before fix)
        A->>Android: connects
        Note over Android: caches: {CCCD handle = 0x0021}
        A->>Android: disconnects
        
        B->>Android: connects
        Note over Android: reads from cache: {CCCD handle = 0x0021} (STALE!)
        B->>Android: gatt.writeDescriptor(handle=0x0021, ENABLE_NOTIFICATIONS)
        Android->>HU: Write Handle 0x0021
        Note over HU: HU sees: unknown handle\n(session table is fresh)
        HU-->>Android: Silently ignores write
        Note over B: Volume notifications never start\nPhone B has no volume control
    end
    
    rect rgb(200, 255, 200)
        Note right of A: SOLUTION
        B->>Android: onConnectionStateChange(STATE_CONNECTED)
        Note over B: gatt.refresh()\n(force-clears Android's GATT cache)
        Note over B: delay(500ms)\n(wait for cache to propagate)
        B->>Android: gatt.discoverServices()
        Android->>HU: Read services
        HU-->>Android: Returns FRESH handles
        B->>Android: writeDescriptor(correct_handle, ENABLE)
        Android->>HU: Write correct handle
        HU-->>Android: Success
        Note over B: CCCD write succeeds\nVolume notifications start ✅
    end
```
## Getting started
* Fork this repository (Click the Fork button in the top right of this page, click your Profile Image)
* Clone your fork down to your local machine

```markdown
git clone https://github.com/your-username/hacktoberfest.git
```

* Create a branch

```markdown
git checkout -b branch-name
```

* Make your changes 
* Commit and push

```markdown
git add .
git commit -m 'Commit message'
git push origin branch-name
```

* Create a new pull request from your forked repository (Click the `New Pull Request` button located at the top of your repo)
* Wait for your PR review and merge approval!
* __Star this repository__ if you had fun!


## Happy new year message.

Hey! new year eve are comming, be prepared to the "FEAST"
Having fun is the first way you think but don't forget to make a wish.

`So let's share our wish.`

## Getting started
* Fork this repository (Click the Fork button in the top right of this page, click your Profile Image)
* Clone your fork down to your local machine

```markdown
git clone (copy the repo link here)
```

* Create a branch

```markdown
git checkout -b branch-name
```

* Make your changes (choose from any task below)
* Commit and push

```markdown
git add .
git commit -m 'Commit message'
git push origin branch-name
```

* Create a new pull request from your forked repository (Click the `New Pull Request` button located at the top of your repo)
* Wait for your PR review and merge approval!
* __Star this repository__ if you had fun!


## Want to wish ? :tada:

Go to USER.json, follow the structure, and that's it.
___
- Your name ```"name":"julkwel"```

- Your image link, (like fb profile, ... , preference github image link) ```"image":"image_link"```

- Your wishes ```"message":"your_message"```

- Github url ```"username":"github_username"```

- Flag : see [this link](http://flag-icon-css.lip.is/?continent=Africa) , eg: `mg` for Madagascar

___

## FEATURE

1. Translation


*Code for fun :blush:* 
