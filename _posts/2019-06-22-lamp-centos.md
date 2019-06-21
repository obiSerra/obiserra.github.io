---
layout: post
title: "Centos LAMP setup"
date:   2019-06-22 00:27:22 +0200
categories: LAMP devOps PHP Apache Centos
---

## Start

Once you are logged into the machine via ssh, you may want to check the OS you are working with, using the command

    $ cat /etc/*-release

The shell output should be something like

    CentOS release 6.3 (Final)

Now let’s start installing the rest of the stack

## Apache

To install Apache, just run

    $ sudo yum install httpd

and type y when the system ask for confirm.

Once done, you should be able to check that apache is installed by typing

    $ which httpd

and getting something like

    /usr/sbin/httpd

as response

Now to start Apache just type

    $ sudo service httpd start

or to restart it after some config updates

    $ sudo service httpd restart


## MySQL

To install MySQL type

    $ sudo yum install mysql-server

if you are working on a Centos 7 machine, note that MySQL is not listed inside the yum repository; you simply have to follow this guide [Install mysql on Centos 7](https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-centos-7)

Once installed MySQL can be started with


    $ sudo service mysqld start



And then setup the mysql password for the root user and other security config (you should provide a strong root password and answer Y for all the remaining question)

    $ sudo /usr/bin/mysql_secure_installation

## PHP

Run this command to install PHP

    $ sudo yum install php php-mysql


## Config iptables

You will probably need to update iptables, the Centos firewall, adding a new rule to accept incoming TCP request on port 80 (the 2 is needed to add the rule before the default last one that reject all other requests, making the http rule unreachable)

    $ iptables -I INPUT 2 -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT

_Note: if you are using Amazon Aws, you may want to update the instance security configuration, that by default allow only ssh requests)_

Check everything works

Open the config file, that should be

    /etc/httpd/conf/httpd.conf

and look for the line starting with DocumentRoot, it should be something like

    DocumentRoot "/var/www/html"

Now all you need to do is to upload or create a file inside the document root

    $ echo "<?php phpinfo();" > info.php

and you can verify it by navigating with your browser to

http://[your-public-ip]/info.php

_Note: Once you have done, remember to delete the info.php page, because it can give some useful information to malicious user that may want to hack your site._

