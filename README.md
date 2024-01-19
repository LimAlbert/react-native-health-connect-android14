
# react-native-health-connect with Android 14

This project is an attempt of making react-native-health-connect work with Android 14. The code is for sure not the optimal way to implement this since I'm no native developper but at least it seems to work !

There's 2 patches: 1 patch for react-native-health-connect and 1 patch for react-native, these patches are explained down below

## Android Manifest

It is not specified in react-native-health-connect doc but this intent filter is mandatory to make health connect work with Android 14. I'm not sure how we can implement it properly using react-native but in this project, I've made an Activity showing a web view.
This intent is called when the user press the privacy policy link in the permissions)

```
<intent-filter>
	<action android:name="android.intent.action.VIEW_PERMISSION_USAGE" />  
	<category android:name="android.intent.category.HEALTH_PERMISSIONS" />  
</intent-filter>  
```

## Patches breakdown
### react-native

- **react-native/ReactAndroid/src/main/java/com/facebook/react/ReactActivity.java**
```diff
@Override
public  void  onRequestPermissionsResult(int  requestCode, String[] permissions, int[] grantResults) {
+	super.onRequestPermissionsResult(requestCode, permissions, grantResults);
	mDelegate.onRequestPermissionsResult(requestCode, permissions, grantResults);
}
```

react-native is preventing **registerForActivityResult** from calling its callback function by catching all the PermissionsResults in onRequestPermissionsResult without calling super

### react-native-health-connect 

- **HealthConnectContractLauncher.kt**
```diff
+	package dev.matinzd.healthconnect  
+  
+	import androidx.activity.ComponentActivity  
+	import androidx.activity.result.ActivityResultLauncher  
+	import com.facebook.react.bridge.Promise  
+	import dev.matinzd.healthconnect.permissions.HCPermissionManager  
+  
+	object HealthConnectContractLauncher {  
+		private var contractLauncher: ActivityResultLauncher<Set<String>>? = null  
+		private var pendingPromise: Promise? = null  
+  
+		fun setProvider(activity: ComponentActivity, providerPackageName: String) {  
+			val contract = HCPermissionManager(providerPackageName).healthPermissionContract  
  
+			contractLauncher = activity.registerForActivityResult(contract) { granted ->
+	  			pendingPromise?.resolve(HCPermissionManager.mapPermissionResult(granted))  
+			}  
+		}  
+  
+		fun launch(permissions: Set<String>, promise: Promise) {  
+			pendingPromise = promise  
+			contractLauncher?.launch(permissions)  
+		}  
+	}
```
- **MainActivity.kt**
```diff
	override fun onCreate(savedInstanceState: Bundle?) {  
		super.onCreate(savedInstanceState)  
+		HealthConnectContractLauncher.setProvider(this, "com.google.android.apps.healthdata")  
}
```
registerForActivityResult has to be called before the Activity is started

- **HealthConnectManager.kt**
```diff
fun requestPermission(reactPermissions: ReadableArray, providerPackageName: String, promise: Promise) {  
	throwUnlessClientIsAvailable(promise) {
+		if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
+			HealthConnectContractLauncher.launch(HCPermissionManager.parsePermissions(reactPermissions), promise)  
		} else {  
			this.pendingPromise = promise  
			this.latestPermissions = HCPermissionManager.parsePermissions(reactPermissions)  
  
			val bundle = Bundle().apply {  
				putString("providerPackageName", providerPackageName)  
			}  
  
			val intent = HCPermissionManager(providerPackageName).healthPermissionContract.createIntent(applicationContext, latestPermissions!!)  
  
			applicationContext.currentActivity?.startActivityForResult(intent, HealthConnectManager.REQUEST_CODE, bundle)
		}  
	}  
}
```

Checking SDK version, and if it is at least Android 14, launch the ResultContract

- **permissions/HCPermissionManager.kt**
```diff
- private fun mapPermissionResult(grantedPermissions: Set<String>): WritableNativeArray {
+ fun mapPermissionResult(grantedPermissions: Set<String>): WritableNativeArray {  
	return WritableNativeArray().apply {  
		grantedPermissions.forEach {  
			val map = WritableNativeMap()  
			val (accessType, recordType) = extractPermissionResult(it)

			map.putString("recordType", snakeToCamel(recordType))  
			map.putString("accessType", accessType)  
			pushMap(map)  
		}  
	}  
}
```

Removed the private keyword since mapPermissionResult is called from **HealthConnectContractLauncher**
