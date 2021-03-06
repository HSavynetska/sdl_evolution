# AOA multiplexing similar to Android BT/SPP multiplexing

* Proposal: [SDL-0095](0095-AOA-multiplexing.md)
* Author: [Jennifer Hodges](https://github.com/jhodges55), [Shinichi Watanabe](https://github.com/shiniwat)
* Status: **In Review**
* Impacted Platforms: [ Android ]


## Introduction

This proposal aims to add AOA multiplexing capability to SdlRouterService.  

## Motivation

Currently SdlRouterService works for MultiplexTransport, which supports Bluetooth transport as the underlying transport. This proposal extends SdlRouterService to support both Bluetooth and AOA (Android Open Accessory) transports, so that multiple SDL applications can share the AOA transport without having to worry about USB connection.
AOA protocol basically allows only a single application to access the USB accessory -- if multiple applications are designed to use the same accessory, the Android system asks which application should use the accessory and shows a permission dialog. This proposal allows multiple SDL applications to share the USB accessory.
 
## Proposed solution

Currently, SDL app chooses the transport (either Bluetooth multiplexing, legacy Bluetooth, USB, or TCP), by specifying BaseTransportConfig when it launches SdlProxyALM instance.
 This proposal adds yet another transport (= AOA multiplexing), and SDL app can specify which transport to use explicitly. 
 Current SdlRouterService works for Bluetooth multiplexing, and this proposal basically extends the SdlRouterService to support both Bluetooth multiplexing and AOA multiplexing.
 To do that, the basic idea is:
- MultiplexTrannsportConfig is extended to be aware of TransportType. A new TransportType "MULTIPLEX_AOA" is supported in addition to "MULTIPLEX".
- SDL application specifies TransportType when configuring MultiplexTransportConfig.
- SdlRouterService internally holds separate Handlers according to the TransportType, and returns the right Handler when binding to the service, so that IPC channel is isolated per TransportType.
- SdlRouterService recognizes the expected TransportType when registering the app to router. SdlRouterService internally recognizes the expected TransportType per session.
- Currently RegisteredApp uses the same TransportType for all sessions, but this will be extended by upcoming simultaneous multiple transports support. RegisteredApp will be extended to support multiple transports per session.
- SdlRouterService uses either MultiplexBluetoothTransport or (newly added) MultiplexAoaTransport depending on TransportType.
- In order to share the UsbAccessory with multiple applications, the app that actually has the USB accessory's permission will send ParcelFileDescriptor to the active router service. If no active router is running, the local app launches router service for AOA transport.
- The affected component is Android proxy only. No change is required for SDL Core.
 
The affected classes in Android Proxy are shown below:

![Class diagram of Android Proxy](../assets/proposals/0095-AOA-multiplexing/0095-aoa-multiplexing-class-diagram.png)

**Fig. 1: affected classes in Android Proxy**


## Detailed design
### Extends MultiplexTransportConfig
Extends MultiplexTransportConfig by adding TransportType. Current MultiplexTransportConfig assumes the underlying transport is Bluetooth only, but this proposal introduces TransportType in the class.

```java
public class MultiplexTransportConfig extends BaseTransportConfig{
	TransportType transportType;

	public TransportType getTransportType() {
		return transportType;
	}

	public void setTransportType(TransportType transportType) {
		this.transportType = transportType;
	}
}
```

### SDL app specifies TransportType when using MultiplexTransportConfig.
Specifies TransportType explicitly in MultiplexTransportConfig if the app uses AOA multiplexing. This would be simpler than adding another TransportConfig class from SDL application's perspective.
```java
    transport = new MultiplexTransportConfig(getApplicationContext(), getAppId());
    ((MultiplexTransportConfig) transport).setTransportType(TransportType.MULTIPLEX_AOA);
    proxy = new SdlProxyALM(this,
                    ....
                    transport);
```

### SdlRouterService holds newly added Handler for AOA clients
Extends SdlRouterService by adding AoaRouterHandler:
```java

	final Messenger aoaMessenger = new Messenger(new AoaRouterHandler(this));

	/**
	 * Handler of incoming messages from AOA clients.
	 */
	static class AoaRouterHandler extends Handler {
	    ...
	}
	
	@Override
	public IBinder onBind(Intent intent) {
	    ...
			}else if(TransportConstants.BIND_REQUEST_TYPE_AOA_CLIENT.equals(requestType)) {
				return this.aoaMessenger.getBinder();
			}
	    ...
	}
```
and extends TransportBroker to specify (newly added) action (== BIND_REQUEST_TYPE_AOA_CLIENT) for the binding Intent:
```java
	private boolean sendBindingIntent(){
        ...
		if (mTransportType.equals(TransportType.MULTIPLEX_AOA)) {
			bindingIntent.setAction(TransportConstants.BIND_REQUEST_TYPE_AOA_CLIENT);
		} else {
            ...
		}
        ...
	}
```

### SdlRouterService recognizes expected TransportType
Extends SdlRouterService so that internal class (RegisteredApp) is aware of TransportType.
Because of upcoming support of simultaneous multiple transports, RegisteredApp supports TransportType per session:
```java
	class RegisteredApp {
		/**
		 * This is a simple class to hold onto a reference of a registered app.
		 * @param appId
		 * @param messenger
		 * @param theType
		 */
		public RegisteredApp(String appId, Messenger messenger, TransportType theType){
		    ...
			transportTypes = new HashMap<>();
			transportTypes.put(0L, theType); // default session maps to the given (primary) TransportType
		}
	}
	
	@Override
	public void handleMessage(Message msg) {
	    ....    
        switch (msg.what) {
	        case TransportConstants.ROUTER_REGISTER_CLIENT:
	            RegisteredApp app = service.new RegisteredApp(appId,msg.replyTo, TransportType.MULTIPLEX); // This is the case for Bluetooth. AOA's message handler specifies TransportType.MULTIPLEX_AOA
	    }
	}
```

### SdlRouterService uses either MultiplexBluetoothTransport or MultiplexAoaTransport depending on the TransportType
```java
public class SdlRouterService extends Service {
    ...
    // underlying transport depending on the TransportType.
	private MultiplexBluetoothTransport bluetoothTransport;
	private MultiplexAoaTransport aoaTransport;
    ...
}
```

### The app that has actual UsbAccessory's permission sends ParcelFileDescriptor to active SdlRouterService
We discussed about a potential issue where multiple SDL applications have the same accessory_filter settings.

When a user plugs in their device to an AOA connection, they are prompted with selection dialog. The dialog contains all apps that support the currently connected accessory. In this dialog there is also an option to always open a specific app for that accessory. We basically assumed the app that receives AOA intent gets the accessory permission, and then the app works as the router for AOA transport. if the user picks an app to always receive the AOA intent and it propagates the router service, all multiplexed connections will go through that apps router service regardless if there is a newer router service present. If by chance that is not a trusted router service, no other apps will ever connect until the user clears the flag.

The underlying issue is that the app user chooses to give the accessory permission is not necessarily the trusted router service.
To solve the issue, we take the approach where sending the USB device's descriptor (ParcelFileDescriptor) from user granted app to active router service.
The following sequence diagram shows how it works:

![sequence dialog for possible permanent routerservice issue](../assets/proposals/0095-AOA-multiplexing/0095-aoa-possible-permanent-routerservice-solution.png)

**Fig. 2: Sequence diagram for possible permanent RouterService solution**

## Potential downsides

This feature introduces TransportType for TransportBroker and SdlRouterService. While this approach should have no obvious downsides, backward compatibility should be taken into account as much as we can.
In particular, the following cases need to be confirmed:
- Case #1: Older proxy's TransportBroker binds to new SdlRouterService. In this case, SdlRouterService assumes TransportType == MULTIPLEX for backward compatibility.
- Case #2: Newer proxy's TransportBroker binds to older SdlRouterService. The older SdlRouterService won't support AOA multiplexing. In this case, the expected behavior is "don't bind to older SdlRouterService; instead, start and bind to newer (local) SdlRouterService". We can utilize existing version check and trusted router logic to make this case work.

This feature also increases the IPC transaction between TransportBroker and SdlRouterService. While Android system has Binder Transaction Limit, which is explained at [TransactionTooLargeException Android document](https://developer.android.com/reference/android/os/TransactionTooLargeException.html), we won’t run into TransactionTooLargeException cases in real scenario based on our test, unless underlying transport has a fatal error. The fatal error case would be, for example, the case where we cannot write Bluetooth socket and/or USB's ParcelFileDescriptor for some reason. Those fatal error cases can be discussed outside of this proposal.

## Impact on existing code

All changes will be made to Android Proxy, and there's no change required to SDL Core and/or other components.
The proposed solution keeps existing logic of SdlRouterService intact, and hence there should be no obvious side effect for existing Bluetooth multiplexing.
Because AOA multiplexing application explicitly specifies the transport, existing applications that use other transports should have no impact, as AOA multiplexing transport is not enabled. 

## Alternatives considered

Alternative solution would be to utilize (existing) AltTransport. AltTransport is essentially just another Message Handler provided externally by binding to SdlRouterService. 
If apps are using Bluetooth multiplexing, it appears that existing Bluetooth sessions need to be disconnected first before setting up AltTransport. This proposal takes different approach, so that existing SDL apps don't have to disconnect when another app starts utilizing AOA multiplexing transport.
Logically speaking, however, it is possible to implement the same transport-aware logic by using AltTransport. From router client's perspective, AltTransport consumes additional IPC channel for communicating with SDL Core. So it may have performance overhead when compared with this proposed approach. 
