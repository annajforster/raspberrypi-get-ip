# raspberrypi-get-ip

Have you ever been annoyed that every time you reboot your Raspberry Pi (or take it to a different network) you have to guess its IP address so that you can ssh into it? No? Well, I have, and that's why I've created this tutorial.

Here, we'll set your Raspberry Pi up to send you an email with its IP address every time it boots using Gmail.

**Note:** If you feel like this is an overkill, you can always use `nmap`, for example:

```
$ nmap -sP 192.168.1.1/24
```

## Install and configure msmtp

This section is based on [this forum post](https://www.raspberrypi.org/forums/viewtopic.php?f=28&t=244147#p1496767).

```
$ sudo apt-get install msmtp msmtp-mta
```

Now open the config file and add your credentials.
This file can be found at `~/.msmtprc` or `$XDG_CONFIG_HOME/msmtp/config` if you're using version 1.8.6 or newer, or at `/etc/msmtprc` for older versions.

```
# Generics
defaults
auth           on
tls            on
# following is different from ssmtp:
tls_trust_file /etc/ssl/certs/ca-certificates.crt
# user specific log location, otherwise use /var/log/msmtp.log, however, 
# this will create an access violation if you are user pi, and have not changes the access rights
logfile        ~/.msmtp.log

# Gmail specifics
account        gmail
host           smtp.gmail.com
port           587

from          root@raspi-buster
user           your-gmail-accountname@gmail.com
password       your-gmail-account-password

# Default
account default : gmail
```

Test it:

```
$ echo "Hello world!" | msmtp you@example.com
```
```
n10871195@qut.edu.au
```

**Note:** You might bump into authentication problems on your first try. In my case, Gmail sent me an email alerting that a less secure application was trying to log into my account. I followed the links in the message and changed settings to allow ssmtp to log in. I also had to enable IMAP access on my Gmail settings.

## Create the email sending script

Create a file named `mail_ip.sh` (or whatever you prefer) and edit it adding your email address:

```
mailto="you@example.com"
ip=`ip route list | awk '{print NR,$(NF-2)}'`

{
	echo To: $mailto
       	echo "Subject: [RasPi] My IP"
	echo "$ip"
} | /usr/bin/msmtp $mailto
echo "$ip"
echo "Finished running at `date`"
```

Then, make sure it is executable and try to run it:

```
$ chmod +x mail_ip.sh
$ ./mail_ip.sh
```

If everything goes well, you should receive an email like this:

```
1 192.168.1.46
2 192.168.1.46
```

## Set your script to run on reboot

There are different ways to do this, we'll use `crontab` here.

```
$ crontab -e
```

Add the following line to the end of the file (make sure to use the absolute path to the script):

```
@reboot sleep 120 && /home/pi/mail_ip.sh > /home/pi/mail_ip.log 2>&1
```

**Note:** In my case, I had to add a 2-minute sleep before running it, so that ssmtp had time to start up. You might need a different delay, or none at all (it took me a while of trial and error to figure this out). Redirecting stdout and stderr to a log file is not necessary, but it helped me debug it.

You can also add `MAILTO=you@example.com` at the top of the crontab file to make sure you'll receive notifications when cron jobs fail.

## Try it out!

Either run `$ sudo reboot` or unplug and plug your Raspberry Pi. You should get an email after a couple of minutes.

