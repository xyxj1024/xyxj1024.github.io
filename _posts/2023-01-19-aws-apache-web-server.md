---
layout:             post
title:              "Apache Web Server Setup Guide"
category:           "Web Applications and Cybersecurity"
tags:               web-development apache cse503-assignment
permalink:          /posts/apache-web-server-setup-guide
---

Individual portion of [Module 2](https://classes.engineering.wustl.edu/cse330/index.php/Module_2) from CSE 330/503S: "Rapid Prototype Development and Creative Programming" at Washington University in St. Louis. The Apache web server will be installed and configured on an AWS EC2 instance. A Mac laptop is used as the local machine.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Create an AWS EC2 Instance

Log in to your AWS Management Console and launch a new instance. Choose the image labeled "Amazon Linux 2 AMI" and make sure you have $$64$$-bit x86 selected. Make sure your security group includes a rule of type `SSH`, protocol `TCP`, port range `22`, and source of `0.0.0.0/0` or `Anywhere`.

In the dialog box, create a new key pair, enter some name for it (e.g., `sp2023`), and then click "Create & Download Your Key Pair". A `.pem` file will be downloaded. This is the private key for the default user (`ec2-user` on AMI). Save your key somewhere secure. Click "Launch Instances" and you are ready to launch your virtual server.

### SSH Configuration

Open a terminal window on your local computer (in my case a MacBook Air) and run the following command:

```bash
# Passphrase is optional
ssh-keygen
```

Your keys should have created in the directory `~/.ssh`:

```bash
$ ls ~/.ssh
id_rsa  id_rsa.pub
```

The first file stores your private key and the second stores the corresponding public key. If you are using macOS like me, run the following command to add your key to the SSH agent:

```bash
ssh-add
```

Then, you need to do the following to fix the permissions:

```bash
chgrp Users ~/.ssh/id_rsa

chmod 600 ~/.ssh/id_rsa
```

### Log in as the default user

To connect to the default user on your Amazon EC2 instance, run this command in your local terminal:

```bash
ssh -i </path/to/default-privatekey.pem> <ec2-user>@<ec2-xx-xx-xx-xx.compute-1.amazonaws.com>
```

Then you need to correctly set the permissions on your `.pem` file:

```bash
chmod 600 </path/to/default-privatekey.pem>
```

### Create a new user with SSH permissions

On your EC2 instance, run the following commands:

```bash
$ sudo useradd -r -m -c "<your full name>" <username>
$ sudo passwd <username>
Enter new UNIX password: 
Retype new UNIX password:
```

For security reasons, you should never SSH into your server as the root user. Instead, you should use a normal user to whom you give `sudo` privileges. To give a user `sudo` privileges, use the command `visudo`, which opens up the `sudo` configuration file in the system's default text editor. (Never edit the file `/etc/sudoers` directly!) `sudo` users are specified using lines similar to:

```text
<username>  ALL=(ALL)   ALL
```

Add that line immediately below the line defining `root`:

```text
root    ALL=(ALL)   ALL
```

When you are finished, save and close the file. Finally, restart the SSH server:

```bash
sudo service ssh restart || sudo service sshd restart
```

Okay, cool: you have created your custom username on your remote instance. Now, you need to enable this account to accept your SSH key when authenticating. To do this, you need to add your public key to a file named `authorized_keys`, which is stored in your remote username's SSH configuration directory. To create the directory and edit the file, run on your instance:

```bash
# Switch to your own user to ensure that permissions are set correctly:
su <username>

# Change the working directory to your user's home directory
cd

# Make the nonexistent SSH configuration directory and set its permissions:
mkdir .ssh
chmod 700 .ssh

# Create authorized_key file and set its permissions:
touch .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
```

Inside of the `authorized_keys` file, paste in your public key. Your public key is the content of `~/.ssh/id_rsa.pub` on your local machine. If you are using Mac OS X, run:

```bash
cat ~/.ssh/id_rsa.pub | pbcopy
```

At this stage, you can log in as your new user:

```bash
ssh <username>@<ec2-xx-xx-xx-xx.compute-1.amazonaws.com>
```

To save from typing this command in every time you want to access your server, you can set an alias in your terminal, e.g.:

```bash
alias aws='ssh <username>@<ec2-xx-xx-xx-xx.compute-1.amazonaws.com>'
```

## Set Up the Apache Web Server

### Install essential packages

The package management tool in Red Hat Enterprise Linux (RHEL), and therefore also your Amazon EC2 instance, is `rpm`. If you have an `rpm` package, you can install it by:

```bash
rpm -i <somepackage>.rpm
```

This requires that `<somepackage>.rpm` be in your current directory, which means you will have to download the file yourself (or create it). It requires you to manually install any dependencies the package has. A better alternative is to use a repository-based package manager. In RHEL, this is `yum`. Before you install new software, you need to ensure that your local list of available packages is up-to-date. Run the following commands to perform this operation:

```bash
yum check-update
```

After you have ensured that your package list is synced with the remote repository, you can start installing packages. To install a package, issue the following command:

```bash
yum install <somepackage>
```

To save you some headaches later on, it is recommended that you install a few essential package bundles when you first create a new Linux instance. These packages include things like Make and a C compiler.

```bash
sudo yum groupinstall "Development Tools"

sudo yum install kernel-devel kernel-headers
```

### Set the time zone

The time zone files are in the directory `/usr/share/zoneinfo`. They are further organized within subdirectories grouped by region. For instance, Rome's time zone file is stored within `/usr/share/zoneinfo/Europe`. In order to set the time zone, simply copy the desired time zone file to our `/etc` directory as a new file named `localtime`. For example, to set the the machine's system time to Rome's time zone, we would enter the command:

```bash
sudo cp /usr/share/zoneinfo/Europe/Rome /etc/localtime
```

### Install the Apache web server

In `yum`, Apache is distributed under the package name `httpd` (for "hypertext transfer protocol daemon"):

```bash
sudo yum update

sudo yum install httpd
```

You'll need to run this command to add Apache as a startup item:

```bash
sudo /sbin/chkconfig --levels 235 httpd on
```

In RHEL, most Apache configurations are stored in `/etc/httpd/conf/httpd.conf` and others are located in the directory `/etc/httpd/conf.d/`. At this point, Apache has been installed, but is not yet running. To start the web server, run this command:

```bash
sudo /usr/sbin/apachectl start
```

In order for your web server to be accessible, you need to open up Port 80 on your EC2 instance. (By default amazon blocks all traffic to our instance.) Go to the "Security Groups" under "Network & Security" on the AWS EC2 website. Select your security group and add a new `Custom TCP` rule with a port range of `80`. Leave the source at `0.0.0.0/0` (for all traffic). Click "Add Rule", and then click "Apply Rule Change".

To make sure things are working, create a file, like `hello.txt`, in your web server root. Give it some content (might I suggest "Hello, world!"). The web server root is at `/var/www/html`. You should now be able to visit your server load up the file using your web browser! Example Link: [http://ec2-xxx-xxx-xxx-xx.compute-1.amazonaws.com/hello.txt](http://ec2-xxx-xxx-xxx-xx.compute-1.amazonaws.com/hello.txt). Depending on the city in which your server is located, your link might look like: [http://ec2-xxx-xxx-xxx-xxx.us-west-2.compute.amazonaws.com/hello.txt](http://ec2-xxx-xxx-xxx-xxx.us-west-2.compute.amazonaws.com/hello.txt).

Finally, you need to edit the master Apache configuration file. Open `/etc/httpd/conf.d/userdir.conf` in your favorite text editor and find the line that says:

```text
UserDir disabled
```

Change it to:

```text
UserDir disabled root
```

Additionally, find the line that says:

```ini
#UserDir public_html
```

and uncomment it. This tells Apache that the directory containing each user's html files is a subdirectory of their home directory called `public_html`. You may also need to change the permissions of your user directory and your `public_html` directory to allow Apache to read and execute inside them. To do this, run the following commands:

```bash
mkdir /home/<username>/public_html

sudo chmod o+x /home/<username>

sudo chmod o+rx /home/<username>/public_html
```

Restart Apache. There are several different ways to restart Apache; they all functionally do (almost) the same thing, so choose your favorite:

```bash
sudo /usr/sbin/apachectl restart
sudo apachectl restart  # if /usr/sbin is in your PATH (which it is *not* by default in RHEL)
sudo /etc/init.d/httpd restart
sudo /sbin/service httpd restart
sudo service httpd restart
```

## PHP Installation and Workspace Setup

PHP is a server-side scripting language widely used in web development. PHP is what turns your website from a static file server into a dynamic web application. By default, Apache only knows how to serve static HTML pages to the client. You can write as many HTML pages as you want, which is fine for basic websites. But if you want to have users post to a forum, or process credit card transactions, or make your own online calendar, a server-side language like PHP is what you need. Here are some points to keep in mind:

- A PHP file on your server is not an HTML document. It is a PHP script.
- PHP generates documents. By default, it generates HTML documents.
- PHP can generate other document types by changing the MIME Type in the Content-Type HTTP Header.
- Apache runs the PHP interpreter on your PHP script before transmitting it to the browser. The content that is generated after the PHP interpreter runs is expected to match the MIME type: by default, an HTML document.
- The W3C Validator checks for valid markup in an HTML document. Everywhere that PHP generates an HTML document, that HTML document should be valid.

To install PHP 7.2, run the following command to install `php7.2` from the Amazon Linux Extras repository:

```bash
sudo amazon-linux-extras install -y php7.2
sudo service php-fpm start
```

Note that a later version of PHP might not be compatible with the underlying Amazon Linux distribution. Once you install the required packages, restart Apache for the changes to take effect. The PHP configuration file is called `php.ini`. In RHEL, it is located at `/etc/php.ini`. Open `php.ini` in your favorite text editor, find the `display_errors` option, and set it to `On`:

```ini
display_errors = On
```

You will need to restart Apache for any `php.ini` changes to take effect.

The PHP Extension and Application Repository (PEAR) is a package manager that enables you to install classes and/or extensions to PHP. It is available through both the `yum` and `apt` repositories under the package name `php-pear`:

```bash
# In RHEL
sudo yum install php-pear
```

Using PEAR to install PHP extensions is just as easy as `yum`, `apt`, `gem`, and `pip`. Just run `sudo pear install xxx`.

If you are using an external SFTP client, you can use FileZilla. Download and install FileZilla from [https://filezilla-project.org/download.php?type=client](https://filezilla-project.org/download.php?type=client). When you launch FileZilla, go to Edit->Settings or FileZilla->Preferences, and go to the SFTP options (under Connection). Click "Add key file...". Choose your `id_rsa` file. (**macOS Tip:** Press Shift+Command+Period to reveal hidden files in the file chooser window.) On FileZilla's main screen, there are four fields: `Host`, `Username`, `Password`, and `Port`. Fill them in as follws:

- `Host`: `sftp://<ec2-xxx-xx-xx-xxx.compute-1.amazonaws.com>`
- `Username`: The username you created on your server
- `Password`: You may leave this blank.
- `Port`: 22

Then click "Quickconnect". If everything is configured correctly, FileZilla should log into your server. On the left and right of the FileZilla window, you can drag files between your local computer and your server. Thus, you can edit a file in a text editor on your decktop, and then upload it to your server by simply dragging it from the left pane to the right pane in FileZilla.