# Creating the Shared Module

The goal of this tutorial is to demonstrate the reusability of Kotlin code between Android and iOS. Let's start
by manually creating a `SharedCode` sub-project in our Gradle project. The source code from the `SharedCode`
project will be shared between platforms.
We will create several new files in our project to implement this.

## Updating Gradle Scripts

The `SharedCode` sub-project should generate several artifacts for us:
 - A JAR file for the Android project, from the `androidMain` source set
 - The Apple framework 
   - for iOS device and App Store (`arm64` target)
   - for iOS simulator (`x86_64` target)

Let's update the Gradle scripts now to implement this and configure our IDE.
First, we add the new project to the `settings.gradle` file, simply by adding the following line to the end of the file:

```groovy
include ':SharedCode'
```

Next,
we need to create a `SharedCode/build.gradle.kts` file with the following content:
 
```kotlin
import org.jetbrains.kotlin.gradle.plugin.mpp.KotlinNativeTarget

plugins {
    kotlin("multiplatform")
}

kotlin {
    //select iOS target platform depending on the Xcode environment variables
    val iOSTarget: (String, KotlinNativeTarget.() -> Unit) -> KotlinNativeTarget =
        if (System.getenv("SDK_NAME")?.startsWith("iphoneos") == true)
            ::iosArm64
        else
            ::iosX64

    iOSTarget("ios") {
        binaries {
            framework {
                baseName = "SharedCode"
            }
        }
    }

    jvm("android")

    sourceSets["commonMain"].dependencies {
        implementation("org.jetbrains.kotlin:kotlin-stdlib-common")
    }

    sourceSets["androidMain"].dependencies {
        implementation("org.jetbrains.kotlin:kotlin-stdlib")
    }
}

val packForXcode by tasks.creating(Sync::class) {
    val targetDir = File(buildDir, "xcode-frameworks")

    /// selecting the right configuration for the iOS 
    /// framework depending on the environment
    /// variables set by Xcode build
    val mode = System.getenv("CONFIGURATION") ?: "DEBUG"
    val framework = kotlin.targets
                          .getByName<KotlinNativeTarget>("ios")
                          .binaries.getFramework(mode)
    inputs.property("mode", mode)
    dependsOn(framework.linkTask)

    from({ framework.outputDirectory })
    into(targetDir)

    /// generate a helpful ./gradlew wrapper with embedded Java path
    doLast {
        val gradlew = File(targetDir, "gradlew")
        gradlew.writeText("#!/bin/bash\n" 
            + "export 'JAVA_HOME=${System.getProperty("java.home")}'\n" 
            + "cd '${rootProject.rootDir}'\n" 
            + "./gradlew \$@\n")
        gradlew.setExecutable(true)
    }
}

tasks.getByName("build").dependsOn(packForXcode)
```

We need to refresh the Gradle project to apply these changes. Click on the `Sync Now` link or 
use the *Gradle* tool window and click the refresh action from the context menu on the root Gradle project.
The `packForXcode` Gradle task is used for Xcode project integration. We will discuss this later in the
tutorial.  

## Adding Kotlin Sources

The idea is to make every platform show similar text: `Kotlin Rocks on Android` and 
`Kotlin Rocks on iOS`, depending on the platform. We will reuse the way we generate the message. 
Let's create the file (and missing directories) `SharedCode/src/commonMain/kotlin/common.kt` with the following contents
under the project root directory

```kotlin
package com.jetbrains.handson.mpp.mobile

expect fun platformName(): String

fun createApplicationScreenMessage() : String {
  return "Kotlin Rocks on ${platformName()}"
}

```

That is the common part. The code to generate the final message. It `expect`s the platform part
to provide the platform-specific name from the `expect fun platformName(): String` function. We will use
the `createApplicationScreenMessage` from both Android and iOS applications.

Now we need to create the implementation file (and missing directories) for Android in the `SharedCode/src/androidMain/kotlin/actual.kt`:
```kotlin
package com.jetbrains.handson.mpp.mobile

actual fun platformName(): String {
  return "Android"
}

```

We create a similar implementation file (and missing directories) for the iOS target in the `SharedCode/src/iosMain/kotlin/actual.kt`:
```kotlin
package com.jetbrains.handson.mpp.mobile

import platform.UIKit.UIDevice

actual fun platformName(): String {
  return UIDevice.currentDevice.systemName() +
         " " +
         UIDevice.currentDevice.systemVersion
}
```

Here we can use the [UIDevice](https://developer.apple.com/documentation/uikit/uidevice?language=objc)
class from the Apple UIKit Framework, which is not available in Java, it is only usable in Swift and Objective-C.
The Kotlin/Native compiler comes with a set of pre-imported frameworks, so we can use
the UIKit Framework without having to do any additional steps.
The Objective-C and Swift Interop is covered in detail in the [documentation](/docs/reference/native/objc_interop.html)

## Multiplatform Gradle Project

The `SharedCode/build.gradle.kts` file uses the `kotlin-multiplatform` plugin to implement 
what we need. 
In the file, we define several targets `common`, `android`, and `iOS`. Each
target has its own platform. The `common` target contains the common Kotlin code 
which is included into every platform compilation. It is allowed to have `expect` declarations.
Other targets provide `actual` implementations for all the `expect`-actions from the `common` target. 
A more detailed explanation of multiplatform projects can be found in the
[Multiplatform Projects](/docs/reference/building-mpp-with-gradle.html) documentation.

Let's summarize what we have in the table:

| name | source folder | target | artifact |
|---|---|---|---|
| common | `SharedCode/commonMain/kotlin` |  - | Kotlin metadata |
| android | `SharedCode/androidMain/kotlin` | JVM 1.6 | `.jar` file or `.class` files |
| iOS | `SharedCode/iosMain` | iOS arm64 or x86_64| Apple framework |

Now it is again time to refresh the Gradle project in Android Studio. Click *Sync Now* on the yellow stripe 
or use the *Gradle* tool window and click the `Refresh` action in the context menu on the root Gradle project.
The `:SharedCode` project should now be recognized by the IDE.

We can use the `step-004` branch from the 
[github.com/kotlin-hands-on/mpp-ios-android](https://github.com/kotlin-hands-on/mpp-ios-android/tree/step-004)
repository as a solution for the tasks that we've done above. We can also download the
[archive](https://github.com/kotlin-hands-on/mpp-ios-android/archive/step-004.zip) from GitHub directly
or check out the repository and select the branch.

Let's use the `SharedCode` library from our Android and iOS applications.
