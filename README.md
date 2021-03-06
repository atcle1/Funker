<img src="https://www.dibiasi.nl/img/funker.png" title="Funker" width="400" height="400">

# Funker

An easy to use bluetooth library for RFCOMM and OBEX on Android. Powered by Rx!

[![](https://jitpack.io/v/ddibiasi/Funker.svg)](https://jitpack.io/#ddibiasi/Funker)


## Features
- Find bluetooth devices
- Send & listen for RFCOMM commands
- OBEX file transfer

These [slides](https://dibiasi.nl/share/funker_pres.pdf) where part of my Bachelors thesis, have a look at them if you want.

## Examples

Search for specific bluetooth devices
```kotlin
finder
    .search(checkPaired = true, indefinite = true)
    .distinct()
    .filter { it.name?.contains("your-device-name", ignoreCase = true) ?: false }
    .subscribeBy(
        onNext = { Log.d(TAG, "Found device: ${it.address}") },
        onError = { Log.e(TAG, "Error occurred ${it.message}") },
        onComplete = { Log.d(TAG, "Completed search") }
    )
```

Send files via OBEX
```kotlin
rxOBEX
    .putFile(file, "remote/directory")
    .subscribeBy(
        onComplete = { Log.d(TAG, "Succesfully sent a testfile to ${device.address}") },
        onError = { Log.e(TAG, "Error occurred ${it.message}") }
    )
    
```

Send RFCOMM commands
```kotlin
rxSpp
    .send("your-command")
    .subscribeBy(
        onComplete = { Log.d(TAG, "Succesfully sent $command to ${device.address}") },
        onError = { Log.e(TAG, "Error occurred ${it.message}") }
     )
        
```

---

## Setup

- Add JitPack to your root build.gradle

```
allprojects {
 		repositories {
 			...
 			maven { url 'https://jitpack.io' }
 		}
 	}
```
- Add Funker to your project level build.gradle
```
dependencies {
        implementation 'com.github.ddibiasi:Funker:0.0.x'
}
```

- Add [RxJava](https://github.com/ReactiveX/RxJava) and optionally [RxAndroid](https://github.com/ReactiveX/RxAndroid), [RxKotlin](https://github.com/ReactiveX/RxKotlin)
```
implementation 'io.reactivex.rxjava2:rxjava:2.x.x'
implementation 'io.reactivex.rxjava2:rxandroid:2.x.x'
implementation("io.reactivex.rxjava2:rxkotlin:2.x.x")
```

- [AutoDispose](https://github.com/uber/AutoDispose) by Uber is recommended but not necessary
```
implementation 'com.uber.autodispose:autodispose:1.x.x'
implementation 'com.uber.autodispose:autodispose-lifecycle:1.x.x'
implementation 'com.uber.autodispose:autodispose-android:1.x.x'
implementation 'com.uber.autodispose:autodispose-android-archcomponents:1.x.x'
implementation 'com.uber.autodispose:autodispose-rxlifecycle:1.x.x'
```


## Usage
### RFCOMM
The RFCOMM package wraps everything neccessary to communicate via RFCOMM in a user friendly way.

Connect to a device
```kotlin
val rxSpp = RxSpp(device)
rxSpp
    .connect()
    .subscribeBy(
        onError = { Log.e(TAG, "Received an error", it) },
        onComplete = {Log.d(TAG, "Connected to device")}
    )
```

Read data from connected device
```kotlin
rxSpp
    .read()
    .retryConditional(
        predicate = { it is IOException },
        maxRetry = 5,
        delayBeforeRetryInMillis = 100
    )
    .subscribeBy (
        onNext = {
            Log.d(TAG, "Received: $it")
        },
        onError = { Log.e(TAG, "Received an error", it) },
        onComplete = { Log.d(TAG, "Read completed")}
    )
```

Send data to a device
```kotlin
rxSpp
    .send("light on")
    .subscribeBy(
        onError = { e ->
            Log.e(TAG, "Received error!")
        },
        onComplete = {
            Log.d(TAG, "Succesfully sent $command to device")
        }
    )
```

### OBEX
Send file to device
```kotlin
val rxOBEX = RxObex(device)
rxOBEX
    .putFile("rubberduck.txt", "text/plain", "oh hi mark".toByteArray(), "example/directory")  // Name of file, mimetype, bytes of file, directory
    .subscribeBy(
        onComplete = {
            Log.d(TAG, "Succesfully sent a testfile to device")
        },
        onError = { e ->
            Log.e(TAG, "Received error!")
        }
    )
```

Delete file from device
```kotlin
rxOBEX
    .deleteFile("rubberduck.txt", "example/directory")
    .subscribeBy(
        onComplete = {
            Log.d(TAG, "Succesfully deleted rubberduck.txt from device")
        },
        onError = { e ->
            Log.e(TAG, "Received error!")
        }
    )
```

List files in given directory on device. The remote file structure gets automatically mapped to a FolderListing object.
```kotlin
rxOBEX
    .listFiles("example/directory") // List files in /example/directory
    .subscribeBy(
        onSuccess = {
            folderlisting ->
            Log.d(TAG, "Retrieved folderlisting")
            Log.d(TAG, folderlisting.toString())
        },
        onError = { e ->
            Log.e(TAG, "Received error!")
        }
    )
```

### UTILS
The utils package includes multiple bluetooth related helper methods, to make your life as a developer easier.

If you want to search for bluetooth devies, you can use the BluetoothDeviceFinder.
```kotlin
val finder = BluetoothDeviceFinder(applicationContext)
finder
    .search(checkPaired = true, indefinite = true)
    .subscribeOn(Schedulers.io())   // run in background
    .observeOn(Schedulers.io())
    .distinct() // don't emit devices multiple times
    .filter {   // you can filter for specific things, like the name, bondstate, address, etc..
        it.name?.contains("rubberduck", ignoreCase = true) ?: false // will only emit devices with the string "rubberduck" in them
    }
    .autoDisposable(scopeProvider)  // See the AutoDispose GitHub page from Uber
    .subscribeBy(
        onNext = { device ->
            Log.d(TAG, "Found device: ${device.name}")  // A device has been found
        },
        onError = {
            Log.e(TAG, "Error occoured")
            it.printStackTrace()
        },
        onComplete = { Log.d(TAG, "Completed search") }
    )
```

Check if bluetooth is enabled
```kotlin
     FunkerUtils.isBluetoothEnabled()
```

If you want to completely remove the bond to a bluetooth device
```kotlin
 var device: BluetoothDevice = ..
 device.removeBond()
 ...
```

If you want to retry (resend) a specific command, you can use retryConditional.
The following example retries reading 5 times, when an IOException occours and waits 100 millis before restarting.
```kotlin
rxSpp
    .read()
    .retryConditional(
      predicate = { it is IOException },
      maxRetry = 5,
      delayBeforeRetryInMillis = 100
    )
....
```
---

## License

[![License](http://img.shields.io/:license-mit-blue.svg?style=flat-square)](http://badges.mit-license.org)
