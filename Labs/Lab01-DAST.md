# Lab 01: Dynamic Application Security Testing (DAST) #

## Introduction ##
In this lab, you will learn how to automate a mature, open source penetration testing tool called ZAP (which stands for Zed Attack Proxy).  ZAP was created by The Open Web Application Security Project ([OWASP](https://owasp.org/)) circa 2010 and is a popular, powerful, and configurable DAST tool for scanning and attacking web applications.  ZAP can be easily incorporated into CD (continuous deployment/delivery) pipelines.  Note that a DAST tool like ZAP won't make sense in CI (continuous integration) automation since it must scan a deployed/running web application.  We highly encourage incorporating Static Application Security Testing (SAST) tools into your CI pipelines or even upstream via IDE-integrated tools as a way to "shift left" all the way, but SAST is beyond the scope of this lab.  Follow this guide to learn: ZAP basics, how to leverage "packaged scans" to automate ZAP for integration in a CD pipeline, and next steps for integrating into your pipeline tools of choice, whether GitHub Actions, GitLab, Jenkins, or otherwise.

## Learning Objectives ##
- Use of a deliberately vulnerable web app for testing
- Automate a leading DAST tool
  - Surface security alerts
  - Understand alert categories and severities
- How to include DAST in your pipelines

## Prerequisites ##
This lab requires Docker (Docker Desktop or otherwise) and 2 container images (Juice Shop and ZAP).  See the README at [this repository](https://github.com/andrewdouglas/CodeMash2023-AppSec) for instructions on obtaining the prerequisite container images.  If today is the precompiler session and you don't have the images, and would like to quickly download them via flash drive, ask Andy or Mick and we'll hook you up (these images are somewhat large and WIFI may not be adequate).  **If you downloaded the images from the flash drive**, you'll need to use 'docker load' vs. 'docker pull' like this:

In Mac/Linux:
```
docker load < juice-shop.tar
docker load < zap.tar
```
In Windows:
```
docker load -i juice-shop.tar
docker load -i zap.tar
```

## Prepare the Target ##
In order to dive into DAST with ZAP, we need to spend a few minutes deploying a web application locally that will serve as the target for our scans/attacks.  We will use the famous open-source OWASP Juice Shop web application as our target in this lab.  If you're not familiar with Juice Shop, here's the description from the [project site](https://owasp.org/www-project-juice-shop/):
> OWASP Juice Shop is probably the most modern and sophisticated insecure web application! It can be used in security trainings, awareness demos, CTFs and as a guinea pig for security tools! Juice Shop encompasses vulnerabilities from the entire [OWASP Top Ten](https://owasp.org/www-project-top-ten) along with many other security flaws found in real-world applications!

<font size="1">Disclaimer: before attacking/penetration-testing any real-world web application, be sure you have permission to do so.</font>

It's highly recommended that before attacking a web application, you understand the application's functionality, to ensure that the attacks add value in finding vulnerabilities or proving the application's defenses amidst well-executed attacks.  Let's get familiar with our target web application next.

## Run Juice Shop Locally ##
1. Run Juice Shop in a container by executing:
```
docker run --rm -p 3000:3000 bkimminich/juice-shop:v14.3.1
```
2. If the above commands worked, you'll see this at the bottom of the output:
> info: Server listening on port 3000

Congratulations! You're now running a containerized version of a very insecure application!  We'll use this application as our target for DAST scans and attacks.

## Explore the Juice Shop Application ##
1. Open a web browser to http://localhost:3000.  You should see the Juice Shop application.  If you do, skip to step 3.
2. If you're using Chrome and it redirects to https, follow these steps:
    - In your browser, enter this in the URL: chrome://net-internals/#hsts
    - Click **Domain Security Policy** in the left-hand navigation menu.
    - Under **Delete domain security policies** at the bottom of the screen, type localhost in the "Domain:" textbox, and click Delete.
    - Try to refresh your browser, or navigate again to http://localhost:3000.
3. Spend a few minutes to explore the running application, and optionally hack through some of the security challenges if you'd like.  Juice Shop gamifies the process of discovering vulnerabilities that were deliberately baked into it which adds an element of fun to hacking/learning.

## Brief Intro to ZAP ##
We'll use ZAP as our DAST tool for this session.  ZAP facilitates many different usage patterns including: manual/exploratory scans using a fat-client/desktop UI, headless automation via "daemon mode", and container-based pipeline-integrated scans which will be our focus.  The recommended approach for pipeline integration is to leverage ZAP's "Docker Packaged Scans", which include 3 scans: zap-baseline, zap-full-scan, and zap-api-scan, all of which are included in the owasp/zap2docker-stable container image you downloaded earlier.  Let's start exploring ZAP using the most basic, fastest scan, zap-baseline.  We'll explore zap-full-scan later.  We won't be using zap-api-scan in this guide, but if you have REST, GraphQL, or SOAP APIs you'd like to scan, zap-api-scan can help you (at the time of writing and as far as I'm aware, support for [gRPC](https://grpc.io/) APIs hasn't yet been incorporated into ZAP).  The following exercises will use these packaged scans.

## Scan Juice Shop ##
1. Ensure that Juice Shop is still running (you can execute command "docker ps" to see your running containers).
2. Open a separate command shell and execute the following command to initiate a scan using zap-baseline:

```
docker run --rm -t --network host owasp/zap2docker-stable:2.12.0 zap-baseline.py -t http://localhost:3000
```

Note: If you're running a Windows machine, and you see file permissions errors returned in the output, try adding "--user root" before the "--network" arg.  This is a known issue on Windows hosts which shouldn't affect most pipeline tools.

The above command should take around 30 seconds to complete, so while it's running, let's learn what it's doing.  The command runs 1 of the 3 packaged scans included with ZAP: zap-baseline.py.  The second -t argument specifies the target application as the URL where Juice Shop is running.  The zap-baseline scan, as with the other 2 packaged scans, wraps the ZAP tool, and in this case, facilitates automating a basic/"baseline" scan of an application.  Note that the baseline scan is unique from the other 2 packaged scans in that it is a "passive" scan; it executes a scan without conducting penetration attacks, and as such, is safe to use in Production environments per [ZAP documentation](https://www.zaproxy.org/docs/docker/baseline-scan/).  The baseline scan with default arguments as we used above will spider/crawl the target application for a maximum of 1 minute, so for a security scan, it's fast.

Review the console output when the command completes.  Toward the top, one important result is "Total of 12 URLs" which shows you the number of URLs found by the ZAP spider.  Toward the bottom of the output, you should see "WARN-NEW: 9" which means that this scan is alerting us to 9 distinct alerts.  ZAP alert levels include (in descending order of severity): FAIL, WARN, INFO, and PASS.  By default, ZAP displays all alerts as WARNs.  ZAP is designed to differentiate between new and existing vulnerabilities i.e. WARN-NEW vs. WARN-INPROG through the use of a "progress file" (-p argument), though that's also beyond the scope of this lab (but easy to configure).

When ZAP exits, it returns one of the following exit codes:
> 0 (success)
>
> 1 (at least 1 FAIL)
>
> 2 (0 FAILs and at least 1 WARN)
>
> 3 (any other failure)

To see the last exit code of your last command, execute:

From Powershell:
```
echo $LastExitCode
```

From Windows CMD:
```
echo %ERRORLEVEL%
```

From Linux/Mac shell:
```
echo $?
```

It should show:
> 2

The value is 2 because we had > 0 WARNs and 0 FAILs.  In a real-world scan of a web application, we may want the "Cross-Domain Misconfiguration" alert to produce a FAIL-level alert, and we may want to ignore the "Modern Web Application" alert.  To do this, we could ask ZAP to create a configuration file for the alerts with the -g argument, update these alerts to FAIL and IGNORE, and then re-scan with the -c argument specifying the path to the configuration file, but we'll skip that for this lab (though [this ZAP documentation](https://www.zaproxy.org/docs/docker/baseline-scan/#configuration-file) will help if you want to do this).

**NOTE: Exit codes are important when including scans in CD pipeline automation as any non-zero exit code will stop the pipeline.**  (So since our last scan produced a 2, it would stop a pipeline.)

### Optional ###
If you're curious about the ZAP image and want to poke around inside the ZAP container with an interactive terminal, you can execute:
```
docker run --rm -it --network host owasp/zap2docker-stable:2.12.0
```

## Explore ZAP CLI Arguments ##
1. Execute the command:
```
docker run --rm owasp/zap2docker-stable:2.12.0 zap-baseline.py -h
```
2. Note from the list of arguments the arguments -r, -w, -x, and -J for generating scan reports in various formats which will explore in the next set of steps.  By the way, ZAP arguments are common across the 3 packaged scans: zap-baseline.py, zap-api-scan.py, and zap-full-scan.py.

## Create a scan report ##
Let's generate an HTML scan report in a volume mapped to ZAP's working directory by executing:
```
docker run --rm -v $(pwd):/zap/wrk/:rw -t --network host owasp/zap2docker-stable:2.12.0 zap-baseline.py -t http://localhost:3000 -r testreport-baseline.html
```
NOTE: for Windows, replace $(pwd) with the directory to which you would like to save the HTML scan report e.g. C:\Users\YourUserName\.

Navigate to testreport-baseline.html and open it in a browser.  The report exposes much richer content surrounding scan alerts than the console output displays.  Notice that no references to FAIL, WARN, INFO, nor PASS appear.  Instead, we have Risk Levels of: High, Medium, Low, Informational, and False Positives.  The table at the top shows the grouped/aggregated counts of alerts falling into each Risk Level.  The table under it shows alerts grouped by the alert name, and the count of instances for each alert found.  HTML and Markdown report formats have the top summary table while XML and JSON only contain the alerts themselves and the number of instances (the bottom table only).  The console defaults to showing all alerts as WARN level vs. the report which associates alerts with the Risk Levels.

The counts shown in console output e.g. "WARN-NEW: Cross-Domain JavaScript Source File Inclusion [10017] x 4" match with the count of 4 alerts by the same name in the HTML report.  Most of the alerts' counts match between the console and HTML report, but the counts between the 2 formats are off, with the console showing 9 distinct alerts (9 WARN-NEW) vs. the HTML report showing 11 distinct alerts (sum of the Number of Alerts column in the top table).  This seems like a minor bug, due to the console grouping 3 of the HTML categories into 1 alert.  ("Non-Storable Content" appears with 10 instances in the console output as compared with 1 instance in the HTML report.  The console output seems to group "Storable and Cacheable Content" as well as "Storable but Non-Cacheable Content" into the same alert as "Non-Storable Content".  I find this to be a little confusing so I wanted to call it out here.)

Take a few minutes to explore the HTML report.  Notice that each alert comes with: a description, the URL that triggered the alert, and advice on the solution to resolve the alert.  OWASP ZAP documentation surrounding alerts can be found at this [OWASP reference page](https://www.zaproxy.org/docs/alerts/).  Based on that OWASP reference information, of the 11 alerts raised in our HTML report, 7 are tagged as being related to 2021 OWASP Top 10 vulnerabilities (e.g. the first alert "[Content Security Policy (CSP) Header Not Set](https://www.zaproxy.org/docs/alerts/10038/)" is tagged with OWASP_2021_A05 which corresponds to  [A05:2021 - Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)).  So the report is surfacing some nice information for us about the number of distinct issues, their risk levels, their correlation to OWASP Top 10 vulnerabilities, and recommendations on fixing the issues.  Not bad.

One of the alerts in the HTML report as well as the console output is "Modern Web Application".  The description for it shows:
>The application appears to be a modern web application. If you need to explore it automatically then the Ajax Spider may well be more effective than the standard one.

Let's add the Ajax spider to our scan command using the -j switch to see how the results differ (warning - expect a longer scan time):
```
docker run --rm -v $(pwd):/zap/wrk/:rw -t --network host owasp/zap2docker-stable:2.12.0 zap-baseline.py -t http://localhost:3000 -r testreport-baseline-w-ajax-spider.html -j
```
<font size="1">NOTE: for Windows, replace $(pwd) with the directory to which you would like to save the HTML scan report e.g. C:\Users\YourUserName\.</font>

Adding the -j argument significantly lengthens the duration of the baseline scan.  It overrides the default behavior to add the Ajax spider in addition to the traditional spider.  Documentation for the ZAP Ajax spider can be found [here](https://www.zaproxy.org/docs/desktop/addons/ajax-spider/).  In exchange for a greatly increased scan time, the scan results include discovery of 55 total URLs (43 more than our first scan!), and 13 total WARN-NEW alerts (4 more than the first scan with the traditional spider).  The Ajax scanner is designed to improve scan results for Javascript-based applications like single page applications where client-side routes are determined by Javascript code vs. static routes.  For scans of your own web apps, some consideration is required as to how you will balance scan speed against thoroughness of the scan.

## More about ZAP Alerts ##
Security alerts shown in the console and on HTML or other report formats are made by way of ZAP plugins.  Currently, ZAP plugins support nearly 200 distinct alerts spanning 4 severities: High, Medium, Low, Informational.  Many alerts are tagged e.g. with "OWASP Top 10".  See this documentation for all alerts supported by ZAP: https://www.zaproxy.org/docs/alerts/.

## Attack Juice Shop ##
We're going on the attack!  The following exercises incorporate zap-full-scan.py which adds active attacks to the passive scan functionality of zap-baseline.py.  This is not recommended for use in Production pipelines due to longer scan times and the possibility of impact to Production data.  Instead, if the more thorough scanning is desired for your application, the recommendation is to automate via a scheduled scan not related to a software pipeline.

Execute the following command to execute zap-full-scan.py packaged scan, showing the short version of console output (-s argument), and outputting an HTML report of the results (-r argument).  This scan may take between 3-5 minutes depending on your system specs.
```
docker run --rm -v $(pwd):/zap/wrk/:rw -t --network host owasp/zap2docker-stable:2.12.0 zap-full-scan.py -t http://localhost:3000 -r testreport-full.html -s
```
<font size="1">NOTE: for Windows, replace $(pwd) with the directory to which you would like to save the HTML scan report e.g. C:\Users\YourUserName\.</font>

<font size="1">NOTE 2: If this command is still running after about 10 minutes, it likely hung and should be terminated with Ctrl+C.</font>

With the lengthy scan times involved with zap-full-scan, you likely will **not** want to include this scan with an automated CD pipeline (use zap-baseline for CD pipelines).  Instead, the recommendation for zap-full-scan is to configure periodic scheduled scans; GitHub Actions, GitLab, Spinnaker, Jenkins, and other pipeline tools support cron-based jobs to facilitate lengthier periodic scans.

If your previous scan worked, it should show a small amount of console output including "WARN-NEW".  If you don't see "WARN-NEW" at the bottom of the output and you see pages of console output, it's likely that your scan errored.  The zap-full-scan.py packaged scan appears to be less reliable than zap-baseline.py which is discouraging, though the zap-baseline.py packaged scan may be sufficient to meet your needs.  (I had some success with running zap-full-scan.py while in Airplane Mode as I noticed issues logged in the output related to downloading things, but with the -j switch, and after a 45-minute+ scan, it produced results including 11 WARN-NEW issues via console output and an HTML report which you can find in this repository named zap-full-scan.html.)

## Move to the Pipeline (Beyond this Lab) ##
To put DAST into practice, we recommend incorporating ZAP into your CD pipelines.  OWASP ZAP makes this easy via the 3 Docker Packaged Scans,  2 of which we directly executued in this lab plus easy GitHub Actions [YAML-based integration](https://github.com/marketplace?query=owasp+zap).

Basic configuration to run zap-baseline.py is ([from here](https://github.com/zaproxy/action-baseline)):
```
steps:
  - name: ZAP Scan
    uses: zaproxy/action-baseline@v0.7.0
    with:
      target: 'https://www.zaproxy.org'
```

Advanced configuration:
```
on: [push]

jobs:
  zap_scan:
    runs-on: ubuntu-latest
    name: Scan the webapplication
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          ref: master
      - name: ZAP Scan
        uses: zaproxy/action-baseline@v0.7.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          docker_name: 'owasp/zap2docker-stable'
          target: 'https://www.zaproxy.org'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'
```

Each of the 3 Docker Packaged Scans (Baseline, Full, and API) have [GitHub Actions configurations](https://github.com/marketplace?query=owasp+zap) to support easy integration with your CD pipelines.  We highly recommend measuring the performance of your scans to ensure efficient operation before including ZAP into CD pipelines in order to strike the right balance for your organization between security coverage and deployment speed.  The zap-baseline scan is the fastest scan and is designed to be safe to execute in Production, so it's the best candidate for CD pipeline inclusion, though you may wish to experiment with zap-api-scan, and zap-full-scan as well.  With the basics of ZAP under your belt from this lab, and ZAP's comprehensive documentation, you will be able to quickly configure ZAP to meet your needs.

## Recap ##
In this lab, you completed the following learning objectives:
- Use of a deliberately vulnerable web app for testing ([link](#run-juice-shop-locally))
- Automate a leading DAST tool ([link](#scan-juice-shop))
  - Surface security alerts ([link](#create-a-scan-report))
  - Understand alert categories and severities ([link](#create-a-scan-report))
- How to include DAST in your pipelines ([link](#move-to-the-pipeline))

## Additional ZAP Resources ##
- OWASP ZAP Automation in CI/CD ([YouTube](https://www.youtube.com/watch?v=tR93F-llbo8)): ZAP founder Simon Bennets walks through ZAP intro, scan options like scripting, fuzzing, and authentication, and provides recommendations to maximize the value it provides
- ZAP Deep Dive: ZAP Automation ([YouTube](https://www.youtube.com/watch?v=B-MDsECikqM)): Similar to the above video, but perhaps more concise (~28 minutes including Q&A), ZAP founder Simon Bennets walks through automating ZAP including the 3 packaged scans
- ZAP project site: https://www.zaproxy.org/
- ZAP automation options: https://www.zaproxy.org/docs/automate/
