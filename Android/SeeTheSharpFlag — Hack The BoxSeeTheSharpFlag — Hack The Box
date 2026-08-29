SeeTheSharpFlag — Hack The Box Mobile Challenge Write-up
**🧭 Overview**

The goal of this challenge was to analyze an Android application and retrieve the secret flag by reversing its internal logic.

---

**🔎 Static Analysis**

After downloading the APK, I installed and launched the application.
The UI was very simple and contained:

An input field prompting: “Enter the secret”
A button to validate the input
Testing with random values resulted in the message:

> `Sorry, not correct password
`

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/irugi63bkr6xqt63udkh.png)

---

**📦 Decompilation & Framework Identification**

I loaded the APK into JADX for static analysis.
However, there was no meaningful validation logic in the source code.

While inspecting the structure, I noticed that the app was built using the Xamarin framework.
In Xamarin apps, most of the business logic resides inside managed .dll assemblies rather than the Java layer.

---

**🗂 Extracting Assemblies**

To access the assemblies:

1-Renamed the .apk file to .zip 
`mv com.companyname.seethesharpflag-x86.apk com.companyname.seethesharpflag-x86.zip`

2- Extracted its contents
3- Located multiple DLL files inside the packag


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sd4qhzgf6qgz610rqmcg.png)


Two interesting assemblies were identified:

`- SeeTheSharpFlag.dll
``- SeeTheSharpFlag.Android.dll
`
Based on naming conventions, **SeeTheSharpFlag.dll** was the most likely candidate to contain the core application logic, while **SeeTheSharpFlag.Android.dll** seemed to represent the Android-specific implementation layer.

(The remaining DLL files appeared to be framework libraries with no application-specific logic.)

---
**🧩 Handling Xamarin Compression**

Xamarin often compresses assemblies using algorithms such as XALZ.


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bbavjhkin3oawi3ew41q.png)
Because of that, the DLL could not be analyzed directly.

To resolve this, I used the Xamarin decompression tool:

https://github.com/NickstaDB/xamarin-decompress

After decompression, the assembly became suitable for reverse engineering.


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/91osm845tfqsnks9z30n.png)

---
**🔬 Reverse Engineering the Assembly**

The decompressed DLL was analyzed using dotPeek.

[https://www.jetbrains.com/decompiler/download/?section=web-installer](https://www.jetbrains.com/decompiler/download/?section=web-installer)

Let’s see what I found:


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xc30cds4bj3ugnex37h3.png)


- A ciphertext stored as a Base64 string
- An AES key encoded in Base64
- An IV encoded in Base64

This confirmed that the application validates the input by decrypting a hardcoded AES ciphertext.

Using CyberChef you simply decrypt it and solve the challenge.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/58idr4d8hfrueur227zd.png)

I entered the recovered flag into the application.
The app responded with:

> `Congratz! You found the secret message`


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3a37g4mi7w0bpuby0bma.png)



**Why so serious? The flag was just the punchline** 


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5y7hb5bl4ia3y55o39cl.png)


