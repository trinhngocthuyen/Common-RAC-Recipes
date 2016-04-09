# Common-RAC-Recipes
Common recipes for ReactiveCocoa

### Table of contents
1. [Overview](#overview)
	a. [Common types](#common-types)
	b. [Common operators](#common-operators)
	c. [Legacy RAC 2.x](#legacy-rac2)
2. [Examples: Common case studies](#examples)
	a. [Text validation](#examples-1)
	b. [Binding](#examples-2)
	c. [Tasks management](#examples-3)
3. [Resources](#resources)

Note: Some codes in this article are writtin in pseudo code. Some properties such as `rac_textProducer, rac_enable, rac_hidden` are custom extension, not originally supported in RAC. You could also write those extensions. For reference, take a look at <https://github.com/neilpa/Rex>

<a name="overview"></a>
## Overview
<a name="common-types"></a>
### Common types
#### Signal vs. SignalProducer
* `Signal`: Hot observable. When a signal is initialized, it emits value immediately.
* `SignalProducer`: Cold observable. Next value is only emitted if the signalProducer started.
*Personally, I prefer SignalProducer than Signal.*

#### Create Signal, SingalProducer
Common ways:

* Init with value/values:

```
let producer1 = SignalProducer<String, NSError>(value: "abc")
let producer2 = SignalProducer<String, NSError>(values: ["abc", "xyz"])
```

* Init with a start handler (closure):

```
let producer = SignaProducer<String, NSError> { observer, disposable in
	delay(1) {
		observer.sendNext("abc")
		observer.sendCompleted()
	}
}
```

* Use operator `pipe` to create a tuple (signal, observer):

```
// Sometimes we want to send a value to observer outside the SignalProducer
// but it's hidden inside the SignalProducer

let (signal, observer) = Signal<String, NSError>.pipe()
let producer = SignalProducer(signal: signal)
```

* Use operator `lift`: Haven't use it before but they use it so often in the source code. Gonna inspect it & update later.

<a name="common-operators"></a>
### Common operators
To understand these operators in a straightforward & vivid way, I recommend you guys take a look at this website: <http://neilpa.me/rac-marbles/>
#### map, filter, reduce

```
// deviceProducer: SignalProducer<Device, NSError>
let connectedDeviceProducer = deviceProducer.filter { $0.isConnected }
```
```
// stringProducer: SignalProducer<Device, NSError>
let validStringProducer = stringProducer.map { $0.characters.count > 6 }
```
```
// dataProducer: SignalProducer<NSData, NSError>
// In this case, we only care about the final data (combined by different parts)
let downloadProducer = data.reduce(NSData(), { combineData($0, $1) })
```
Note: There is a slight different between `reduce` and `scan`. The `reduce` producer only emits the **last** *combined-value* before completed while the `scan` producer emits *combined-value* everytime a next value is sent.

#### flatMap
Widely used in chaining tasks. Some examples:

```
// APIs
class APIs {
	func fetchAccessToken(email: String, _ password: String) -> SignalProducer<AccessToken, NSError>
	func fetchUserInfo(accessToken: AccessToken) -> SignalProducer<User, NSError>
}

let accessTokenProducer = APIs.fetchAccessToken("john@doe.com", "p@ssWord")
let userInfoProducer = accessTokenProducer.flatMap(.Latest) { accessToken in
	APIs.fetchUserInfo(accessToken)
}
```

#### combineLatest, zip
CombineLatest is commonly used in such features: text validation, live-search with multiple criteria...

```
let validEmailProducer = emailTextField.rac_textProducer.filter { isValidEmail($0) }
let validPasswordProducer = passwordTextField.rac_textProducer.filter { isValidPassword($0) }
let validEmailAndPasswordProducer = validEmailProducer.combineLatestWith(validPasswordProducer)
let validInputProducer = validEmailAndPasswordProducer.map { $0 && $1 }

// validEmailAndPasswordProducer:	SignalProducer<(Bool, Bool), NSError>
// validInputProducer: 				SignalProducer<Bool, NSError>
```
There is also a free function `combineLatest` if it fits your coding style.

`zip` is a bit different. Take a look at <http://neilpa.me/rac-marbles/#zip> for reference. 

#### takeUntil
```
// We want to update data from local first, 
// until fetching data from network is successful

let localDataProducer = Service.fetchDataFromLocal()
let networkDataProducer = Service.fetchDataFromNetwork()
let dataProducer = localDataProducer.takeUntilReplacement(networkDataProducer)
```
```
// We want to fetch data from network,
// But the task could be canceled by an action (ex: tap cancel button)

let networkDataProducer = Service.fetchDataFromNetwork()
let cancelProducer = cancelButton.action.producer
let cancelableNetworkDataProducer = networkDataProducer.takeUntil(cancelProducer)
```

#### skip, skipUntil


#### then


#### retry
```
// We want to retry fetching user info if it fails

let accessTokenProducer = APIs.fetchAccessToken(email, password)
let userInfoProducer = accessTokenProducer.flatMap(.Latest) { accessToken in
	APIs.fetchUserInfo(accessToken).retry(1)
}
```

#### throttle

```
// We want to delay a few seconds before searching
// The value changed withing the throttle interval will be skipped
// Expected: If we continously type 1 digit every second, the next value would be
// 1 -> 123 -> 12345 -> 1234567 -> ...

let searchTextProducer = textField.rac_textProducer
	.throttle(2, onScheduler: QueueScheduler.mainQueueScheduler)

```

#### scan
```
// We want to append new scanned device to a list of total scanned devices
// Every change on the list should notify its obersevers

let newScannedDeviceProducer = Service.scanDevices()
let totalScannedDevicesProducer = newScannedDeviceProducer.scan([], { $0 + [$1] })
```

#### flatMapError (aka catch in RAC 3.0)
Usually used to trigger another task when the current task is failed.

```
// We want to erase temporary downloaded cache data if the task fails

let dataProducer = Service.downloadData(url)
let dataEraseIfNeededProducer = data.flatMapError { _ in Service.eraseDownloadedCachedData() }
```

We could also use this operator to ignore error. You would use this in binding data because sometimes it requires `SignalProducer<SomeType, NoError>` instead of `SignalProducer<SomeType, NSError>`.

```
// Sometimes we don't care if a task fails or not

let searchTextProducer = textField.rac_textProducer.flatMapError { _ in SignalProducer<String, NoError>.empty }
```

<a name="legacy-rac2"></a>
### Legacy: RAC 2.x (Objective-C style)
* So far now (RAC 4.0), RAC consists of both Swift code and ObjC code. You could find ObjC code with prefix: RAC, ex: `RACSignal`, `RACCommand`... Try to avoid using ObjC types if possible. Instead, convert them to corresponding types in Swift. For instance, `RACSignal` (in ObjC) corresponds to `Signal` (in Swift). `RACSignal` has a method `toSignalProducer()` to transform it to Swift things.

=====
<a name="examples"></a>
## Examples: Common case studies
<a name="examples-1"></a>
### [1] Text validation
Keywords: `combineLatestWith`

See operator combineLatestWith for examples.

<a name="examples-2"></a>
### [2] Binding Text validation
Keywords: `MutableProperty`, `<~`

```
loginButton.rac_enable <~ validInputProducer
```
```
helloLabel.rac_text <~ textField.rac_textProducer.map { "Hello " + $0 }
```
```
helloLabel.rac_text <~ textField.rac_textProducer.map { "Hello " + $0 }
```
```
loadingMoreButton.rac_hidden <~ viewModel.isLoadingProducer.map { !$0 }
```
Binding with CocoaAction (aka RACCommand in RAC 2.x): update later

<a name="examples-3"></a>
### [3] Task management Text validation
#### Network request

```
func fetchAccessToken(email: String, _ password: String) -> SignalProducer<AccessToken, NSError> {
	return SignalProducer { observer, disposable in
		// Make request, parse response JSON
		if let error = errorOccurred() {
			sendFailed(error)
        } else {
           let result = parseJSONAndGetResult()
           observer.sendNext(result)
           observer.sendCompleted()
        }
	}
}
```
You could write

```
func fetchAccessToken(email: String, _ password: String) -> SignalProducer<AccessToken, NSError> {
	// Make request, parse response JSON
	if let error = errorOccurred() {
		return SignalProducer<AccessToken, NSError>(error: error)
    } else {
        let result = parseJSONAndGetResult()
        return SignalProducer<AccessToken, NSError>(value: result)
    }
}
```

#### Chaining tasks
See Common operators (`flatMap`) for examples.

#### Combine tasks
A few senarios:

* Task 1 starts and end when task 2 starts. Besides, we don't want to observe value of task2

``` 
task1.takeUntil(task2)

// In this case, task2 must be of type: SignalProducer<(), NSError>
// In case the type of task2 does not satisfy, we could map task2 to the appropriate type

task1.takeUntil(task2.map { _ in () })

```

* Task 1 starts and end when task 2 starts. Besides, we want to observe value of task2. Ex: fetchDataFromLocal and fetchDataFromServer
 
``` 
// In this case, task1 & task2 must have the same type

task1.takeUntilReplacement(task2)

```

* Task1 starts and end when an action happens (ex: tap a button, viewDidDisappear...): 
	* Create a signalProducer and `sendNext` to it when the action takes place
	* Use this signalProducer as a trigger with operator `takeUntil`

#### Task with retry, timeout
See Common operators (`retry`) for examples.

=====
<a name="resources"></a>
## Resources
 
[1] ReactiveCocoa - Framework Overview
<https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/Documentation/FrameworkOverview.md>

[2] ReactiveCocoa - Design Guidelines
<https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/Documentation/DesignGuidelines.md>

[3] RAC Marbles
<http://neilpa.me/rac-marbles>

[4] ReactiveCocoa source codes. They did well document every function

[5] MVVM with ReactiveCocoa <http://blog.scottlogic.com/2015/05/15/mvvm-reactive-cocoa-3.html>




