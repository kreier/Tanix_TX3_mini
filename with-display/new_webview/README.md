# Tanix TX3 — Installing and Testing a New Android System WebView

## Purpose

This document describes how to test a newer Android System WebView on a Tanix TX3 running Android 8.1.

The purpose is specifically to determine whether the TX3's original WebView 52 causes a JW Library document-scrolling problem.

The procedure is designed to avoid modifying the firmware's original WebView until it has been established that a newer WebView actually solves the problem.

---

# 1. Device characteristics

The TX3 currently reports:

```text
Android:      8.1
API level:    27
CPU ABI:      armeabi-v7a
ABI list:     armeabi-v7a,armeabi
Display:      1920 × 1080
```

Check the architecture with:

```sh
adb shell getprop ro.product.cpu.abi
adb shell getprop ro.product.cpu.abilist
```

Expected result:

```text
armeabi-v7a
armeabi-v7a,armeabi
```

Therefore an ARMv7 WebView APK must be used.

---

# 2. Original WebView

The firmware contains:

```text
Package:      com.android.webview
Version:      52.0.2743.100
Version code: 275610000
Location:     /system/app/webview/webview.apk
```

Verify with:

```sh
adb shell pm path com.android.webview
adb shell dumpsys package com.android.webview | grep -E 'versionCode|versionName'
```

Expected output:

```text
package:/system/app/webview/webview.apk

versionCode=275610000 minSdk=21 targetSdk=25
versionName=52.0.2743.100
```

---

# 3. Back up the original APK

Before experimenting with the WebView, save a copy of the original firmware APK.

From the development computer:

```sh
adb pull /system/app/webview/webview.apk webview-52-original.apk
```

This creates a local backup:

```text
webview-52-original.apk
```

Do not delete or overwrite the original system APK during the initial test.

---

# 4. Select a compatible replacement for testing

The test package used here is:

```text
Google Android System WebView
Version:      112.0.5615.100
Architecture: armeabi-v7a
Android:      7.0+
```

The APK is suitable for the TX3's:

```text
Android 8.1
armeabi-v7a
```

The Google WebView package name is:

```text
com.google.android.webview
```

This is different from the firmware's:

```text
com.android.webview
```

That distinction becomes important later.

---

# 5. Avoid Bash filename problems

APKMirror filenames can contain parentheses, for example:

```text
..._(armeabi-v7a)(nodpi)_...
```

Bash interprets parentheses unless the filename is quoted.

Either quote the filename:

```sh
adb push 'filename.apk' /sdcard/
```

or rename it first.

The simpler approach is:

```sh
mv 'long-apkmirror-filename.apk' webview112.apk
```

Then:

```sh
adb push webview112.apk /sdcard/
```

The APK can be verified on the device:

```sh
adb shell ls -lh /sdcard/webview112.apk
```

---

# 6. First installation attempt

The normal installation command is:

```sh
adb shell pm install -r /sdcard/webview112.apk
```

On this TX3 it produced:

```text
Failure [INSTALL_FAILED_VERSION_DOWNGRADE]
```

This happened even though WebView 112 has a higher human-readable version number than WebView 52.

This is because Android compares the APK's internal `versionCode`, not its `versionName`.

---

# 7. Installing with downgrade allowed

For this controlled experiment, downgrade protection can be bypassed with:

```sh
adb shell pm install -r -d /sdcard/webview112.apk
```

The command returned:

```text
Success
```

The `-d` option allows installation when Android believes the new APK has a lower version code.

This should be treated cautiously on a system component.

---

# 8. The installation did not replace the original WebView

After installation:

```sh
adb shell pm path com.android.webview
```

still returned:

```text
package:/system/app/webview/webview.apk
```

and:

```sh
adb shell dumpsys package com.android.webview
```

still reported:

```text
codePath=/system/app/webview
resourcePath=/system/app/webview
versionCode=275610000 minSdk=21 targetSdk=25
versionName=52.0.2743.100
```

Therefore the original system package was untouched.

---

# 9. Discovery of the second WebView package

The package list revealed:

```sh
adb shell pm list packages -f | grep webview
```

Output:

```text
package:/data/app/com.snc.test.webview2-2/base.apk=com.snc.test.webview2
package:/system/app/webview/webview.apk=com.android.webview
package:/data/app/com.google.android.webview-1/base.apk=com.google.android.webview
```

This demonstrates that the Google WebView was installed separately.

The resulting configuration is:

| Package                      | Version            | Location                                 | Role                             |
| ---------------------------- | ------------------ | ---------------------------------------- | -------------------------------- |
| `com.android.webview`        | 52.0.2743.100      | `/system/app/webview`                    | Original firmware WebView        |
| `com.google.android.webview` | 112.0.5615.100     | `/data/app/com.google.android.webview-1` | Newly installed test WebView     |
| `com.snc.test.webview2`      | firmware-dependent | `/data/app/...`                          | Existing WebView-related package |

The original provider has therefore **not been overwritten**.

---

# 10. Verify the newly installed provider

Run:

```sh
adb shell dumpsys package com.google.android.webview | \
    grep -E 'codePath|resourcePath|versionCode|versionName'
```

The expected important values are:

```text
codePath=/data/app/com.google.android.webview-1
resourcePath=/data/app/com.google.android.webview-1
versionName=112.0.5615.100
```

The exact version code should also be recorded.

---

# 11. Determine the active WebView provider

Android does not necessarily use every installed WebView package.

The currently selected provider can be queried with:

```sh
adb shell settings get secure webview_provider
```

If it returns:

```text
com.android.webview
```

then JW Library is still using the original firmware WebView.

In that case, simply installing `com.google.android.webview` has not changed the behavior of the system.

---

# 12. Inspect Android's WebView provider manager

The most important diagnostic command is:

```sh
adb shell dumpsys webviewupdate
```

This reports information maintained by Android's WebView update service.

It should help determine:

* which WebView packages Android recognizes;
* which package is currently selected;
* which packages are considered valid providers;
* whether `com.google.android.webview` can be selected;
* whether the firmware imposes additional provider restrictions.

This information should be collected before changing system files.

---

# 13. Potential provider switch

If Android recognizes:

```text
com.google.android.webview
```

as a valid provider, it may be possible to select it with:

```sh
adb shell settings put secure webview_provider com.google.android.webview
```

Then verify:

```sh
adb shell settings get secure webview_provider
```

If necessary, reboot:

```sh
adb reboot
```

After reboot, verify again:

```sh
adb shell dumpsys webviewupdate
```

and:

```sh
adb shell dumpsys package com.google.android.webview | \
    grep -E 'versionCode|versionName'
```

The important question is not merely whether the package exists, but whether Android reports it as the **current WebView provider**.

---

# 14. Test JW Library

Once the newer provider is confirmed as active, launch JW Library and reproduce the exact operation that originally failed.

The test should be kept identical:

1. Open the same document.
2. Navigate to the same location.
3. Perform the same action that should scroll to the end.
4. Observe whether the document reaches the expected position.

Record the result.

### Test A — original WebView

```text
Provider:
com.android.webview

Version:
52.0.2743.100

Result:
Scrolling-to-end problem reproduced.
```

### Test B — new WebView

```text
Provider:
com.google.android.webview

Version:
112.0.5615.100

Result:
[record result here]
```

The value of this experiment comes from changing only the WebView provider while keeping the rest of the environment as constant as possible.

---

# 15. Interpretation

If WebView 112 fixes the scrolling problem:

```text
WebView 52
    ↓
JW Library scrolling fails

WebView 112
    ↓
JW Library scrolling works
```

then the result provides strong evidence that the problem is caused by an incompatibility or limitation in the old WebView implementation.

It does not necessarily establish that WebView is the only possible cause, but it gives a strong reproducible correlation.

If WebView 112 does **not** fix the problem, the investigation should continue with JW Library, Android input handling, WebView configuration, and the document itself rather than immediately replacing the system WebView.

---

# 16. Do not replace `/system/app/webview/webview.apk` yet

At this stage the original file is still:

```text
/system/app/webview/webview.apk
```

It should remain untouched.

Manual replacement of this file is substantially more invasive because:

* the system partition may be read-only;
* the firmware may use package signatures or permissions associated with the original APK;
* Android's WebView provider service may have expectations about the provider;
* an incompatible WebView can prevent applications from displaying WebView content;
* a broken WebView can affect multiple applications;
* recovery may require restoring the original firmware APK.

The additional-provider approach is therefore preferable for the first test.

---

# 17. If the provider cannot be switched

If:

```sh
dumpsys webviewupdate
```

shows that `com.google.android.webview` is not considered a valid provider, then the next step is to determine why.

Possible reasons include:

1. The firmware only declares `com.android.webview` as a valid provider.
2. The Google WebView package does not satisfy the provider requirements of this particular Android build.
3. The provider requires specific signatures.
4. The firmware's WebView configuration was customized by the manufacturer.
5. Android 8.1's provider selection mechanism rejects the newly installed package.
6. The package was installed successfully but is not eligible to become the system WebView.

Only after identifying the reason should a system-level replacement be considered.

---

# 18. Useful diagnostic commands

The following commands provide a compact picture of the WebView configuration.

### Android version

```sh
adb shell getprop ro.build.version.release
adb shell getprop ro.build.version.sdk
```

### CPU architecture

```sh
adb shell getprop ro.product.cpu.abi
adb shell getprop ro.product.cpu.abilist
```

### Original WebView

```sh
adb shell pm path com.android.webview
adb shell dumpsys package com.android.webview | \
    grep -E 'codePath|resourcePath|versionCode|versionName'
```

### Google WebView

```sh
adb shell pm path com.google.android.webview
adb shell dumpsys package com.google.android.webview | \
    grep -E 'codePath|resourcePath|versionCode|versionName'
```

### Installed WebView packages

```sh
adb shell pm list packages -f | grep webview
```

### Current provider

```sh
adb shell settings get secure webview_provider
```

### WebView provider manager

```sh
adb shell dumpsys webviewupdate
```

---

# 19. Current known configuration

At the time of writing, the TX3 has reached the following state:

```text
Android:
8.1 / API 27

Architecture:
armeabi-v7a

Original WebView:
com.android.webview
52.0.2743.100
/system/app/webview/webview.apk

Additional WebView:
com.google.android.webview
112.0.5615.100
/data/app/com.google.android.webview-1/base.apk

Installation:
successful with pm install -r -d

Original system APK:
still present and unchanged
```

The remaining question is:

```text
Can Android 8.1 on this particular TX3
use com.google.android.webview as its active WebView provider?
```

That question should be answered using:

```sh
adb shell settings get secure webview_provider
adb shell dumpsys webviewupdate
```

before any system partition modifications are attempted.

---

# 20. Recovery

If the test provider causes problems, the original provider remains available because it was not overwritten.

The first recovery step should be to select the original provider again if Android permits it:

```sh
adb shell settings put secure webview_provider com.android.webview
adb reboot
```

If necessary, the experimental package can subsequently be removed:

```sh
adb shell pm uninstall com.google.android.webview
```

The exact uninstall behavior may depend on how Android registered the package. Because the original WebView is a system package, removing the experimental package should not remove:

```text
/system/app/webview/webview.apk
```

---

# 21. Final objective

The objective of this procedure is **not initially to upgrade the TX3 permanently**.

The objective is to perform a controlled experiment:

```text
                    ┌── WebView 52 ──→ JW Library
                    │                   ↓
                    │              scrolling fails
                    │
same TX3 + same JW ─┤
Library environment │
                    │
                    └── WebView 112 ─→ JW Library
                                        ↓
                                   test result
```

If the second configuration works, a more permanent WebView upgrade can then be investigated with much greater confidence.

Until that point, the safest state is the current one: **the original firmware WebView remains intact, while WebView 112 is installed separately for testing.**
