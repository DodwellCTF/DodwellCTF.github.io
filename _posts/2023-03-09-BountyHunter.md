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

From a first look over the website we can simple page that has a download for a pricing guide and a contact form. In the drop down menu in the top right we can also see an option for a portal. Before going through anymore of the website I will run a gobuster scan to see if there are any extra directories to get information from.

![Gobuster scan](gobuster.PNG)

After going through the extra directories I found the 'db.php' the most interesting as all it showed was a blank page so there may be so php code running in the background. I will remember this directory for potential use later. 

My next step from here is to take a look at the portal page that was shown on the home page. 

![portal page](portal.PNG)

Looks like the portal is still under development so it won't be of use to us. However, they have a link to check out the bounty tracker.