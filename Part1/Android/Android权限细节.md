# 1. 哪些App属于system app？
  **具有ApplicationInfo.FLAG_SYSTEM标记的，被视为System app；**
  ## 1.1 第一类System app： 特定的shared uid的app 属于system app
  例如：shared uid为android.uid.system，android.uid.phone，android.uid.log，android.uid.nfc，android.uid.bluetooth，android.uid.shell。这类app都被赋予了ApplicationInfo.FLAG_SYSTEM标志.

  在PackageManagerService的构造方法中，代码如下：
  ```
  mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
              ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
      mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
              ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
      mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
              ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
      mSettings.addSharedUserLPw("android.uid.nfc", NFC_UID,
              ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
      mSettings.addSharedUserLPw("android.uid.bluetooth", BLUETOOTH_UID,
              ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
      mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
              ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);


  ```
  ## 1.2 第二类System app： 特定目录中的app属于system app
  特定目录包括：/vendor/overlay，/system/framework，/system/priv-app，/system/app，/vendor/app，/oem/app。这些目录中的app，被视为system app。

  在PackageManagerService的构造方法中，代码如下：
  ```
  // /vendor/overlay folder
    File vendorOverlayDir = new File(VENDOR_OVERLAY_DIR);
    scanDirLI(vendorOverlayDir, PackageParser.PARSE_IS_SYSTEM
            | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags | SCAN_TRUSTED_OVERLAY, 0);

    // Find base frameworks (resource packages without code). /system/framework folder
    File frameworkDir = new File(Environment.getRootDirectory(), "framework");
    scanDirLI(frameworkDir, PackageParser.PARSE_IS_SYSTEM
            | PackageParser.PARSE_IS_SYSTEM_DIR
            | PackageParser.PARSE_IS_PRIVILEGED,
            scanFlags | SCAN_NO_DEX, 0);

    // Collected privileged system packages. /system/priv-app folder
    final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
    scanDirLI(privilegedAppDir, PackageParser.PARSE_IS_SYSTEM
            | PackageParser.PARSE_IS_SYSTEM_DIR
            | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

    // Collect ordinary system packages. /system/app folder
    final File systemAppDir = new File(Environment.getRootDirectory(), "app");
    scanDirLI(systemAppDir, PackageParser.PARSE_IS_SYSTEM
            | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

    // Collect all vendor packages.
    File vendorAppDir = new File("/vendor/app");
    scanDirLI(vendorAppDir, PackageParser.PARSE_IS_SYSTEM
            | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

    // Collect all OEM packages. /oem/app folder
    final File oemAppDir = new File(Environment.getOemDirectory(), "app");
    scanDirLI(oemAppDir, PackageParser.PARSE_IS_SYSTEM
            | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

  ```
  scanDirLI参数中的PackageParser.PARSE_IS_SYSTEM最终会被转换为Package的ApplicationInfo.FLAG_SYSTEM属性。这个过程相关的代码流程（简略）：
  ```
  scanDirLI(PackageParser.PARSE_IS_SYSTEM)

-> scanPackageLI(file, parseFlags | PackageParser.PARSE_MUST_BE_APK, scanFlags, currentTime, null);

-> scanPackageLI(pkg, parseFlags, scanFlags | SCAN_UPDATE_SIGNATURE, currentTime, user);

-> final PackageParser.Package res = scanPackageDirtyLI(pkg, parseFlags, scanFlags, currentTime, user);

-> scanPackageDirtyLI()中
    if ((parseFlags & PackageParser.PARSE_IS_SYSTEM) != 0) {
       pkg.applicationInfo.flags |= ApplicationInfo.FLAG_SYSTEM;
    }

  ```

# 2. 什么是privileged app（特权app）？
注：privileged app，在本文中称之为 特权app，主要原因是此类特权app可以使用protectionLevel为signatureOrSystem或者protectionLevel为signature|privileged的权限。

从PackageManagerService的isPrivilegedApp()可以看出特权app是具有ApplicationInfo.PRIVATE_FLAG_PRIVILEGED标志的一类app。
```
private static boolean isPrivilegedApp(PackageParser.Package pkg) {
    return (pkg.applicationInfo.privateFlags & ApplicationInfo.PRIVATE_FLAG_PRIVILEGED) != 0;
}
```
特权app首先必须是System app。也就是说 System app分为普通的system app和特权的system app。
```
System app = 普通的system app + 特权app

```
直观的（但不准确严谨）说，普通的system app就是/system/app目录中的app，特权的system app就是/system/priv-app目录中的app。
BTW： priv-app 是privileged app的简写。

**区分普通system app和特权app的目的是澄清这个概念："signatureOrSystem"权限中的System不是为普通system app提供的，而是只有特权app能够使用。**

进一步说，android:protectionLevel="signatureOrSystem"中的System和android:protectionLevel="signature|privileged"中的privileged的含义是一样的。
可以从PermissionInfo.java中的fixProtectionLevel()看出来：
```
// PermissionInfo.java
public static int fixProtectionLevel(int level) {
    if (level == PROTECTION_SIGNATURE_OR_SYSTEM) {
        // "signatureOrSystem"权限转换成了"signature|privileged"权限
        level = PROTECTION_SIGNATURE | PROTECTION_FLAG_PRIVILEGED;
    }
    return level;
}

```
所以，当我们说起system app时，通常指的是前面提到的特定uid和特定目录中的app，包含了普通的system app和特权app。
当我们说起有访问System权限或者privileged权限的app时，通常指特权app。

# 3. 哪些app属于privileged app（特权app）？
**特权app首先是System app，然后要具有ApplicationInfo.PRIVATE_FLAG_PRIVILEGED标志。**
## 3.1 第一类privileged app： 特定uid的app
shared uid为"android.uid.system"，"android.uid.phone"，"android.uid.log"，"android.uid.nfc"，"android.uid.bluetooth"，"android.uid.shell"的app被赋予了privileged的权限。这些app同时也是system app。

代码如下：
```
// PackageManagerService的构造方法
mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
mSettings.addSharedUserLPw("android.uid.nfc", NFC_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
mSettings.addSharedUserLPw("android.uid.bluetooth", BLUETOOTH_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
        ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
```
## 3.2 第二类privileged app：/system/framework和/system/priv-app目录下的app
/system/framework和/system/priv-app目录下的app被赋予了privileged的权限。
其中/system/framework目录中的apk，只是包含资源，不包含代码（dex）。

代码如下：
```
// PackageManagerService的构造方法
// /system/framework
File frameworkDir = new File(Environment.getRootDirectory(), "framework");
scanDirLI(frameworkDir, PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR
        | PackageParser.PARSE_IS_PRIVILEGED,
        scanFlags | SCAN_NO_DEX, 0);

// /system/priv-app
final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
scanDirLI(privilegedAppDir, PackageParser.PARSE_IS_SYSTEM
        | PackageParser.PARSE_IS_SYSTEM_DIR
        | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

```
PackageParser.PARSE_IS_PRIVILEGED标志最终会转换为Package的ApplicationInfo.PRIVATE_FLAG_PRIVILEGED标志。大概的代码流程，如下：
```
scanDirLI(PackageParser.PARSE_IS_PRIVILEGED)

-> scanPackageLI(file, parseFlags | PackageParser.PARSE_MUST_BE_APK, scanFlags, currentTime, null);

-> PackageParser.Package scannedPkg = scanPackageLI(pkg, parseFlags, scanFlags | SCAN_UPDATE_SIGNATURE, currentTime, user);

-> final PackageParser.Package res = scanPackageDirtyLI(pkg, parseFlags, scanFlags, currentTime, user);

-> scanPackageDirtyLI()中
    if ((parseFlags & PackageParser.PARSE_IS_PRIVILEGED) != 0) {
        pkg.applicationInfo.privateFlags |= ApplicationInfo.PRIVATE_FLAG_PRIVILEGED;
    }

```

# 4. Android app中的权限是必须先声明后使用吗？
在本文中，**声明权限**是指在AndroidManifest.xml中使用了<permission>，**使用权限**是指在AndroidManifest.xml中使用了<uses-permission>。**获得权限（或赋予权限）**是指真正的可以通过系统的权限检查，调用到权限保护的方法。

场景：App A中声明了权限PermissionA，App B中使用了权限PermissionA。
那么App A必须比App B先安装，App B才能获取到权限吗？

答案是不一定，要看具体情况而定。这个具体情况就是权限的保护级别。

    情况一：PermissionA的保护级别是normal或者dangerous
    App B先安装，App A后安装，此时App B没有获取到PermissionA的权限。
    即，此种情况下，权限必须先声明再使用。即使App A和App B是相同的签名。

    情况二：PermissionA的保护级别是signature或者signatureOrSystem
    App B先安装，App A后安装，如果App A和App B是相同的签名，那么App B可以获取到PermissionA的权限。如果App A和App B的签名不同，则App B获取不到PermissionA权限。
    即，对于相同签名的app来说，不论安装先后，只要是声明了权限，请求该权限的app就会获得该权限。
    这也说明了对于具有相同签名的系统app来说，安装过程不会考虑权限依赖的情况。安装系统app时，按照某个顺序（例如名字排序，目录位置排序等）安装即可，等所有app安装完了，所有使用权限的app都会获得权限。

在这里提供一种方法，可以方便的验证上面的说法：

## 4.1 验证某个app是否获得了某个权限的方法
可以用下面2个命令来验证：
```
adb shell dumpsys package permission <权限名>

adb shell dumpsys package <包名>
```

其中，adb shell dumpsys package permission <权限名>**可以查看某个权限是谁声明的，谁使用了**。
adb shell dumpsys package <包名>**可以看到某个app是否获得了某个权限**。

例如，App A中声明了权限com.package.a.PermissionA，App B中使用了权限com.package.a.PermissionA。
App A的包名为com.package.a，App B的包名为com.package.b。
### 4.1.1 命令一： adb shell dumpsys package permission <权限名>
查看某个权限是谁声明的，谁使用了。

通过下面的命令，查看com.package.a.PermissionA权限：
```
adb shell dumpsys package permission com.package.a.PermissionA

```
输出结果为：（注：这里只显示关键信息，省略了其他很多信息）
```
Permission [com.package.a.PermissionA] (a2930ef):
    sourcePackage=com.package.a            // 权限的声明者
    uid=10226 gids=null type=0 prot=normal //保护级别为normal

Packages:
  Package [com.package.b] (350d95):      // 权限的使用者
    requested permissions:
      com.package.a.PermissionA
    install permissions:
      com.package.a.PermissionA, granted=true, flags=0x0

```
其中granted=true表明App B 获取到了com.package.a.PermissionA权限。
如果App B没有获取到com.package.a.PermissionA权限，则输出结果中只有权限的声明信息，如下：
```
Permission [com.package.a.PermissionA] (a2930ef):
    sourcePackage=com.package.a
    uid=10226 gids=null type=0 prot=normal 保护级别为normal

```
### 4.1.2 命令二： adb shell dumpsys package <包名>
查看某个app是否获得了某个权限。

查看App B（包名为com.package.b）是否获得了权限com.package.a.PermissionA：
```
adb shell dumpsys package com.package.b

```
输出结果为：（注：省略了很多其他信息）
```
Packages:
  Package [com.package.b] (6611d16):
    requested permissions:         // 申请了哪些权限
      com.package.a.PermissionA
    install permissions:           // 获得了哪些权限
      com.package.a.PermissionA, granted=true, flags=0x0
```
上面的输出结果表明，App B获取到了com.package.a.PermissionA权限。
如果App B没有获取到 com.package.a.PermissionA 权限，那么在 install permissions 中不会出现com.package.a.PermissionA, granted=true, flags=0x0。

# 5. 关于signature权限和signatureOrSystem权限的获取（或拒绝）
这里要说的是这种场景：App A中声明了权限com.package.a.PermissionA，App B中使用了权限com.package.a.PermissionA。其中com.package.a.PermissionA的保护级别为signature或者signatureOrSystem。App A先安装，App B后安装。App B和App A的签名可能一样，也可能不一样。App B的签名也可能是系统签名，这是对于厂商的app来说的。

安装App B时，PackageManagerService会对App B能否获得com.package.a.PermissionA权限做检查。
大概的代码流程如下：
![d](https://raw.githubusercontent.com/hdjjun/LearningNotes/master/Part1/Android/install_apk_flow.png)
安装App B的时候：
- installPackageLI()
```
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    final File tmpPackageFile = new File(args.getCodePath());// 被安装的apk的路径
    final PackageParser.Package pkg;
    try {
        pkg = pp.parsePackage(tmpPackageFile, parseFlags);
    }
    ...
    if (replace) {
        ...
    } else {
        // run here
        installNewPackageLI(pkg, parseFlags, scanFlags | SCAN_DELETE_DATA_ON_FAILURES,
                args.user, installerPackageName, volumeUuid, res);
    }
}
```
- installNewPackageLI()
```
private void installNewPackageLI(PackageParser.Package pkg, int parseFlags, int scanFlags, UserHandle user, String installerPackageName, String volumeUuid, PackageInstalledInfo res) {
        PackageParser.Package newPackage = scanPackageLI(pkg, parseFlags, scanFlags, System.currentTimeMillis(), user);

        updateSettingsLI(newPackage, installerPackageName, volumeUuid, null, null, res, user);// run here
}

```
- updateSettingsLI()
```
private void updateSettingsLI(PackageParser.Package newPackage, String installerPackageName,
        String volumeUuid, int[] allUsers, boolean[] perUserInstalled, PackageInstalledInfo res,
        UserHandle user) {

        updatePermissionsLPw(newPackage.packageName, newPackage,
                UPDATE_PERMISSIONS_REPLACE_PKG | (newPackage.permissions.size() > 0
                        ? UPDATE_PERMISSIONS_ALL : 0));
```
- updatePermissionsLPw()
```
private void updatePermissionsLPw(String changingPkg,
        PackageParser.Package pkgInfo, int flags) {
    if (pkgInfo != null) {
        grantPermissionsLPw(pkgInfo, (flags&UPDATE_PERMISSIONS_REPLACE_PKG) != 0, changingPkg);
    }
}

```
- grantPermissionsLPw()
```
private void grantPermissionsLPw(PackageParser.Package pkg, boolean replace,
        String packageOfInterest) {

    // 遍历安装包pkg中所有请求的权限
    final int N = pkg.requestedPermissions.size();
    for (int i=0; i<N; i++) {
        final String name = pkg.requestedPermissions.get(i); // 权限的名字
        final BasePermission bp = mSettings.mPermissions.get(name);// 权限的信息

        final String perm = bp.name;
        boolean allowedSig = false;
        int grant = GRANT_DENIED;

        final int level = bp.protectionLevel & PermissionInfo.PROTECTION_MASK_BASE; // 权限的保护级别, PROTECTION_MASK_BASE 为0xf，signature级别时，bp.protectionLevel = 2；signatureOrSystem级别时，bp.protectionLevel = 0x12，所以对这两种级别的权限，level都是2，即，PermissionInfo.PROTECTION_SIGNATURE
        switch (level) {
            case PermissionInfo.PROTECTION_NORMAL: { // value is 0
                // For all apps normal permissions are install time ones.
                grant = GRANT_INSTALL;
            } break;

            case PermissionInfo.PROTECTION_DANGEROUS: {// value is 1
                // 略
            } break;

            // 被安装app使用了其他app声明的signature或者signatureOrSystem权限，上面提到的场景会执行到这里
            case PermissionInfo.PROTECTION_SIGNATURE: {// value is 2
                // For all apps signature permissions are install time ones.
                allowedSig = grantSignaturePermission(perm, pkg, bp, origPermissions);
                if (allowedSig) {
                    grant = GRANT_INSTALL;
                }
            } break;
        }

```
- grantSignaturePermission()
注：pre23的权限，请参考链接： [关于pre23权限](https://blog.csdn.net/u013553529/article/details/53167072#pre23)
```
private boolean grantSignaturePermission(String perm, PackageParser.Package pkg,
        BasePermission bp, PermissionsState origPermissions) {
    boolean allowed;
    // 这里检查被安装的app的签名（pkg.mSignatures）与声明权限的app的签名（bp.packageSetting.signatures.mSignatures）是否一致，
    // 如果不一致，则再检查被安装app的签名是否与系统签名（mPlatformPackage.mSignatures）一致。
    // 如果其中一个是一致的，则赋予被安装app signature权限。
    // 注意：只要被安装的app的签名是系统签名，则其可以访问任意第三方声明的signature权限。
    allowed = (compareSignatures(
            bp.packageSetting.signatures.mSignatures, pkg.mSignatures)
                    == PackageManager.SIGNATURE_MATCH)
            || (compareSignatures(mPlatformPackage.mSignatures, pkg.mSignatures)
                    == PackageManager.SIGNATURE_MATCH);

    // 如果被安装app的签名既不是声明权限的app的签名，也不是系统签名，则继续检查其他标志位。
    // 首先检查声明的权限是否是privileged权限（也就是signatureOrSystem中的System权限），如果权限是privileged的，那么对于系统应用（满足isSystemApp(pkg)）并且是privileged应用（满足isPrivilegedApp(pkg)），就会赋予
    if (!allowed && (bp.protectionLevel
            & PermissionInfo.PROTECTION_FLAG_PRIVILEGED) != 0) {
        if (isSystemApp(pkg)) {// 如果被安装的app是system应用（见前面对system app的说明）
            if (pkg.isUpdatedSystemApp()) { // 如果是更新系统app
                // 略
            } else {// 不是更新系统app，如果被安装的app是privileged的，则赋予其权限。也就是说，对于/system/priv-app目录中的app，会获取到权限。这种情况发生在privileged app第一次被安装时，[或者系统被root后，强行push apk到system/priv-app目录？（有待验证）]。
                allowed = isPrivilegedApp(pkg);
            }
        }
    }

    if (!allowed) {
        if (!allowed && (bp.protectionLevel
                & PermissionInfo.PROTECTION_FLAG_PRE23) != 0
                && pkg.applicationInfo.targetSdkVersion < Build.VERSION_CODES.M) {// 如果之前没有获取到权限，再判断是否是pre23的权限。如果是pre23的权限，且被安装的app的targetVersion是22及以下，则赋予其权限。
            allowed = true;
        }
        if (!allowed && (bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_INSTALLER) != 0
                && pkg.packageName.equals(mRequiredInstallerPackage)) {// 只有指定的system installer才能使用该权限。
            allowed = true;
        }
        if (!allowed && (bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_VERIFIER) != 0
                && pkg.packageName.equals(mRequiredVerifierPackage)) {// 只有指定的system verifier才能使用该权限。
            allowed = true;
        }
        if (!allowed && (bp.protectionLevel
                & PermissionInfo.PROTECTION_FLAG_PREINSTALLED) != 0
                && isSystemApp(pkg)) { // 对于preinstalled级别的权限，只有系统app可以使用该权限。
            // Any pre-installed system app is allowed to get this permission.
            allowed = true;
        }
        if (!allowed && (bp.protectionLevel
                & PermissionInfo.PROTECTION_FLAG_DEVELOPMENT) != 0) {
            // For development permissions, a development permission
            // is granted only if it was already granted.
            allowed = origPermissions.hasInstallPermission(perm);
        }
    }
    return allowed;
}
```
# 6. 哪些app可以使用某app提供的signature权限或signatureOrSystem权限？
根据上面的代码，我们可以得出以下结论：
- 如果App A声明了signatureOrSystem权限，即android:protectionLevel="signature|privileged"或者android:protectionLevel="signatureOrSystem"，则可以使用该权限的app包括：
    - （1）与App A有相同签名的app
    - （2）与系统签名相同的app，即与厂商签名（厂商ROM中的系统app签名）相同的app
    - （3）在/system/priv-app目录中的app，不管是否满足（1）和（2）的条件。即任意app只要放到了/system/priv-app就可以使用App A的signatureOrSystem级别的权限。
- 如果App A声明了signature权限，即android:protectionLevel="signature"，则可以使用该权限的app包括：
    - （1）与App A有相同签名的app
    - （2）与系统签名相同的app，即与厂商签名（厂商ROM中的系统app签名）相同的app

**注意：与系统签名相同，即与厂商签名相同，这是指厂商推出的app，这些app有很大的访问权限。**

# 7. 关于pre23的权限
在framework/base/core/res/AndroidManifest.xml中，有2个权限从normal级别提升到signature级别，它们是

```
android.permission.WRITE_SETTINGS
android.permission.SYSTEM_ALERT_WINDOW

```

可以预见，当市场上绝大多数app的targetSdkVersion为23或23以上（即Android 6.0或以上）时，pre23字样将会被去掉，到那时这两个权限将会得到更好地保护。

pre23的含义是，如果某个app的targetSdkVersion是22或者22以下（即Android 5.x及以下），那么该app就可以获取到这2个权限。
```
<permission android:name="android.permission.WRITE_SETTINGS"
    android:label="@string/permlab_writeSettings"
    android:description="@string/permdesc_writeSettings"
    android:protectionLevel="signature|preinstalled|appop|pre23" />

<permission android:name="android.permission.SYSTEM_ALERT_WINDOW"
    android:label="@string/permlab_systemAlertWindow"
    android:description="@string/permdesc_systemAlertWindow"
    android:protectionLevel="signature|preinstalled|appop|pre23|development" />

```

# 8. 关于install权限和runtime权限
- **install权限**：安装时权限，是指在安装app的时候，赋予app的权限。normal和signature级别的权限都是安装时权限。不会给用户提示界面，系统自动决定权限的赋予或拒绝。
- **runtime权限**：运行时权限，是指在app运行过程中，赋予app的权限。这个过程中，会显示明显的权限授予界面，让用户决定是否授予权限。如果app的targetSdkVersion是22（Lollipop MR1）及以下，dangerous权限是安装时权限，否则dangerous权限是运行时权限。

# 9. 官方文档关于权限保护级别的说明
## 9.1 <permission>的权限级别 android:protectionLevel
参考: https://developer.android.com/guide/topics/manifest/permission-element.html

| Value     | Meaning     |
| :------------- | :------------- |
| “normal”       | normal级别是默认值。低风险的权限采用此级别。在app安装的时候，系统自动赋予此app请求的所有normal权限，而不会征求用户的同意（但是，在安装app之前，用户总是有权选择检查这些权限）。      |
|“dangerous”    | 较高风险的权限，此级别的权限意味着，请求权限的app将要访问用户的隐私数据或者控制设备，这可能给用户带来负面影响。因为dangerous权限会引入潜在的风险，所以系统不会自动赋予此类权限给app。例如，在安装app的时候，会将dangerous权限展示给用户，并请求用户确认。 |
| “signature”   | 只有请求权限的app与声明权限的app的签名是一样的时候，系统才会赋予signature权限。如果签名一致，系统会自动赋予权限，而不会通知用户或者征求用户的同意。 |
| “signatureOrSystem”  | 系统赋予此类权限有2种情况：（1）请求权限的app与声明权限的app的签名一致；（2）请求权限的app在Android 系统镜像（system image）中。 signatureOrSystem权限主要用在这个场景：多个软件供应商的apps预装到了系统目录（system/priv-app）中，而且这些apps之间会共享一些功能。除此之外，尽量不要使用此类权限级别。|

## 9.2 PermissionInfo中关于 protectionLevel
参考： https://developer.android.com/reference/android/content/pm/PermissionInfo.html
PermissionInfo.java中的权限保护级别主要有PROTECTION_NORMAL, PROTECTION_DANGEROUS, 或者 PROTECTION_SIGNATURE。
```
public static final int PROTECTION_NORMAL = 0;
public static final int PROTECTION_DANGEROUS = 1;
public static final int PROTECTION_SIGNATURE = 2;
```
**注意：PROTECTION_SIGNATURE_OR_SYSTEM 和PROTECTION_FLAG_SYSTEM已经被弃用。**
```
public static final int PROTECTION_SIGNATURE_OR_SYSTEM = 3;
public static final int PROTECTION_FLAG_SYSTEM = 0x10;
```
## 9.3 R.attr.html中 protectionLevel
参考： https://developer.android.com/reference/android/R.attr.html#protectionLevel

这里涵盖了protectionLevel所有可能的取值。

protectionLevel描绘了权限所蕴含的潜在风险，并且指明了系统在赋予某个app请求的权限时所要遵守的规程（procedure）。Android标准的权限有着事先定义好的、固定不变的protectionLevel（事实上，protectionLevel也会变，见pre23相关的说明)。

自定义的权限的protectionLevel属性可以取下面列表中的‘Constant’值，如果有多个’Constant’值，则需要用‘|’分隔开。如果没有定义protectionLevel，则默认权限为normal。

权限级别可以分为两类：**基础权限级别**和**附加权限级别**。
- 基础权限级别

    有3种：normal、dangerous和signature。（注：signatureOrSystem处于弃用状态，不建议使用）
- 附加权限级别

    0x10及其之后的权限级别都属此类，例如，privileged、appop等。它们必须附加在基础权限上。目前貌似只能附加在signature权限上。

| Constant     | Value     | Description     |
| :------------- | :------------- | :------------- |
|    normal    |  0      |   normal级别是默认值。低风险的权限采用此级别。在app安装的时候，系统自动赋予此app请求的所有normal权限，而不会征求用户的同意（但是，在安装app之前，用户总是有权选择检查这些权限）。    |
|   dangerous     |   1     |   较高风险的权限，此级别的权限意味着，请求权限的app将要访问用户的隐私数据或者控制设备，这可能给用户带来负面影响。因为dangerous权限会引入潜在的风险，所以系统不会自动赋予此类权限给app。例如，在安装app的时候，会将dangerous权限展示给用户，并请求用户确认。    |
|  signature      |   2     |   只有请求权限的app与声明权限的app的签名是一样的时候，系统才会赋予signature权限。如果签名一致，系统会自动赋予权限，而不会通知用户或者征求用户的同意。    |
|  signatureOrSystem      |   3     |   系统赋予此类权限有2种情况：（1）请求权限的app与声明权限的app的签名一致；（2）请求权限的app在Android 系统镜像（system image）中。 signatureOrSystem权限主要用在这个场景：多个软件供应商的apps预装到了系统目录（system/priv-app）中，而且这些apps之间会共享一些功能。除此之外，尽量不要使用此类权限级别。    |
|  privileged      |     	0x10    |   只能与signature同时使用。signature I privileged与signatureOrSystem意义相同。Android中system/priv-app目录和system/framework目录中的app可以访问privileged权限    |
|  system      |  0x10      |  意义与privileged相同。注意：由于system的概念会造成混乱，不建议使用。system权限并不是system/app目录中的app能访问的权限。     |
|   development     |   0x20     |   development applications可以访问此权限。Android标准权限中，development是与signature和privileged一起用的。例如如android:protectionLevel="signature I privileged I development" |
|  appop      |   0x40     |  由AppOpsManager来检查app是否访问此类权限     |
|  pre23      |   0x80     |  此类权限自动被赋予那些targetSdkVersion在22（Android 5.x）或22以下的app     |
| installer       |   0x100     |  此类权限自动被赋予负责安装apk的系统app     |
|   verifier     |    	0x200     |    此类权限自动被赋予负责验证apk的系统app   |
|  preinstalled      |   0x400     |   此类权限可以自动被赋予任何预安装在system image中的app，不只是privileged app    |
|  setup      |    	0x800     |  此类权限自动被赋予‘安装向导’app     |

# 10. 权限在四大组件中的使用以及URI权限
参考： https://developer.android.com/guide/topics/security/permissions.html
# 10.1 在AndroidManifest.xml使用权限保护组件
通过在AndroidManifest.xml中采用高保护级别的权限，来保护系统或者app的组件不会被随意访问。实现方式是，为组件添加android:permission属性。

**Activity** 权限(用于 <activity> 标记) 限定了 哪些app可以启动Activity。权限检查是在 Context.startActivity() 和Activity.startActivityForResult()的时候进行的。如果调用者没有此权限，则抛出SecurityException异常。

**Service** 权限(用于 <service> 标记) 限定了哪些app可以启动（start）或者绑定（bind）服务。权限检查是在Context.startService(), Context.stopService() 和Context.bindService()的时候进行的。如果调用者没有此权限，则抛出SecurityException异常。

**BroadcastReceiver** 权限(用于<receiver> 标记) 限定了哪些app可以发送广播给receiver。权限检查是在Context.sendBroadcast()返回之后，在系统尝试将广播递交给receiver的时候。即使调用者没有权限，也不会抛出异常，只是不递送intent（广播）给receiver。

同理，在调用Context.registerReceiver()的时候，也可以加上权限，来控制谁可以发广播给这个动态注册的receiver。也可以在调用Context.sendBroadcast()的时候加上权限，来限定哪些BroadcastReceiver 允许接收这个广播。

注意，可以同时给receiver（动态或静态receiver）加上权限，并且给Context.sendBroadcast()加上权限，在这种情况下，需要双方都验证通过，才能完成信息传递。
注：动态receiver是指Context.registerReceiver()声明的receiver。
静态receiver是指AndroidManifest.xml中<receiver> 声明的receiver。

**ContentProvider** 权限(用于 <provider> 标记) 限定了哪些app可以访问ContentProvider中的数据。与其他组件不同，ContentProvider有2种权限android:readPermission和android:writePermission，分别控制读和写。
注意：如果provider同时受读权限和写权限保护，调用者只拥有写权限，是不能够执行读操作的。
在获取provider的时候，会进行权限检查。如果没有读权限和写权限，则抛出SecurityException异常。

在操作provider的时候，也会进行权限检查。在这里操作provider是指增删查改操作，读写文件。

查询的时候（ContentResolver.query() 被调用），需要读权限；添加、修改、删除的时候（ContentResolver.insert(), ContentResolver.update(), ContentResolver.delete()被调用 )，需要写权限。如果没有相应的权限，则抛出SecurityException异常。

读文件（openFile()时，mode值含有r），需要读权限；写文件（openFile()时，mode值含有w），需要写权限。

具体都有哪些情况检查读写权限，可以参考ContentProvider.java。
## 10.2 URI Permissions
URI 权限是对上面的权限的补充，是用这个场景中的：App A的content provider不能够开放读写权限给App B（或者其他app），但是App A又需要传递一些数据给App B。在这个场景中，就可以使用URI权限。

典型的例子是，查看邮件app中的附件（例如图片）。访问邮件是受读写权限保护，因为是用户的敏感数据。通过image viewer这个app查看邮件附件中的图片，但是不能够让image viewer拥有读取邮件的权限，这时，image viewer无法打开附件中的图片。

解决办法，就是使用URI权限。这也算是，给没有读写权限的app关上一扇门，然后给那些app打开一扇窗。

当**启动activity** 或者 **返回结果给activity** 的时候，调用者可以设置Intent.FLAG_GRANT_READ_URI_PERMISSION 和/或 Intent.FLAG_GRANT_WRITE_URI_PERMISSION标志。.这时接收数据的activity（例如上面场景中的App B或者image viewer，为便于描述，在这里称之为App B吧）就有权限访问Intent中的URI指定的数据了，即使App B没有读写content provider的权限。

注意：App B保持访问特定URI的权限，不是永久的，只持续到App B应用退出之前或者 URI权限回收之前。

Content provider通过android:grantUriPermissions或者<grant-uri-permissions>来支持URI权限。

也可以在启动Activity的时候，通过Context.grantUriPermission()赋予URI权限。Context.revokeUriPermission()可收回URI权限，Context.checkUriPermission()用来检查URI权限。

# 11. 如果targetSdkVersion版本低，那些在targetSdkVersion之后新加的权限就会被自动赋予
参考： https://developer.android.com/guide/topics/security/permissions.html#auto-adjustments

随着时间的推移，Android中会对一些特定的API加上权限保护（应该是用新的权限对已有的API进行保护）。对于已经存在的app，以前调用这些API是不需要权限的，但是在新版本的Android系统中需要检查权限。这个问题就是旧app运行在新系统上的问题。

为了让旧app能够在新版本的Android系统中正常的运行，Android会自动给这些app的manifest中加上相应的权限。何时给某个app加上缺失的权限，取决于targetSdkVersion。**如果targetSdkVersion的值比添加新权限时的Api level低，那么Android将为其加上权限，即使该app并不需要此权限**。

例如，WRITE_EXTERNAL_STORAGE权限是在API level 4的时候添加的，如果某个app的targetSdkVersion为3或者更小，那么当这个app运行在新版本的Android中时，自动会获得WRITE_EXTERNAL_STORAGE权限。在这里，新版本的Android是指Api level为4或4以上。

在这种情况下，查看这个app请求的所有权限的时候，会看到Android自动为这个app添加的权限。（注：可以用命令 adb shell dumpsys package <包名>查看某个app所请求的全部权限。）

为了避免不必要的权限添加到你的app中，要经常更新targetSdkVersion，越大越好。查看Android每一个版本中都新加了哪些权限，请参考：https://developer.android.com/reference/android/os/Build.VERSION_CODES.html

# 12. Android版本演进过程中新添加的权限
注：一般的权限名的前缀为android.permission.，也有部分权限不是，例如UNINSTALL_SHORTCUT，完整权限为com.android.launcher.permission.UNINSTALL_SHORTCUT。

| Api level    | Android 版本     | 新加的权限     |
| :------------- | :------------- | :------------- |
|   4     |   Donut, 1.6 (2009.9)     |   WRITE_EXTERNAL_STORAGE -- READ_PHONE_STATE  CHANGE_WIFI_MULTICAST_STATE -- GLOBAL_SEARCH -- INSTALL_LOCATION_PROVIDER     |
|   5     |  Eclair, 2.0 (2009.11)      |    ACCOUNT_MANAGER    |
|   8     |  Froyo, 2.2 (2010.6)      |   BIND_DEVICE_ADMIN -- BIND_WALLPAPER  -- KILL_BACKGROUND_PROCESSES -- SET_TIME     |
|   9     |  Gingerbread, 2.3 (2010.11)      |   NFC -- SET_ALARM -- USE_SIP     |
|   11     | Honeycomb, 3.0 (2011.2)       |    BIND_REMOTEVIEWS    |
|   14     |  Ice Cream Sandwich, 4.0 (2011.10)      |  ADD_VOICEMAIL -- BIND_TEXT_SERVICE -- BIND_VPN_SERVICE      |
|   16     |  JellyBean, 4.1 (2012.6)      |  READ_CALL_LOG -- WRITE_CALL_LOG -- BIND_ACCESSIBILITY_SERVICE -- READ_EXTERNAL_STORAGE **注意：请求 WRITE_EXTERNAL_STORAGE权限的时候，将会自动获取到READ_EXTERNAL_STORAGE权限。 **     |
|   18     |   JellyBean MR2, 4.3 (2013.7)     |   BIND_NOTIFICATION_LISTENER_SERVICE -- LOCATION_HARDWARE -- SEND_RESPOND_VIA_MESSAGE     |
|   19     |   Kitkat, 4.4 (2013.10)     |  BIND_NFC_SERVICE -- BIND_PRINT_SERVICE -- BLUETOOTH_PRIVILEGED -- CAPTURE_AUDIO_OUTPUT -- CAPTURE_SECURE_VIDEO_OUTPUT -- CAPTURE_VIDEO_OUTPUT -- INSTALL_SHORTCUT -- MANAGE_DOCUMENTS -- MEDIA_CONTENT_CONTROL -- TRANSMIT_IR -- UNINSTALL_SHORTCUT      |
|   20     |   Kitkat Watch, 4,4W (2014.6)     |   BODY_SENSORS     |
|   21     |  Lollipop, 5.0 (2014.11)      |   BIND_DREAM_SERVICE -- BIND_TV_INPUT -- BIND_VOICE_INTERACTION -- READ_VOICEMAIL -- WRITE_VOICEMAIL     |
|   22     | Lollipop MR1, 5.1 (2015.3)       |   BIND_CARRIER_MESSAGING_SERVICE **注意：此权限在Api level23的时候被弃用，采用BIND_CARRIER_SERVICES来代替。 **    |
|   23     | Marshmallow, 6.0       |   ACCESS_NOTIFICATION_POLICY -- BIND_CARRIER_SERVICES -- BIND_CHOOSER_TARGET_SERVICE -- BIND_INCALL_SERVICE -- BIND_MIDI_DEVICE_SERVICE -- BIND_TELECOM_CONNECTION_SERVICE -- GET_ACCOUNTS_PRIVILEGED -- PACKAGE_USAGE_STATS -- REQUEST_IGNORE_BATTERY_OPTIMIZATIONS -- REQUEST_INSTALL_PACKAGES -- USE_FINGERPRINT     |
|   24     | Nougat, 7.0       |   BIND_CONDITION_PROVIDER_SERVICE -- BIND_QUICK_SETTINGS_TILE -- BIND_SCREENING_SERVICE -- BIND_VR_LISTENER_SERVICE     |

参考：https://developer.android.com/reference/android/os/Build.VERSION_CODES.html
完整权限名，请参考：https://developer.android.com/reference/android/Manifest.permission.html

Android版本与Api level的对应关系，请参考：https://developer.android.com/guide/topics/manifest/uses-sdk-element.html#ApiLevels

————————————————

  原文链接：https://blog.csdn.net/u013553529/article/details/53167072
