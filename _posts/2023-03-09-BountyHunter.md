---
title: BountyHunter
date: 2023-03-09
categories: [HackTheBox, Linux]
tags: [http, burp, xxe, ssh, python, command-injection]
img_path: /assets/img/BountyHunter/
---



This write-up is about the machine BountyHunter from Hack The Box. This machine involves some XXE injection to find the initial credentials to gain access. Then some misconfigured python commands that allow us to our user to root.



## <u>Recon</u>

As always the first step of any box is nmap to find the open ports. 

![Nmap scan](nmap.PNG)

From this scan we can see that the machine is SSH (port 22) and HTTP (port 80). Since there is http running we'll first take a look at the website.

![Bounty Hunter Website](website.PNG)

From a first look over the website we can see a simple web page that has a download button for a pricing guide and a contact form. In the drop down menu in the top right we can also see an option for a portal. Before going through anymore of the website I will run a Gobuster scan to see if there are any extra directories to get information from.

![Gobuster scan](gobuster.PNG)

After going through the extra directories I found the `db.php` the most interesting as all it showed was a blank page so there may be so php code running in the background. I will remember this directory for potential use later. 

My next step from here is to take a look at the portal page that was shown on the home page. 

![portal page](portal.PNG)

Looks like the portal is still under development so it won't be of use to us. However, there is a link to check out the bounty tracker. So we will check that out next.

The bounty tracker contains a simple fillable form that has options for Exploit Title, CWE, CVSS Score and Bounty Reward. When this form is filled out and submitted we receive a message that tells us that the DB isn't ready and what would have been added. 

![Bounty Tracker First Look](bountytracker.PNG)

This seems normal at first however as we saw earlier there was a `db.php` directory. So if that db is not for the bounty tracker then what could it contain. 



## <u>Initial Access</u>

My next step is going to be capturing the submission of the bounty form in burp suite as it is being sent. 

From our intercepted POST request we can see the data being sent is encoded. 

![Intercepted POST Request](burpcapture.PNG)

To know what is being sent is this POST request we will first have to decode the data being sent. Upon first look we can see that the data has been url encoded by the "%3D". After performing the first level decoding we can see that the data is still encoded. At the end of decoded data we can "=" at the end which tells us that the data had also been base64-encoded. Finally, after performing this second set of decoding we can finally see that plain data being sent.

![Decoded data](decoded.PNG)

With this newly decoded data we can see that the form is using XML (Extensible Markup Language) so we are going to perform an XXE (XML External Entity) attack. To find the syntax to perform this attack I can use the example from [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XXE%20Injection). 

For my first test to see if the website is vulnerable to the XXE attack is by trying to grab the /etc/passwd file. 

```
<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE file [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
        <bugreport>
        <title>&xxe;</title>
        <cwe>2</cwe>
        <cvss>3</cvss>
        <reward>4</reward>
        </bugreport>
```

Once I have crafted my attack I can now use a tool like [CyberChef](https://gchq.github.io/CyberChef/#recipe=To_Base64('A-Za-z0-9%2B/%3D')&input=PD94bWwgIHZlcnNpb249IjEuMCIgZW5jb2Rpbmc9IklTTy04ODU5LTEiPz4KPCFET0NUWVBFIGZpbGUgWyA8IUVOVElUWSB4eGUgU1lTVEVNICJmaWxlOi8vL2V0Yy9wYXNzd2QiPiBdPgogICAgICAgIDxidWdyZXBvcnQ%2BCiAgICAgICAgPHRpdGxlPiZmaWxlOzwvdGl0bGU%2BCiAgICAgICAgPGN3ZT4yPC9jd2U%2BCiAgICAgICAgPGN2c3M%2BMzwvY3Zzcz4KICAgICAgICA8cmV3YXJkPjQ8L3Jld2FyZD4KICAgICAgICA8L2J1Z3JlcG9ydD4) to encode it in base64 and then encode it in url.

![First XXE encoded](firstencoded.PNG)

Now all that is left is to send the attack. To do this, in burp we will send our initial intercepted request to the repeater. Next we will change the data being sent with our own encoded attack and send it off. 

![Passwd XXE attack](xxePasswd.PNG)

Looks like we were successful!

Next we will try to use this attack to try to view the `db.php`. As we are going to be trying to read a php file we will need to make an adjustment to our command. Instead of using `file://` we will need to use `php://filter/convert.base64-encode/resource=`. So our next XXE will look a little like this,

```
<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE file [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=db.php"> ]>
        <bugreport> 
        <title>&xxe;</title>
        <cwe>2</cwe>
        <cvss>3</cvss>
        <reward>4</reward>
        </bugreport>
```

After crafting the XXE once again we need to encode it in base64 and then again in url. After encoding we can send it through the burp repeater and see what we get back. 

![db file decoded](dbXXE.PNG)

Looks like the `db.php` contained a to-do list with some logins. As users often like to reuse credentials we can potentially use this information to make an attempt to login in to the SSH. Using the password `m19RoAU0hP41A1sTsq6K`that we found, I first tried the usernames contained within the same file but had no luck. The next step to try was the users listed in `/etc/passwd` that were allowed to login. One of the users that can login is `development` which upon connecting to SSH with works!

![ssh login](development.PNG)

Upon listing the directory we can see the user flag waiting for us.

![User Flag](userflag.png)



## <u>Privilege Escalation</u>

 

