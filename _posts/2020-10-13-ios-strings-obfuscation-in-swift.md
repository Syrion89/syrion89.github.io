---
layout: post
title:	"iOS Strings Obfuscation in Swift"
date:	2020-10-13
categories:
    - blog
tags:
    - iOS
---


![Screenshot]({{ site.baseurl }}/images/2020-10-13-ios-strings-obfuscation-in-swift/img0.png)

### Introduction

Usually when reversing an iOS Application, it’s common to see methods and strings that can help an attacker to figure out how the application works. 
When I’m looking for jailbreak detection mechanisms, I usually start to search for strings and functions containing the word **“jailbr”** (**jail**, **jailbreak** or **jailbroken**) or **“root”**. If I’m not lucky with these method names, I start to search common strings used by anti-jailbreak methods, for example **“cydia://”**, **“/bin/bash”** and other words like these. While most developers rename their anti-jailbreak method, no one take care of the strings. Using these words an attacker can easily figure out the purpose of the methods.

### Anti-Jailbreak Check Example

For the purpose of this blog post, I wrote a simple application that shows a message indicating if the device is **jailbroken** or not by checking the presence of **Cydia**. This is just a little example, I took this part of code from the [IOSSecuritySuite](https://github.com/securing/IOSSecuritySuite), if you are looking for a real anti-jailbreak mechanism, you should take a look at the suite.

~~~
func canOpenUrlFromList(input: String) -> Bool {
    if let url = URL(string: input) {
        if UIApplication.shared.canOpenURL(url) {
            return true
        }
    }
    return false
}
~~~

~~~
func isJailbroken(){
    
    let name = "cydia://"
    if(canOpenUrlFromList(input: name )){
        message.text = "Jailbroken!"
    }
    else{
        message.text = "Not Jailbroken!"    
    }
}
~~~

The method **isJailbroken** calls the **canOpenUrlFromList** method and give the string **“cydia://”** as parameter, it will return a Boolean value indicating if the url **“cydia://”** is opened or not, then the application shows the message **“Jailbroken!”** or **“Not Jailbroken!”**.
If we run the application on a the device, we will see the string indicating the device is not **jailbroken** as showed in the image below.

![Screenshot]({{ site.baseurl }}/images/2020-10-13-ios-strings-obfuscation-in-swift/img1.png)

If we reverse the application using Hopper, we can find the method name (It’s obvious) and the strings as report in the two images below.

![Screenshot]({{ site.baseurl }}/images/2020-10-13-ios-strings-obfuscation-in-swift/img2.png)

![Screenshot]({{ site.baseurl }}/images/2020-10-13-ios-strings-obfuscation-in-swift/img3.png)

By looking for the string **“cydia://”** references, we can easily find the method that performs the anti-jailbreak mechanism.

![Screenshot]({{ site.baseurl }}/images/2020-10-13-ios-strings-obfuscation-in-swift/img4.png)

![Screenshot]({{ site.baseurl }}/images/2020-10-13-ios-strings-obfuscation-in-swift/img5.png)

The same would happen if we had an obfuscated method name. How to solve this problem? We must obfuscate our strings!

### Obfuscated Strings

For this article I used the [CryptoSwift](https://github.com/krzyzanowskim/CryptoSwift) Module in order to encrypt my strings using **AES** with a **128 bit** key.
I created 2 methods to encrypt and decrypt our strings, In the application we only need the decrypt one, because our strings must be already encrypted in the application.

~~~
func aesEncrypt(myString: String, myKey: String, myIv: String) throws -> String {

    let aes = try AES(key: myKey, iv: myIv)
    let ciphertext = try aes.encrypt(Array(myString.utf8))
    return ciphertext.toBase64()!
}
~~~

~~~   
func aesDecrypt(myString: String, myKey: String, myIv: String) throws -> String {

    let aes = try AES(key: myKey, iv: myIv)
    let data =  try myString.decryptBase64(cipher: aes)
    return String(bytes: data, encoding: .utf8)!
}
~~~

The method takes the string to encrypt/decrypt, the secret key and the IV. The encrypt method returns the **base64** of the encrypted string while the decrypt return the original string from the **base64**. I modify the example we have seen before as shown below.

~~~
func a2feb289b83e9045ba93f957e304fabb6(input: String) -> Bool {
    
    if let url = URL(string: input) {
        if UIApplication.shared.canOpenURL(url) {
            return true
        }
    }
    return false
}
~~~

~~~
func a565b6787dfc3f65ee7678dc7c9742e7b(){
    
    let key = "Passw0rdPassw0rd"
    let iv = "drowssapdrowssap"
    
    do{
        let name = try aesDecrypt(myString: "sjmIgYddguu/wdfkAi7xfw==",myKey: key ,myIv: iv)
        
        if(a2feb289b83e9045ba93f957e304fabb6(input: name )){
            message.text = try aesDecrypt(myString: "yVI76S8UGSSANGYjofb9wA==",myKey: key,myIv: iv)
        }
        else{
            message.text = try aesDecrypt(myString: "FtF69D9RLKTlQJVGvOwzNw==",myKey: key,myIv: iv)
        }
    }
    catch {
        message.text = "Error!"
    }
}
~~~

If we try to reverse again, we will not find anything related to the word **“jail”** and **“cydia”**.

![Screenshot]({{ site.baseurl }}/images/2020-10-13-ios-strings-obfuscation-in-swift/img6.png)

![Screenshot]({{ site.baseurl }}/images/2020-10-13-ios-strings-obfuscation-in-swift/img7.png)

The attacker will only find a lot of base64 strings.

![Screenshot]({{ site.baseurl }}/images/2020-10-13-ios-strings-obfuscation-in-swift/img8.png)

**NOTE 1**: This is just a basic example using **AES**  with a weak password and a weak IV of 16 bytes (128 bits), how to generate a strong key and IV is not in the scope of this blog post, please don't use the this key and IV! 

**NOTE 2**: The attacker will be able to see the secret key and the IV, how to safe manage the key and the IV is not in the scope of this blogpost and there are plenty of blog explains how to do it in the right way.

![Screenshot]({{ site.baseurl }}/images/2020-10-13-ios-strings-obfuscation-in-swift/img9.png)

You can encrypt the strings with [CyberChef](https://gchq.github.io/CyberChef).

![Screenshot]({{ site.baseurl }}/images/2020-10-13-ios-strings-obfuscation-in-swift/img10.png)

### Conclusion

This obfuscation is a **security in depth**/** defence in depth ** practice, it will not stops an experienced attacker, as for the jailbreak checks the purpose of these methods is to make the reverse engineering more time consuming for the attacker, and trust me, when you’re reversing an application with a lot of checks, it’s not very nice to not see the strings the application is using.
I’m not an iOS application developer, I’m sure a good developer can do it in a better way. Let me know if there are errors or doubts!
