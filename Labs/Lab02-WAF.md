# Lab 02: Web Application Firewalls (WAF)

## Introduction:

In this lab we will use a web application firewall (WAF) to filter malicious traffic. We will also explore a WAF's logging capabilities. 

> Important security considerations for WAFs: 
> In this lab, we will show how a WAF can be used to rapidly improve the security posture of a web application. While WAFs are useful tools, it's important to remember they are not perfect. Researchers are continually discovering WAF bypass techniques. Later in this lab, we will even highlight some interesting limitations! 
>
> While you should use a WAF as part of your overall security posture, you should not rely only on them. It is vital that code be made as resilient to attack as practical. Additionally the infrastructure the code runs on should be hardened per consensus driven guides such as those published by the Center for Internet Security.

Best practice is to use a WAF for:

- Providing passive protections against common attacks
- Temporary "virtual patching" of known software flaws
- An additional layer of monitoring/detection




## Learning Objectives

- Passive WAF filtering with community rules
- Using a WAF as a centralized logging point
- Analysis techniques attackers use when WAFs are present 



## Prerequisites:

- Docker (Docker Desktop or the Docker CLI engine)
- The bkimminich/juice-shop:v14.3.1 container
- The OWASP/modsecurity-crs:3.3.4-apache-202211240811 container
- The owasp/zap2docker-stable:2.12.0 container



## Phase 1: Using a WAF to block common attacks

Common attacks are commonly used because they commonly work! The good news is that this makes them a known commodity. Most WAFs offer subscriptions to either commercial or community based rule sets which are updated frequently. These rules allow a WAF to do what's called "deep packet inspection". Though it may sound fancy, Deep Packet Inspection is nothing more than looking for attack indicators on traffic being sent to the web application.

In this portion of the lab, we will show how a vulnerable web application can become more resilient to attack without needing to adjust any of the code.

1. Start the target site, Juice Shop. 
   Open a terminal and start the target container, with the following command:
   
   ```
   docker run --rm -t -p 3000:3000 bkimminich/juice-shop:v14.3.1
   ```

2. Start the WAF, ModSecurity
   Open a new terminal and copy and paste the entire multi-line command.

   In Mac/Linux:
   ```
   docker run -t -p 3001:8080 --rm \
      -e PARANOIA=3 \
      -e BACKEND=http://172.17.0.1:3000 \
      -e PORT=8080 \
      -e PROXY=1 \
      owasp/modsecurity-crs:3.3.4-apache-202211240811
   ```
   In Windows Powershell:
   ```
   docker run -t -p 3001:8080 --rm `
      -e PARANOIA=3 `
      -e BACKEND=http://172.17.0.1:3000 `
      -e PORT=8080 `
      -e PROXY=1 `
      owasp/modsecurity-crs:3.3.4-apache-202211240811
   ```
   
   > Note: we are configuring the WAF to listen on port 3001, and forwarding traffic to the Juice Shop instance running on port 3000.
   > This will allow us to have:
   > 3000 -- a vulnerable web service
   > 3001 -- the same web service protected by our WAF
   > in a "real world" scenario, you would never allow direct access to a vulnerable web service. We are doing this only to make the rest a series of A/B tests.

3. Now we will conduct a SQL injection attack to bypass the Juice Shop login screen.

   Open your browser and go to http://localhost:3000

   Next, click "Account --> Login" to be prompted with the login page

   In the "Email" field enter the following:
   `' OR 1=1--`

   in the "Password" filed you can put whatever you want.

   Finally, click "Log in"

   > Note: because Juice Shop is vulnerable to SQL injection in this spot, we're fooling the database. Normally, the database is looking for situations where the supplied email and password match those found in the Users table. If the email and password match, the login system gets a TRUE response, meaning you're allowed to proceed as as that user. However, because we've used `OR 1=1,--` we're tricking the web application. Even if we fail to provide a valid username and password the `OR` clause will trigger. Since 1=1 is always true... we're able to login without knowing any username or password! The `--` is the SQL comment marker. Anything after the `--` will be ignored. This allows the query to be syntactically correct.

   

4. Next, we will attempt to do the SQL injection attack again, but this time through the WAF that is running on port 3001: 

   Open a new browser window or tab and go to http://localhost:3001
   
   Next, click "Account --> Login" to be prompted with the login page
   
   In the "Email" field enter the following:
   `' OR 1=1--`
   
   in the "Password" filed you can put whatever you want
   
   Finally click "Log in"

   > Note: in this case, the login did not work! The WAF saw the SQL injection attempt and correctly blocked the login attempt. We know that the WAF did the block because we were returned a 403 status code. This code is used for when a web client attempts to access a valid URL (as opposed to an invalid/non-existent URL which gives a 404), but is not permitted to access the resource. 

   

## Phase 2: Leveraging WAF logging

While the ability to filter suspicious traffic is compelling enough reason to use a WAF, it's not the only reason they should be used. In many instances, WAFs provide superior logging capabilities! You may wish to explore using a WAF for logging -- in addition to the filtering capabilities if:

- You have many legacy web applications/services which have limited or no logging options. This is a common issue for embedded systems.
- Your architecture is made up of many different types (or even versions) of web applications and services. The resulting inconsistent log formats can be difficult to parse and track.
- The web systems behind the WAF are "thinnly provisioned" and have limited processing power. This is an increasingly common issue with systems like Kubernetes clusters. By shifting logging onto the WAF, the CPU, RAM, and disk I/O of the web services are offloaded to a more powerful system, which allows these minimal web containers to be lightweight and responsive.

1. (optional) If you accidentally stopped the Juice Shop container, please restart it with this command:
   
   ```
   docker run -t --rm -p 3000:3000 bkimminich/juice-shop:v14.3.1
   ```

2. Create a directory for WAF logging with this command:
   
   In Mac/Linux:
   ```
   sudo mkdir -p /opt/apsec/waf/logs
   ```
   In Windows Powershell:
   ```
   mkdir $env:USERPROFILE\apsec\waf\logs
   ```

3. Re-launch the ModSecurity WAF and use a persistent volume to mount the logging directory into the container with the following command:

   In Mac/Linux:
   ```
   docker run -t -p 3001:8080 --rm \
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
   docker run -t -p 3001:8080 --rm `
      -v $env:USERPROFILE\apsec\waf\logs:/var/log/apache2 `
      -e PARANOIA=1 `
      -e BACKEND=http://172.17.0.1:3000 `
      -e PORT=8080 `
      -e PROXY=1 `
      -e LOGLEVEL=warn `
      -e ACCESSLOG=/var/log/apache2/access.log `
      -e ERRORLOG=/var/log/apache2/error.log `
      owasp/modsecurity-crs:3.3.4-apache-202211240811
   ```

4. Run another light touch scan
   
   ```
   docker run --rm -t --network host owasp/zap2docker-stable:2.12.0 zap-baseline.py -t http://localhost:3001
   ```

5. Once the scan completes, press CTRL+C in the terminal running the WAF to terminate it (if running Docker Desktop, you may need to navigate to the "Containers" option and click the stop button).

6. Go to the directory with the logs:
   
   In Mac/Linux:
   ```
   cd /opt/apsec/waf/logs
   ```
   In Windows Powershell:
   ```
   cd $env:USERPROFILE\apsec\waf\logs
   ```

7. You can see that our access.log and error.log files are there:
   `ls`

   You should see output that shows 2 files: access.log and error.log.

8. Open the access.log file using a tool of your choosing e.g.:
   `cat access.log`, or using Notepad, etc.

   Observe how fast all the requests happened! Typically web traffic is sporadic, and will have some time between the requests. All too often defenders try to do fancy analysis, when quick checks like looking for bursts of high volumes of traffic in short a time frame is enough to begin a more thorough investigation. 

   Another avenue of analysis should be the User Agent String. Although attacker tools may attempt to blend in (ZAP tries to appear to be a fairly recent build of Firefox), many other tools do NOT. A perfect example of this is SQLMap, a powerful SQL injection tool.  SQLMap has a default user agent string of "SQLMap ver VER NUMBER HERE". If you're not looking at user agent strings, you are leaving a low effort, yet effective, method of investigation unused.



## Phase 3: WAF limitations

As stated at the start of this lab, WAFs are not perfect. In the first section of this lab, we did an interactive attack against the Juice Shop application. Because WAFs work by looking at the content received from web clients (again, this is sometimes called Deep Packet Inspection), they do a great job of preventing many attacks from being successful. While exceptionally helpful, some mistake the level of protection a WAF provides.

Any web scanning tool worth using -- including ZAP -- has a "passive scan mode". Unlike active scans where malicious content is sent to a web application, a passive scan works by crawling the site like a 'normal' user. All output from the site is then automatically -- and carefully -- reviewed. While passive scans cannot detect injection style attacks, they can do an amazing job at finding other potentially exploitable vulnerabilities. Because passive scans never send malicious content through a WAF, an attacker has an excellent chance of learning about existing flaws... without the targeted organization being aware! Remember, this looks like a typical bot crawling a site.

In this phase of the lab, we will show you how a WAF typically provides no protection against passive scanning.

1. Open a new terminal window and conduct a passive scan directly against Juice Shop
   
   ```
   docker run --rm -t --network host owasp/zap2docker-stable:2.12.0 zap-baseline.py -t http://localhost:3000
   ```

   > Note: for best results, please let this scan finish before proceeding. It should take approximately 1 minute.

2. Open another terminal window and conduct a passive scan against Juice Shop through the WAF
   
   ```
   docker run --rm -t --network host owasp/zap2docker-stable:2.12.0 zap-baseline.py -t http://localhost:3001
   ```

3. Compare the results of these scans, you will find that they're similar!

   > Note: for space saving reasons, we're showing the direct scan results... 

   ```
   WARN-NEW: Cross-Domain JavaScript Source File Inclusion [10017] x 10 
   	http://localhost:3000 (200 OK)
   	http://localhost:3000 (200 OK)
   	http://localhost:3000/ (200 OK)
   	http://localhost:3000/ (200 OK)
   	http://localhost:3000/juice-shop/build/routes/fileServer.js:16:13 (200 OK)
   WARN-NEW: Information Disclosure - Suspicious Comments [10027] x 2 
   	http://localhost:3000/main.js (200 OK)
   	http://localhost:3000/vendor.js (200 OK)
   WARN-NEW: Content Security Policy (CSP) Header Not Set [10038] x 12 
   	http://localhost:3000 (200 OK)
   	http://localhost:3000/ (200 OK)
   	http://localhost:3000/ftp (200 OK)
   	http://localhost:3000/ftp/coupons_2013.md.bak (403 Forbidden)
   	http://localhost:3000/ftp/eastere.gg (403 Forbidden)
   WARN-NEW: Storable and Cacheable Content [10049] x 11 
   	http://localhost:3000/robots.txt (200 OK)
   	http://localhost:3000 (200 OK)
   	http://localhost:3000/ (200 OK)
   	http://localhost:3000/assets/public/favicon_js.ico (200 OK)
   	http://localhost:3000/ftp/acquisitions.md (200 OK)
   WARN-NEW: Deprecated Feature Policy Header Set [10063] x 11 
   	http://localhost:3000 (200 OK)
   	http://localhost:3000/ (200 OK)
   	http://localhost:3000/ftp (200 OK)
   	http://localhost:3000/ftp/coupons_2013.md.bak (403 Forbidden)
   	http://localhost:3000/ftp/eastere.gg (403 Forbidden)
   WARN-NEW: Timestamp Disclosure - Uni
