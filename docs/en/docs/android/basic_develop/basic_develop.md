# Android Development Basics

Before getting into Android security, we should understand the basic workflow of Android development as much as possible.

## Fundamental Knowledge

Read the following books in order, progressing from beginner to advanced levels of Android development knowledge:

- "The First Line of Code", reading the first seven chapters is sufficient
- JNI/NDK development, no suitable guide has been found yet.
- "Android Programming: The Big Nerd Ranch Guide" (optional)
- "Android Advanced Programming" (optional)

During the learning process, I personally think the following knowledge areas in Android development deserve special attention:

- Android system architecture
- Basic source file structure
- Basic development methods and coding conventions, understanding the meaning of common code.
- Understanding file formats for configuration resources such as xml.

**Make sure to set up a proper Android development environment!!!!!**

- java
- ddms
- ndk
- sdk, install multiple versions of the sdk, 5.0-8.0

## APK Packaging Process

After writing the App-related code, the final step is to package all the resource files used in the App. The packaging process is shown in the figure below (<u>http://androidsrc.net/android-app-build-overview/</u>):

![](./figure/android_app_build.png)

The specific steps are as follows:

1. Use aapt (The Android Asset Packing Tool) to package resource files and generate the R.java file.
2. If the project uses services provided by AIDL (Android Interface Definition Language), the AIDL tool is needed to parse the AIDL interface files and generate the corresponding Java code.
3. Use javac to compile R.java and AIDL files into .class files.
4. Use the dx tool to convert class files and third-party libraries into dex files.
5. Use apkbuilder to package the resources compiled in the first step, the .dex file generated in the fourth step, and some other resources into an APK file.
6. This step mainly involves signing the APK. There are two scenarios: if we are releasing the App, we use a ReleaseKeystore for signing; otherwise, if we just want to debug the App, we use debug.keystore for signing.
7. Before releasing the official version, we need to modify the starting offset of resource files in the APK package to be a multiple of 4 bytes. This way, the App will run faster afterwards.

## APK File Structure

An APK file is also a type of ZIP file. Therefore, we can use zip extraction tools to decompress it. The structure of a typical APK file is shown in the figure below. The description of each part is as follows:

![](./figure/apk_structure.png)


- AndroidManifest.xml

    - This file is mainly used to declare the application's name, components, permissions, and other basic information.

- class.dex
    - This file is the executable file for the Dalvik virtual machine, containing the application's executable code.
- resource.arsc
    - This file mainly contains the application's compiled binary resources and the mapping between resource locations and resource IDs, such as strings.
- assets
    - This folder is generally used to contain the application's raw resource files, such as fonts and music files. The program can access this information through APIs at runtime.
- lib/
    - The lib directory is mainly used to store native library files used through the JNI (Java Native Interface) mechanism, and subdirectories are created for each supported architecture.
- res/
    - This directory mainly contains the resources referenced by the Android application, stored by resource type, such as images, animations, menus, etc. There is also a values folder that contains various attribute resources:
- colors.xml --> Color resources
- dimens.xml ---> Dimension resources
- strings ---> String resources
- styles.xml --> Style resources
- META-INF/
    - Similar to JAR files, APK files also contain a META-INF directory, used to store code signatures and other files, to ensure that the APK file is not tampered with.
