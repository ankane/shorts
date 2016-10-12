# Dokku on DigitalOcean

:droplet: Your very own PaaS

## Create Droplet

Create new droplet with Ubuntu 16.04. Be sure to use an SSH key.

## Install Dokku

```sh
wget https://raw.githubusercontent.com/dokku/dokku/v0.7.2/bootstrap.sh
sudo DOKKU_TAG=v0.7.2 bash bootstrap.sh
```

And visit your server’s IP address in your browser to complete installation.

If you have a domain, use virtualhost naming. Otherwise, Dokku will use different ports for each deploy of your app. You can add easily add a domain later.

## Set Up Server

Firewall

```sh
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

[Automatic updates](https://help.ubuntu.com/14.04/serverguide/automatic-updates.html)

```sh
sudo apt-get -y install unattended-upgrades
echo 'APT::Periodic::Unattended-Upgrade "1";' >> /etc/apt/apt.conf.d/10periodic
```

Swap

```sh
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo sh -c 'echo "/swapfile none swap sw 0 0" >> /etc/fstab'
```

Time zone

```sh
sudo dpkg-reconfigure tzdata
```

and select `None of the above`, then `UTC`.

## Deploy

Get the official Dokku client locally

```sh
git clone git@github.com:progrium/dokku.git ~/.dokku

# add the following to either your
# .bashrc, .bash_profile, or .profile file
alias dokku='$HOME/.dokku/contrib/dokku_client.sh'
```

Create app

```sh
dokku apps:create myapp
```

Add a `CHECKS` file

```txt
WAIT=2
ATTEMPTS=15
/
```

Deploy

```sh
git remote add dokku dokku@dokkuhost:myapp
git push dokku master
```

## Workers

Dokku only runs web processes by default. If you have workers or other process types, use:

```sh
dokku ps:scale worker=1
```

## One-Off Jobs

```sh
dokku run rake db:migrate
dokku run rails console
```

## Scheduled Jobs

Two options

1. Add a [custom clock process](https://devcenter.heroku.com/articles/scheduled-jobs-custom-clock-processes) to your Procfile

2. Or create `/etc/cron.d/myapp` with:

  ```
  * * * * * root /usr/local/bin/dokku run myapp rake task1
  0 0 * * * root /usr/local/bin/dokku run myapp rake task2
  ```

  If choosing this option, include a job to clean containers.

  ```
  0 * * * * root /user/local/bin/dokku cleanup
  ```

## Custom Domains

```sh
dokku domains:add www.datakick.org
```

## SSL

Get free SSL certificates thanks to [Let’s Encrypt](https://letsencrypt.org/). On the server, run:

```sh
dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git
dokku letsencrypt:cron-job --add
```

And locally, run:

```sh
dokku config:set --no-restart DOKKU_LETSENCRYPT_EMAIL=your@email.tld
dokku letsencrypt
```

## Logging

Use syslog to ship your logs to a service. [Papertrail](https://papertrailapp.com) is great and has a free plan.

For apps, use:

```sh
dokku plugin:install https://github.com/michaelshobbs/dokku-logspout.git
dokku plugin:install https://github.com/michaelshobbs/dokku-hostname.git
dokku logspout:server syslog+tls://logs.papertrailapp.com:12345
dokku logspout:start
```

For nginx, install [remote_syslog2](https://github.com/papertrail/remote_syslog2)

```sh
cd /tmp
wget https://github.com/papertrail/remote_syslog2/releases/download/v0.18/remote_syslog_linux_amd64.tar.gz
tar xzf ./remote_syslog*.tar.gz
cd remote_syslog
sudo cp ./remote_syslog /usr/local/bin
```

Create `/etc/log_files.yml` with:

```sh
files:
  - /var/log/nginx/*.log
destination:
  host: logs.papertrailapp.com
  port: 12345
  protocol: tls
```

And run:

```sh
remote_syslog
```

## TODO

- Database
- Memcached
- Redis
- [Monitoring](https://www.brianchristner.io/how-to-setup-docker-monitoring/)

## Bonus

Find great Docker projects at [Awesome Docker](https://github.com/veggiemonk/awesome-docker).

## Resources

- [Additional Recommended Steps for New Ubuntu 14.04 Servers](https://www.digitalocean.com/community/tutorials/additional-recommended-steps-for-new-ubuntu-14-04-servers)
