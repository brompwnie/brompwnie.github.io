---
layout: post
title: "Kotlin and Java: How Hackers See Your Code"
categories: reversing
author: "brompwnie"
meta: "android hacking"
---

Hello! In this blog post, I'll be sharing with you a recent experiment I did with Java and Kotlin Android applications. Kotlin is new to the Android space and is picking up momentum and with this in mind, I recently asked myself "I wonder how I am going to reverse Android applications built with Kotlin?". This question I feel is common amongst Android reversers because when you are targeting an Android application (APK), your approaches and tools will differ according to whether the app is using something like Apache Cordova, native Java or Kotlin. After dwelling on this for sometime, I decided to setup a small experiment, which would allow me to see how some commonly used reversing tools would react to Kotlin applications. The remainder of this blog post describes what I did and the anomalies I observerd! Enjoy!

## Step 1: Create A Java and Kotlin Android Application
In this step, I wanted to create a basic AKA vanilla Android application in Java and Kotlin so that I could have two applications to compare against. I wanted the applications to be simple and utilize as many defaults as possible. For this step I created the applications in Android studio and used the defaults. I called the applications "HelloJava" and "HelloKotlin" respectively.

![HelloJava](/assets/HelloJava.png)


## Step 2: Add Functionality  
In this step I wanted to create a  piece of functionality that could be easily replicated across both applications and of course, easy to do (side note, it took me more time than I would like to admit to when it came to figuring out how to make the method as seen in Kotlin app ;). I decided to create a simple private method in the Main activity of the applications which would be called when the activities were created AKA the onCreate method.

![HelloJava](/assets/functionality.png)

![HelloKotlin](/assets/HelloKotlin.png)

## Step 3: Build Signed APK's
In this step, we want to generate a signed release APK for each application. This is done because we don't want any debug symbols (debug release) in our APK's as this will add another variable into our analysis and essentially add a lot of "noise" during our analysis. In this step, we will use the defaults provided to us from Android studio.

![HelloJava](/assets/signing.png)
The image above shows that we are using the Android Studio defaults and the signing scheme for the applications. The image below is just some logcat output from the applications when they run on a device.

![HelloJava](/assets/aMessage.png)

## Step 4: Do a Bit of Analysis
In this step we are going to look at our two APKs, one for Java and one for Kotlin. Firstly let's have a look at the size of our two APKs.

![APKS](/assets/APKS.png)

Here we see our first anomaly, the Kotlin app is larger in size(1.8MB) than that of its Java counterpart(868KB). Hmmmm okay lets take this a step further. We know that APK's are essentially ZIP like packages with a stack of things in them so lets compare the DEX file for each application because the DEX file is essentially the "core" of the application code which is what we want to analyze. I wont go into too much detail about the DEX file but what we need to know is that all our Java and Kotlin code lands up in this file once it is compiled.

![APKS](/assets/dexSize.png)

To get this point, I unzipped each APK with the following command 

```html
unzip HelloKotlinNotObfuscated.apk -d HelloKotlinNotObfuscated
```
which gave me a folder with all the raw contents of the APKs, more specifically, the DEX files.

## Step 5: Lets Reverse Some Code
There are many ways to reverse Android applications and at this step, we are going to look at two tools, Jadx and APKTOOL. Both have their strengths and weaknesses and are a good starting point if you want to reverse Android applications. So how do these two tools function when we feed them the Java and Kotlin applications? Currently we have two APK's which means we don't have much to work with unless you like staring at byte code (which some do and nothing wrong with that) but lets use Jadx to see what we can get.

First lets have a look at the Java application which is shown below.

![APKS](/assets/jadxJava.png)

The image above shows that we can successfully get the Java code and all the usuals from the Java application i.e translated the DEX file into Java. Great, easy-peasy-lemon-squeezy.

Lets now do the same thing but with our Kotlin application.

![APKS](/assets/jadxKotlin.png)

Oookay the image above looks slightly different from the Java version and appears to contain some weird looking "stuff" that makes the code less clean than the Java counterpart. However we are still able to see the private method we created and still get a sense of what the code is doing but there is just some extra code and weirdness.

## Step 6: But that's not all

Step 5 only looked at the applications using a single tool to get a single view of the code, the Java code. But that is not the only way to analyze our target DEX files. Ideally we would want to look at target DEX code and the Java equivalent but that is not always the case. In such cases, we can look at the SMALI representation of the code. I won't go in too much detail about SMALI but what is important for us to know is that SMALI can be seen as an intermediate representation of the DEX code such that it sits in-between the DEX and Java code, slightly easier to read than DEX but not as easy to read as Java. Okay enough about that, lets look at some SMALI.

![Proguard](/assets/apktoolJava.png)

![Proguard](/assets/apktoolKotlin.png)

The poorly cropped images above show the results of running the following apktool command to get the SMALI for each application, 
```html
apktool d HelloJavaNotObfuscated.apk
```
 After running this command, apktool by default will create a directory of the decompiled resources in a folder which is named after the name of the targeted APK i.e HelloJavaNotObfuscated. As we can see from the images above, apktool created an extra folder for the Kotlin application called "Kotlin". But that's not what we want to look at right now, we want to look at the MainActivity files for each application.

![Proguard](/assets/smaliFolderK.png)

![Proguard](/assets/smaliFolderJ.png)

To view the Smali generated for the applications, apktool places the files in the folder Smali in the top level generated folder and placed individual files according to their packages. The images above show the contents of the folders "com.brompwnie.java" and "com.brompwnie.kotlin" which contains the MainActivity.smali for each application. It appears the same amount of files have been generated for both the Java and Kotlin application.


![Proguard](/assets/twoSmalis.png)

The image above shows that the MainActivity.smali generated for the Kotlin and Java applications differ significantly. The Kotlin MainActivity.smali contains 173 lines of Smali and the Java MainActivity.smali contains 71 lines of Smali. 

![Proguard](/assets/onCreateMethods.png)

However, from the image above, it can be seen that the two MainActivity.smali's both actually have the same onCreate and aPrivateMethod methods. It appears that that the contents of the two files are indeed similar as well. The Kotlin application has some extra code injected at the "top" but the remainder of MainActivity.smali is the same as the Java MainActivity.smali. Interesting. It appears that when we have to reverse Kotlin applications, we may have more code to analyze. However at this point, we've been dealing with unobfuscated applications which leads me to the next step.


## Step 7: Lets Reverse Some Obfuscated Code
Reversing unobfuscated code is always a win but obfuscation is a common technique used to hinder reversers so lets now go ahead and recompile our Java and Kotlin applications to use some obfuscation. We will make use of Proguard that comes with Android Studio and use the Proguard defaults. By using obfuscation, we will gain some insight as to how our applications will look when they are reversed.

![Proguard](/assets/proguard.png)

The image above shows the "minifyEnabled" flag set to true which allows our Android applications to be compiled with obfuscation which will be done on the DEX code.


 ![Proguard](/assets/obfuscatedAPKS.png)

 Keeping to the steps taken with the unobfuscated applications, we can see that the size of the obfuscated Java and Kotlin applications both differ but are also smaller than their unobfuscated counterparts. This is expected as Proguard performs optimization which often translates to a level of obfuscation. This then leads us to have a look at the DEX files for each application.

 ![Proguard](/assets/obfuscatedJavaDex.png)

 The image above shows the contents of the exploded obfuscated Java APK which shows us that the size of classes.dex is 826KB.

  ![Proguard](/assets/obfuscatedKotlinDex.png)

The image above shows the contents of the exploded obfuscated Kotlin APK which shows us that the size of classes.dex is 826KB, the exact same size of the obfuscated Java application!

Another interesting observation is that most of the files in the APK are the same size, the only differences are the folders "META-INF" and "kotlin".

## Step 8: Lets Look at some Obfuscated Code
In the previous step, we were looking at some meta differences between the two obfuscated applications. In this step, we are going to use good 'ol apktool and jadx to look at our obfuscated applications and see what we get.

![Proguard](/assets/jadxObfuscatedBothApps.png)

The image above shows us that when we use jadx on both the applications, we get the exact same decompiled MainActivity class! Upon further analysis, we can see that the obfuscated APKS for both the Java and Kotlin applications are almost identical with the two exceptions mentioned above. However this is one view of the code, lets see how apktool reacts to the two applications.

![Proguard](/assets/apktoolObfSmali.png)

The image above shows the contents of the directories "com.brompwnie.java" and "com.brompwnie.kotlin" in the smali folder for the two applications. In this case, there is only one file, MainActivity.smali compared to the multiple files that the unobfuscated applications had. Additionally, the size of the Kotlin and Java MainActivity.smali files are exactly the same. This was expected due to what we saw with jadx because apktool and jadx make use of the same file to decompile, classes.dex.

## Conclusion
To summarise our observations, we can see that unobfuscated Kotlin and Java applications will produce different output (classes.dex and APK's) but obfuscated Kotlin applications will produce Java like applications. This is particularly interesting as it can be assumed that if an application is developed using Kotlin, it can't be decompiled because only Java tools exist and Kotlin is "different". The results above show that this is not always the case and Android developers should take note of how their code will be seen by hackers and what they can do to protect their code, if need be.

The reasons for these observed anomalies are interesting and will be discussed further in part 2 of this blog post. Until then, if you made it this far, thanks for reading!

### Useful Links

[https://ibotpeaches.github.io/Apktool/](https://ibotpeaches.github.io/Apktool/)

[https://github.com/skylot/jadx](https://github.com/skylot/jadx)

[https://developer.android.com/studio/index.html](https://developer.android.com/studio/index.html)

