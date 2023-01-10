

# Lab 03: Honey pots

## Introduction

In this lab we'll explore honey pots. Many security and IT professionals misunderstand what honey pots are. Typically, they think they're expensive to setup and maintain... or worse yet, somehow can introduce additional vulnerabilities in your system. We will be showing you a better way to "honey-ify" your systems. Honey pots are decoy objects that lure attackers into interacting or using them. Honey pots serve no purpose other than to fool the attacker. Because they have no business purpose, any interaction or use of the honey object is suspicious. This makes monitoring dramatically easier.

Honey pots come in two forms, complex honey networks/systems and low effort honey objects like files, user accounts, or web elements. In this lab we will focus on the second type... the light touch, low effort honey pots that can aid in detecting attackers, while also frustrating them. 



## Learning Objectives

This lab will show how to use active defense measures to frustrate and annoy our adversaries. We will do this with multiple honey objects.

1. Honey directory
2. Honey user
3. Honey data



## Prerequisites ##
This lab requires Docker (Docker Desktop or otherwise) and 2 container images (Juice Shop and modsecurity).  See the README at [this repository](https://github.com/andrewdouglas/CodeMash2023-AppSec) for instructions on obtaining the prerequisite container images.  If today is the precompiler session and you don't have the images, don't worry - grab a flash drive from the front (these images are somewhat large and WIFI may not be adequate).  **If you downloaded the images from the flash drive**, you'll need to use 'docker load' vs. 'docker pull' like this:

In Mac/Linux (run one command at a time):
```
docker load < juice-shop.tar
docker load < modsecurity.tar
```
In Windows (run one command at a time):
```
docker load -i juice-shop.tar
docker load -i modsecurity.tar
```



## Method 1: 404s as honey URI

Attackers have a repeatable process that is used during assaults against systems. Part of this attack lifecycle is reconnaissance. Attackers must learn as much as they can about your target systems, and the more they know the more success they will have in attacking your host.

One of the most common early reconnaissance methods used by attackers are discovering the directory structure in a web application. Attackers will use sitemaps and crawl all available links on a site in an attempt to discover all components. 

However, attackers also know that there are likely administrative and operational parts of a site that are not meant to be accessed by typical users. Some developers think that "isolating" these parts by not linking to them in the rest of the site is enough to hide, sadly this is not true.

Attackers will use tools like `DirBuster` ([website](https://sourceforge.net/projects/dirbuster/)) and [publicly available lists](https://github.com/daviddias/node-dirbuster/tree/master/lists) of common directories found in many web applications. 

With a little know how, we can use this common attack against them!

Using this [small directory list](https://github.com/daviddias/node-dirbuster/blob/master/lists/directory-list-2.3-small.txt), we can see some of the directories attackers may attempt to access. Let's set a honey pot based on this knowledge! 

You don't need to download the directory list now, but on [line 44](https://github.com/daviddias/node-dirbuster/blob/master/lists/directory-list-2.3-small.txt#L44), there's a entry called "archives". This directory does not exist in Juice Shop. We can now use this non-existing directory as a honey pot!

1. Start Juice Shop

   ```
   docker run --rm -p 3000:3000 bkimminich/juice-shop:v14.3.1
   ```

2. Start ModSecurity

   In Mac/Linux:
   ```
   docker run -p 3001:8080 --rm \
      -v /opt/apsec/waf/logs:/var/log/apache2 \
      -e PARANOIA=1 \
      -e BACKEND=http://172.17.0.1:3000 \
      -e PORT=8080 \
      -e PROXY=1 \
      -e LOGLEVEL=warn \
      -e ACCESSLOG=/var/log/apache2/access.log \
      -e ERRORLOG=/var/log/apache2/error.log \
      owasp/modsecurity-crs:3.3.4-apache-202211240811
   ```
   
   In Windows Powershell:
   ```
   docker run -p 3001:8080 --rm `
      -v $env:USERPROFILE\apsec\waf\logs\:/var/log/apache2 `
      -e PARANOIA=1 `
      -e BACKEND=http://172.17.0.1:3000 `
      -e PORT=8080 `
      -e PROXY=1 `
      -e LOGLEVEL=warn `
      -e ACCESSLOG=/var/log/apache2/access.log `
      -e ERRORLOG=/var/log/apache2/error.log `
      owasp/modsecurity-crs:3.3.4-apache-202211240811
   ```

   

3. Run curl against a real/expected resource

   In Mac/Linux:
   ```
   curl http://localhost:3001/
   ```

   In Windows Powershell:
   ```
   Invoke-WebRequest http://localhost:3001/
   ```

4. Look for 404s

   In Mac/Linux:
   ```
   grep 404 /opt/apsec/waf/logs/access.log
   ```

   In Windows Powershell:
   ```
   Select-String 404 $env:USERPROFILE\apsec\waf\logs\access.log
   ```

5. Now generate some 404s

   In Mac/Linux:
   ``` 
   curl http://localhost:3001/archives
   ```

   In Windows Powershell:
   ```
   Invoke-WebRequest http://localhost:3001/archives
   ```

6. Look for 404s

   In Mac/Linux:
   ```
   grep 404 /opt/apsec/waf/logs/access.log
   ```

   In Windows Powershell:
   ```
   Select-String 404 $env:USERPROFILE\apsec\waf\logs\access.log
   ```

> Note: While doing this manually on production sites is not feasible other than specialized attacker hunting activities, it is a good idea to put 404s into a log analysis system to look for `dirbuster` style attacks. Because 404s are common, you will likely want to do some statistical analysis. The authors of this lab suggest looking for IPs that are generating 404 status codes at 2 or 3 standard deviations over your normal sample set. 

Congratulations, you've now used the lowly 404 error code as honey pot!



## Method 2: Application logs looking for honey user

Attackers will often use OSINT or go on the dark web to get lists of users. This is often done for a class of attacks called "credential suffing". Armed with these lists, attackers will frequently try to brute force web application logins. While it's unfortunate when attackers get these lists, you could turn some leaked accounts into honey accounts.

Let's suppose an internal user at our organization has an account named "fliggins" that was part of a leaked set. For security's sake, you likely will have disabled this account and issued a new one to the impacted user. If you want, you can now "reuse" this account as a honey account!



1. Verify that Juice Shop and ModSecurity are still running:

   ```
   docker ps
   ```

2. Look for failed login attempts. (Hint: failed logins generate 401: Not authorized error messages)

   In Mac/Linux:
   ```
   grep 401 /opt/apsec/waf/logs/access.log
   ```

   In Windows Powershell:
   ```
   Select-String 401 $env:USERPROFILE\apsec\waf\logs\access.log
   ```

3. Note: There might not be any existing 401s depending on what you did in prior labs.
   If there are no failed logins, generate some by opening your browser and attempting to login with any invalid username and password. (you can simply guess!)

   

Congratulations, you've now used the lowly 401 error code as honey pot!

