---
layout: post
title: Automatically Start Rails and Sidekiq After Ubuntu Server Reboot
category: devops rails sidekiq ubuntu cron
description: Tutorial how to start Rails Puma and Sidekiq workers automatically after a system reboot on Ubuntu servers using Crontab.
---

Deploying Rails project into cloud servers (like AWS EC2) is painless using Capistrano, what an amazing gem for automating the deployment process.

Run `cap production deploy`, within a few seconds your app will be deployed and run smoothly in production.

But, every time the server gets restarted manually or pumped up the server specs, we need to redeploy or use Capistrano to bootup puma and sidekiq manually, and that's not an ideal approach.

I found the proper way by making services on Unix to manage both puma and sidekiq or resque, tried it out, but it requires some work and time to set it up for every project.  and I didn't like it.

Till recently I found this trick on Crontab to trigger any command on system reboot.

### What do I need?

1. Grap start commands from Capistrano deploy logs (for puma, sidekiq or resque or any serivce required to bootup your app)
2. Combine those commands in a single line by joining them with `&&`.
3. Add this command in crontab as `@reboot` command

### The final command will look like
```bash
@reboot /bin/bash -l -c 'cd /var/www/html/my_app/current && source ~/.rvm/scripts/rvm && ~/.rvm/bin/rvm 2.6.2@app do bundle exec puma -C /var/www/html/my_app/shared/puma.rb --daemon && ~/.rvm/bin/rvm 2.6.2@app do bundle exec sidekiq --index 0 --pidfile /var/www/html/my_app/shared/tmp/pids/sidekiq-0.pid --environment dev --logfile /var/www/html/my_app/shared/log/sidekiq.log --daemon'
```

Change your app path and rvm gemset, and run `sudo crontab -e -u $USER` and add the line above in the crontab. and we are done.

Reboot your server and check if your app started automatically or not.

### Lets break it down

#### 1. Reboot Command

`@reboot /bin/bash -l -c '...'`

`@reboot` allow us to trigger any script or command when the operating system gets rebooted. and in this case, we are running bash script and put all the commands between the quotes.

and basically what I did, I grabbed the required commands from capistrano log after a successful deplyment, and made them in one line by joining them with `&&`.

#### 2. Navigate to project current folder

`cd /var/www/html/my_app/current`

#### 3. Refresh RVM

Just to make sure rvm exists with resent script
`source ~/.rvm/scripts/rvm`

#### 3. Puma start command

Since I'm using rvm, I defined my ruby version and the gemset `2.6.2@app`, and run puma on daemon mode using the shared puma file to load the settings, as in my case it is not the same in the repo. as this file is setup for the current server and the environment.

`~/.rvm/bin/rvm 2.6.2@app do bundle exec puma -C /var/www/html/my_app/shared/puma.rb --daemon`

Note: you can ignore the `-C /var/www/html/my_app/shared/puma.rb` if you want to use the default one in config folder.


#### 4. Sidekiq start command
Sidekiq command is little bit nasty, as have a lot of settings in the dame command.

`~/.rvm/bin/rvm 2.6.2@app do bundle exec sidekiq --index 0 --pidfile /var/www/html/my_app/shared/tmp/pids/sidekiq-0.pid --environment dev --logfile /var/www/html/my_app/shared/log/sidekiq.log --daemon`

here we did the same as puma with the rvm command, and run sidekiq (`~/.rvm/bin/rvm 2.6.2@app do bundle exec sidekiq`)
the extra arguments used to run sidekiq in daemon mode, define the pid file to use it when we restart or stop sidekiq, and defined the log path.

### Finally

Once you made your single command line, try to reboot the system first and run the command manually, and make sure its running.

Then you can add it in the Crontab, and triggre manuall reboot, jump to the browser to check if the script is working.

Cheers 🍻
