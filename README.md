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


