---
layout: post
title:  "Expose a NAS Synology on the web"
date:   2015-01-04 11:38:00
categories: server
tags: server synology ssl
image: /assets/article_images/server-cover1.jpg
---
I installed a Synology NAS at home some time ago. It works perfectly, but now I would like to securely open some services on the web.

<!--more-->

# Step 1: DynDNS

In France, lots of ISP keeps providing dynamic IP ot their client. I already used services like [No-IP](http://no-ip.org) to access my NAS from the Internet, but I'm currently working on cleaning some paid services I don't really need.

I own a domain at [Gandi](http://gandi.net). The main advantage of this provider is the API allowing us to edit a domain from a simple script. I'll use this feature in order to create a DynDNS.

There are already [some](http://gzu.me/gandy-et-ip-dynamique/) [good](http://www.app3l.org/documents/dns-dynamique-gandi) articles explaining how to configure a DynDNS with Gandi based on [Chralu's work](https://github.com/Chralu/gandyn).
I made a fork of it in order to made some refactoring (PEP8 and so) and add Synology notifications. You can find it here [https://github.com/DayS/gandyn](https://github.com/DayS/gandyn).

##  Enable Gandi's API

Login to Gandi, then go to [API management page](https://www.gandi.net/admin/api_key).
You have to enable **OT&E first** which gives you a test API key. Then you'll be able to activate production API in order to get you key needed for the script.

## Installation

Follow theses commands to install the script on your NAS:

{% highlight bash %}
ssh -p custom_port root@mynas.com
cd /root
mkdir gandyn

# If you have Python 3 installed
wget https://github.com/DayS/gandyn/archive/mastr.zip
unzip master.zip
mv gandyn-master/src/* gandyn

# If you have Python 2.7 installed
wget https://github.com/DayS/gandyn/archive/python2.7.zip
unzip python2.7.zip
mv gandyn-python2.7/src/* gandyn
{% endhighlight %}

**Note :** Start from here, I'll use `python`in my examples as I have Python 2.7 installed on my NAS. I you have Python 3, you'll have to rplace this command by `python3`.

Now edit the config file to fit your needs : `vi gandyn/gandyn.fg`.

You can test everything works with : `python gandyn/gandyn.py -c gandyn/gandyn.cfg`.

## Cronjob

Synology's DSM provides an interface to manage cronjob. We'll use it in order to execute the script every hour.

* Navigate to *Control Panel* > *Task Scheduler*.
* Create a new *User-defined script* task
* In **general** tab, fill out the fields with the following values :
	* User : root
	* User-define script : `/usr/bin/python /root/gandyn/gandyn.py -c /root/gandyn/gandyn.cfg`
* In **Schedule** tab make the task run every run of every day

## Configure DynDNS notifications (optional)

> This part is based on [KZL's article](http://blog.gauss-it.net/2014/08/synology-du-dsm-au-terminal/) [FR].

I wanted to be notified via DSM's notifications center when the script makes an update on my domain.
To do so, Synology provides `/usr/syno/bin/synonotify` which takes a **tag_event**, and two important files:

* **/usr/syno/etc/notification/notification_filter.settings**
* **/usr/syno/etc/notification/mails**

In the first one, we can register and configure a custom notification tag. Edit the file and add the following line:

{% highlight text %}
GandyDDNSUpdate="mail,sms,mobile,cms"
{% endhighlight %}

In the second file, we can set the content of the mail sent by the NAS and the title of the notification.
Create or edit the file and add the following lines :

{% highlight text %}
[GandyDDNSUpdate]
Category: System
Title: Gandi DDNS Updated
Subject: DDNS Gandi updated

DDNS Gandi was successfully updated
{% endhighlight %}

**Note :** There is currently no way to inject variables in this template. Also an update of DSM may erase these two files.. But a [feature request](http://forum.synology.com/enu/viewtopic.php?f=3&t=92727) has been sent in order to manage customer notification from DSM's interface.


# Step 2: Configure a valid SSL certificate

> This part is based [Mike Tabor's article](https://miketabor.com/secure-synology-nas-install-ssl-certificate).

As I wanted to be sure I'm not a victim of *Man in the middle* attack while accessing my NAS from the Internet, I also installed a valid SSL certificate.

First, you have to purchase a SSL certificate. I bought a Comodo PositiveSSL via [NameCheap](https://www.namecheap.com/security/ssl-certificates/comodo/positivessl.aspx).

## Create the certificate signing request

Once this is done, navigate to *Control Panel* > *Security* > *Certificate* on your Synology and follow these instructions:

* Click on **Create certificate**
* Select **Create certificate signing request (CSR)**
* Fill out the form
* Download and un zip the generated archive.

You should now have these two files: **server.csr** and **server.key**. Carefully save these !

## Issue the SSL certificate

Go back to [NameCheap control panel](https://manage.www.namecheap.com/myaccount/ssl-list.asp) then :

* Click on **Issue** button next to your certificate.
* In the *Digital Certificate Order Form* page:
	* Select **Other** from the *Select Web Server* drop down menu
	* Paste the content of the **server.csr** file (previously saved) into the *Enter CSR* field
	* Click on **Next**

In the next screen, you have to select an email address on which you'll receive *a link* and a *validation code*.

A few minutes after you confirm the validation code, you should receive another mail with file attachment archive containing multiple files: *AddTrustExternalCARoot.crt*, *COMODORSAAddTrustCA.crt*, *COMODORSADomainValidationSecureServerCA.crt* and *my_domain.crt*.

## Import the SSL certificate

Before going further, we have to prepare a bundle intermediate certificate. If we import our certificate as is, some navigators will consider our website as untrusted, because the certification chain is broken.

You can test it by following instructions of [Mike Tabor's article](https://miketabor.com/secure-synology-nas-install-ssl-certificate) from *Import the SSL Certificate* and check you domain on [SSLShopper](https://www.sslshopper.com/ssl-checker.html).

Hopefully, Comodo [explains how to create a bundle file](https://support.comodo.com/index.php?/Default/Knowledgebase/Article/View/643/17/) from received CRT files.
The important thing here is to **respect the reverse order** (last intermediate certificate first, root certificate last).
In my example I had to type the following command : `cat COMODORSADomainValidationSecureServerCA.crt COMODORSAAddTrustCA.crt AddTrustExternalCARoot.crt > ssl-bundle.crt`.

Now, you can bo back to *Control Panel* > *Security* > *Certificate* on your Synology:

* Click on **Import certificate**
* Fill out the form with these values:
	* *Private key*: the **server.key** file you received in first step
	* *Certificate*: the **my_domain.crt** file received from Comodo archive
	* *Intermediate certificate*: the bundle you generated just before


# Step 3: Enjoy !

You now have your Synology securely exposed on the Internet behind a dynamic IP.
Of course, you still have to be careful :)