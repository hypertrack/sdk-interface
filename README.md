# HyperTrack Generation SDK

The SDK interface methods are grouped under a data type `HyperTrack`. All calls would be in form of `HyperTrack.doSomething`. Ideally it should be impossible to create an instance of `HyperTrack` (in Swift this can be achieved by making `HyperTrack` an enum without cases or by making initializers private).

## Interface

All interface functions/variables should be static on `HyperTrack`.

If the developer integrating our SDK forgot to make some app configuration action required for SDK integration, the SDK should immediately crash on app launch with a helpful message navigating the user to proper integration.

(Previously we outputed a statically defined error in runtime. This was forcing us to update the major version of the SDK every time that enum changed and the developer was free to push to production with incomplete integration. Crashing the app here is appropriate, the dev shouldn't proceed without properly integrating the app. This serves as a dynamic hidden output of the SDK that we can change without breaking public API.)

The list of required integration points:
- [Android, iOS] The developer needs to define a publishable key string in Info.plist on iOS and in the manifest in Android under key: `HyperTrackPublishableKey`. If the key is not found when the user is interfacing though functions and variabled defined below, the SDK should crash with message: "Please set the HyperTrack Publishable Key that can be obtained at https://dashboard.hypertrack.com/setup as `HyperTrackPublishableKey` in Info.plist/App Manifest" (Include `Info.plist` on iOS and the manifest part of the string on Android)
- [iOS]	“Location updates” and “Remote notifications” Background Capabilities. Crash with: "Please add “Location updates” and “Remote notifications” modes in Target > Signing & Capabilities > Background Modes"
- [iOS] Push notifications capability. Crash with "aps-environment". Crash with: "Please add “Push notifications” capability in Target > Signing & Capabilities"
- [iOS] NSLocationWhenInUseUsageDescription string should be present. Crash with "Please add “NSLocationWhenInUseUsageDescription” key in Info.plist if you are going to ask for When In Use locaton permissions and an additional “NSLocationAlwaysAndWhenInUseUsageDescription” if you are going to ask for Always location permissions"

### Automatically Request Permissions

An Info.plist/Android Manifest flag `HyperTrackAutomaticallyRequestPermissions`, boolean. Default - true.

Documentation: if `true`, SDK automatically triggers location and motion activity permissions dialogs when tracking starts.
It's recommended to set this to `false` and to use the `errors` API to handle permissions manualy.

### Allow Mock Locations

An Info.plist/Android Manifest flag `HyperTrackAllowMockLocations`, boolean. Default - false.

Documentation: if `true`, SDK allows simulated locations. This can be useful while developing the app. Make sure to remove this flag in production.

### Logging

An Info.plist/Android Manifest flag `HyperTrackLoggingEnabled`, boolean. Default - false.

Documentation: set to `true` for debugging assist.

### Variables

By variables, we mean a group of functions for accessing or setting a data type. Some languages have special syntax that emulates regular variable/constant access for them, like:
- Setting: `HyperTrack.variable = value`
- Getting: `let value = HyperTrack.variable`

Other languages group those concepts as accessor functions:
- Setting: `HyperTrack.setVariable(value)`
- Getting: `let value = HyperTrack.getVariable()`

Below, all variables are named using the variable/constant access, but the proper convention in the target language should be chosen in each platform.

#### Device ID

`deviceID`. Read-only. Returns a non-optional/non-null/non-empty String data type representation in the target language.

Documentation: A string used to identify a device uniquely.

#### Name

`name`. Read-write. Returns and accepts a string.

Documentation: You can see the device name in the devices list in the Dashboard or through APIs.

#### Metadata

`metadata`. Read-write. Returns a `[String: JSON]` datatype, that is the strictest possible JSON representation in the target language. Should allow easy construction of correct values in the target language, convenience ways to convert to the JSON string if needed. For example in Swift it's non-optional recursive coproduct datatype with cases:
```swift
enum JSON {
  indirect case object([String: JSON])
  indirect case array([JSON])
           case string(String)
           case number(Double)
           case bool(Bool)
           case null
}
```

#### Location

`location`. Read-only. Returns a non-optional generic coproduct datatype `Result<Location, LocationError>` (enum/sealed class/discriminating union/tagged union) that consists of constructors (it's ideal if we use the default abstraction that comes with the language here):
- `success` with a non-optional `Location` datatype, that has two read-only properties `latitude` of type Double and `longitude` of type Double. It's best if this datatype has some convenicence functions on it that helps the users in using this type where location is expected (GeoJSON?)
- `failure` with a non-Optional `LocationError` coproduct datatype consisting of the following cases:
	// The SDK is not collecting locations because it’s neither tracking nor available.
	- `notRunning`
	// The SDK started collecting locations and is waiting for OS location services to respond.
	- `starting`
	// Errors blocking the SDK from tracking
	- `errors` with an associated type: Set<HyperTrackError>, same as `HyperTrack.errors` API

#### Availability

`isAvailable`. Read-write. Accepts and returns a bool value.

Documentation: Device's availability for nearby search.

Implementation detail: call `availability` for getting and setting on iOS and `getAvailability`/`setAvailability` on Android

#### Tracking

`isTracking`. Read-only. Returns a bool.

Documentation: Reflects tracking intent.

#### Errors

`errors`. Read-only. Returns a Set of `HyperTrackError` product data types (struct/class/object/tuple) with the following fields:
- `string`: NonEmptyString
  Documentation: A string that uniquely identifies the error.

It should be impossible to construct by the user. Multiple errors can be returned at one if makes sense (location and motion errors, etc). This design leaves doors open for HyperTrack to add new errors without breaking API changes. 

Documentation: A set of errors that can prevent the app from successfuly tracking. Can be accessed at any point in the app's lifetime. It's best to check with this API before opening the screens where user is expecting to be tracking and handling all the errors until this API returns an empty Set. All error codes and their explanation can be found in HyperTrack documentation portal.

The users can also see the current list of errors by consulting the static constants in HyperTrack.HyperTrackError (the name should be used for the name of the static version and for the text contents of the `string`):
 // GPS satellites are not in view.
 - `gpsSignalLost`
 // The user enabled mock location app while mocking locations is prohibited.
 - `locationMocked`
 // The user denied location permissions.
 - `locationPermissionsDenied`
 // Can’t start tracking in background with When In Use location permissions. SDK will automatically start tracking when app will return to foreground.
 - `locationPermissionsInsufficientForBackground`
 // The user has not chosen whether the app can use location services. SDK automatically asked for permissions.
 - `locationPermissionsNotDetermined` [iOS only]
 // The user didn’t grant precise location permissions or downgraded permissions to imprecise.
 - `locationPermissionsReducedAccuracy`
 // The app is in Provisional Always authorization state, which stops sending locations when app is in background.
 - `locationPermissionsProvisional` [iOS only]
 // The app is not authorized to use location services.
 - `locationPermissionsRestricted` [iOS only]
 // The user disabled location services systemwide.
 - `locationServicesDisabled`
 // The device doesn't have location services.
 - `locationServicesUnavailable` [Android only]
 // Motion activity permissions denied after SDK’s initialization. Granting them will restart the app, so in effect, they are denied during this app’s session.
 - `motionActivityPermissionsDenied`
 // The user has not chosen whether the app can use motion activity services. SDK automatically asked for permissions.
 - `motionActivityPermissionsNotDetermined` [iOS only]
 // The user disabled motion services systemwide.
 - `motionActivityServicesDisabled` [iOS only]
 // The device doesn't have motion activity services.
 - `motionActivityServicesUnavailable` [iOS only]
 // The SDK is not collecting locations because it’s neither tracking nor available.
 - `notRunning`
 // The SDK started collecting locations and is waiting for OS location services to respond.
 - `starting`
 // HyperTrack Publishable key is invalid.
 - `invalidPublishableKey`
 // The SDK was remotely blocked from running.
 - `blockedFromRunning`

### Functions

#### Start Tracking

`startTracking`. Has no arguments, returns nothing.

Documentation: Expresses an intent to start location tracking.

Implementation detail: calls `start()` on Android and iOS.


#### Stop Tracking

`stopTracking`. Has no arguments, returns nothing.

Documentation: Expresses an intent to stop location tracking.


Implementation detail: calls `stop()` on Android and iOS.

#### Add Geotag

`addGeotag`. Accepts JSON.Object datatype described above for metadata. Returns a location that the geotag was attached to as `Result<Location, LocationError>`, same as `location` API. If the user doesn't want the result of this operation, it can be safely discarded without compiler complainint (`@discardableResult` in Swift).

#### Sync

`sync`. Has no arguments, returns nothing. 

Documentation: Manually synchronizes tracking and availability intents with the cloud. Useful if you are starting tracking, making device available or creating a trip through your backend and want the device to know about this intent immediately after.

#### Location Subscription

`subscribeToLocation` (the name depends on target language conventions for subscriptions). A static function that accepts a callback closure `(Result<Location, LocationError>) -> Void` that returns a current value upon subscription and de-duplicated changes afterwards. The function returns a function with the shape `() -> Void`, when calling this function the subscription stops. Not storing this function leads to subscription being automatically stopped. Or a different, preffered pattern in the target language.

#### Availability Subscription

`subscribeToAvailability` (the name depends on target language conventions for subscriptions). A static function that accepts a callback closure `(Bool) -> Void` that returns a current value upon subscription and de-duplicated changes afterwards. The function returns a function with the shape `() -> Void`, when calling this function the subscription stops. Not storing this function leads to subscription being automatically stopped. Or a different, preffered pattern in the target language.

#### Tracking Subscription

`subscribeToTracking` (the name depends on target language conventions for subscriptions). A static function that accepts a callback closure `(Bool) -> Void` that returns a current value upon subscription and de-duplicated changes afterwards. The function returns a function with the shape `() -> Void`, when calling this function the subscription stops. Not storing this function leads to subscription being automatically stopped. Or a different, preffered pattern in the target language.

#### Errors Subscription

`subscribeToErrors`. (the name depends on target language conventions for subscriptions). A static function that accepts a callback closure `(Set<HyperTrackError>) -> Void` that returns a current value upon subscription and de-duplicated changes afterwards. The function returns a function with the shape `() -> Void`, when calling this function the subscription stops. Not storing this function leads to subscription being automatically stopped. Or a different, preffered pattern in the target language.
