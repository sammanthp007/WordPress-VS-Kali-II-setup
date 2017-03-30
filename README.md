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
- [ ] Opening an Attack Surface
1. In the WP admin console, go to the plugins page
2. Search for `reflex gallery` and you should see **Reflex Gallery >> WordPress
Photo Gallery**
3. Click on the result *don't install the plugin yet*. Look at the `Changelog`
tab.

Looks like 3.1.4 fixed a critical security issue (those are the best kind), so 
we should use 3.1.3. But you cannot use the convenient install button, because 
WP only lets you install the latest. You'll need to install it manually.

1. On the right side of the dialog, where it lists the compatibility and installs 
data, click the WordPress.org Plugin Page » link
2. You'll be taken to the official WP plugin page. Click on the Developers tab
Under Other Versions, download the 3.1.3 zip file
3. Go to the manual plugin upload page and upload the zip
4. Go back to the plugins page, find the plugin, and click Activate

**Challenge** : Beyond activating the plugin, you need to use it in a page or a post on the WP instance. Create a gallery and use it in a page before proceeding
