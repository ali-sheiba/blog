---
layout: post
title:  "Analyzing API usage from Nginx logs with GoAccess"
date:   2018-07-17 12:51:23 +0400
categories: devops rails apis
---

In the old days, I used to rely on cPanel Statistics or Google Analytics to get statistics like most visited pages, unique visitors, hits, client operating systems etc…, that works fine for web pages and blogs.

But now days usually I work on backend APIs that been consumed by mobile apps or static frontend apps, and using 3rd party monitoring service is not an option for all the projects, so we need to find an alternative.

## Meet [GoAccess][goaccess]

> [GoAccess][goaccess] is an open source real-time web log analyzer and interactive viewer that runs in a terminal on *nix systems or through your browser.

![goaccess-web][goaccess-image]

Cool, this tool can generate good looking detailed reports (based on the available data logs), and what I really admire is the ability to analyze different formats of logs like (Apache, Nginx, Amazon S3 ... etc)

And I'm going to demonstrate how I used it with Nginx access logs of Rails API Project to generate an HTML report every hour and access it through a web link.

## Lets get started


### 1. Install GoAccess

You can find the [installation guide][goaccess-install] available for multiple operating systems.

In my case, I have a server with **Ubuntu 16.04** that is hosting the backend APIs.

So SSH to the server, and execute:

```shell
# Add goaccess repository to our system
echo "deb http://deb.goaccess.io/ $(lsb_release -cs) main" | sudo tee -a /etc/apt/sources.list.d/goaccess.list
wget -O - https://deb.goaccess.io/gnugpg.key | sudo apt-key add -

# update the index
sudo apt-get update

# install GoAccess
sudo apt-get install goaccess
```

### 2. Generate first output

lets run the `goaccess` command


```shell
goaccess /var/www/html/my-app/shared/log/nginx.access.log -o /var/www/html/reports/api.html --log-format=COMBINED
```

first I defined the access logs file path of the app, the usually Nginx access log file is present in `/var/log/nginx/access.log` , but for our Rails App we defined a separate access log file.

then defined the output by `-o` followed with the full path of the file

last, set the log format into `--log-format=COMBINED` as GoAccess supports different formats, and `COMBINED` works for Nginx logs, for other formats you can check there [configs][goaccess-config].

### 3. Generate report by Crontab

There is an option to use `--real-time-html` which will use Web Socket Server to push the updates to the browser continuously, which is cool, but I don't want it since I need to maintain an extra background service running for it.

Instead, lets generate the report every hour using the crontab in ubuntu.

```shell
# Open Crontab config file
crontab -e
```

and add at the bottom:
```shell
0 * * * * goaccess /var/www/html/my-app/shared/log/nginx.access.log -o /var/www/html/reports/api.html --log-format=COMBINED
```

that's it, so now, the operating system will take care of it :)

### 4. Enabling and Securing Web Access to the Report

As you noticed, We generated the HTML report in the `reports` folder in the root path.

It will be available to the public as long as we have the default `default` config in `sites-enabled`.


But we don’t want our reports to available to the public, so let us restrict the access to the reports folder by prompting a username and password for the access.

```shell
# Set username and password into a file
# ( this command will encript the password by default )
sudo htpasswd -c /var/www/.htpasswd username
```
the command above will generate `.htpasswd` file that include a user(s) with encrypted passwords, then we need to point this file as an authentication list for the reports route

lets open `/etc/nginx/sites-availabe/default` and add

```
server {
  ...

  location /reports {
    auth_basic "Administrator Login";
    auth_basic_user_file /var/www/.htpasswd;
  }

  ...
}
```

now our reports folder is secured with basic authentication.

## Conclusion

I found GoAccess fairly enough at the beginning of new projects, where we have a limited budget or restricted by the client policy with 3rd party analytics services.

Useful to know more about the consumers, detect if there is an APIs used aggressively, planning for suitable maintenance time.

And for sure once the project grows we will need better and advanced monitoring and analyzing services or tools.

 [goaccess]: https://goaccess.io
 [goaccess-install]: https://goaccess.io/download#distro
 [goaccess-image]: https://camo.githubusercontent.com/59a421526a47493253705f2fb12d97b14521215a/68747470733a2f2f676f6163636573732e696f2f696d616765732f676f6163636573732d7265616c2d74696d652d68746d6c2d67682e706e673f3230313730333037303030303030
 [goaccess-config]: https://raw.githubusercontent.com/allinurl/goaccess/master/config/goaccess.conf