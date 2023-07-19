---
layout:             post
title:              "Benchmarking Apache and NGINX Performance"
category:           "Web Applications and Cybersecurity"
tags:               apache nginx cse503-assignment
permalink:          /blog/benchmarking-apache-and-nginx-performance
---

In this post, I would like to present performance measurements for Apache and NGINX web servers on the same free-tier AWS EC2 instance created [here]({{ site.baseurl }}/blog/apache-web-server-setup-guide). Experiments were conducted **inside the AWS EC2 instance** (rather than connected from my local machine through SSH) using the [Apache Bench (`ab`) command-line tool](https://httpd.apache.org/docs/2.4/programs/ab.html). For each run, I explicitly specified the number of HTTP GET requests made to the website I wrote for [CSE 503S Module 5](https://classes.engineering.wustl.edu/cse330/index.php?title=Module_5) and the level of concurrency (i.e., the number of multiple requests to perform at a time).

<br>
![ab-top](/assets/images/benchmarking-apache-nginx/ab-top.png){:class="img-responsive"}
<p style="text-align:center;color:gray;font-size:80%;"><em>
Figure 1: Output of the <code>top</code> command
</em></p><br>

<!-- excerpt-end -->

The Apache web server compile settings:

```console
$ apachectl -V
Server version: Apache/2.4.56 ()
Server built:   Mar 16 2023 14:38:03
Server's Module Magic Number: 20120211:126
Server loaded:  APR 1.7.2, APR-UTIL 1.6.3, PCRE 8.32 2012-11-30
Compiled using: APR 1.7.2, APR-UTIL 1.6.3, PCRE 8.32 2012-11-30
Architecture:   64-bit
Server MPM:     prefork
  threaded:     no
    forked:     yes (variable process count)
Server compiled with....
 -D APR_HAS_SENDFILE
 -D APR_HAS_MMAP
 -D APR_HAVE_IPV6 (IPv4-mapped addresses enabled)
 -D APR_USE_PROC_PTHREAD_SERIALIZE
 -D APR_USE_PTHREAD_SERIALIZE
 -D SINGLE_LISTEN_UNSERIALIZED_ACCEPT
 -D APR_HAS_OTHER_CHILD
 -D AP_HAVE_RELIABLE_PIPED_LOGS
 -D DYNAMIC_MODULE_LIMIT=256
 -D HTTPD_ROOT="/etc/httpd"
 -D SUEXEC_BIN="/usr/sbin/suexec"
 -D DEFAULT_PIDLOG="/run/httpd/httpd.pid"
 -D DEFAULT_SCOREBOARD="logs/apache_runtime_status"
 -D DEFAULT_ERRORLOG="logs/error_log"
 -D AP_TYPES_CONFIG_FILE="conf/mime.types"
 -D SERVER_CONFIG_FILE="conf/httpd.conf"
```

The NGINX web server compile settings:

```console
$ nginx -V
nginx version: nginx/1.22.1
built by gcc 7.3.1 20180712 (Red Hat 7.3.1-15) (GCC) 
built with OpenSSL 1.1.1g FIPS  21 Apr 2020
TLS SNI support enabled
configure arguments: --prefix=/usr/share/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --http-client-body-temp-path=/var/lib/nginx/tmp/client_body --http-proxy-temp-path=/var/lib/nginx/tmp/proxy --http-fastcgi-temp-path=/var/lib/nginx/tmp/fastcgi --http-uwsgi-temp-path=/var/lib/nginx/tmp/uwsgi --http-scgi-temp-path=/var/lib/nginx/tmp/scgi --pid-path=/run/nginx.pid --lock-path=/run/lock/subsys/nginx --user=nginx --group=nginx --with-compat --with-debug --with-file-aio --with-google_perftools_module --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_degradation_module --with-http_flv_module --with-http_geoip_module=dynamic --with-stream_geoip_module=dynamic --with-http_gunzip_module --with-http_gzip_static_module --with-http_image_filter_module=dynamic --with-http_mp4_module --with-http_perl_module=dynamic --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-http_xslt_module=dynamic --with-mail=dynamic --with-mail_ssl_module --with-pcre --with-pcre-jit --with-stream=dynamic --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-threads --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -m64 -mtune=generic' --with-ld-opt='-Wl,-z,relro -specs=/usr/lib/rpm/redhat/redhat-hardened-ld -Wl,-E'
```

Basic system information:

```console
$ uname -a
Linux ip-xxx-xx-xx-xxx.ec2.internal 5.10.177-158.645.amzn2.x86_64 #1 SMP Thu Apr 6 16:53:11 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
```

Hardware information:

```console
$ lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              1
On-line CPU(s) list: 0
Thread(s) per core:  1
Core(s) per socket:  1
Socket(s):           1
NUMA node(s):        1
Vendor ID:           GenuineIntel
CPU family:          6
Model:               63
Model name:          Intel(R) Xeon(R) CPU E5-2676 v3 @ 2.40GHz
Stepping:            2
CPU MHz:             2400.045
BogoMIPS:            4800.03
Hypervisor vendor:   Xen
Virtualization type: full
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            30720K
NUMA node0 CPU(s):   0
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx rdtscp lm constant_tsc rep_good nopl xtopology cpuid tsc_known_freq pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm cpuid_fault invpcid_single pti fsgsbase bmi1 avx2 smep bmi2 erms invpcid xsaveopt
```

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Install NGINX on the AWS EC2 Instance

I followed [this Stack Overflow discussion](https://stackoverflow.com/questions/57784287/how-to-install-nginx-on-aws-ec2-linux-2) to install the NGINX web server. However, after I ran the command `sudo systemctl status nginx`, I found the error message: `nginx.service failed because the control process exited`. To fix this situation, I ran the following two commands:

```console
$ sudo fuser -k 80/tcp
80/tcp:               2957  3067  3068  3069  3070  3071
Could not kill process 3067: No such process
Could not kill process 3068: No such process
Could not kill process 3069: No such process
Could not kill process 3070: No such process
Could not kill process 3071: No such process
$ sudo fuser -k 443/tcp
```

Subsequently, I restarted the server using `sudo systemctl restart nginx` and checked the server status again:

```console
  Active: active (running) since Sat 2023-04-22 14:01:20 UTC; 13s ago
 Process: 4063 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
 Process: 4059 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
 Process: 4058 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
Main PID: 4065 (nginx)
```

The HTML shown below is the output of the cURL command:

```bash
curl -X GET http://ec2-x-xx-xxx-x.compute-1.amazonaws.com/
```

with which I attempted to send a HTTP GET request to the public DNS hostname of my AWS EC2 instance:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Collect Performance Measurements

My experiments proceed as follows:
- For each of the two web servers, I will run seven `ab` commands taking the form of `ab -n <num> -c 100 http://ec2-x-xx-xxx-x.compute-1.amazonaws.com/module5-group/calendar.html`, with `<num>` in $$\{ 1000, 5000, 10000, 15000, 20000, 25000, 30000 \}$$. The `module5-group` directory should be present in `/var/www/html` (the root directory of Apache) and `/usr/share/nginx/html` (the root directory of NGINX).
- Before each run, I will restart the web server.
- After each run, I will write down three major metrics produced by the `ab` command: time taken for tests measured in seconds, total number of bytes received from the server, and total number of requests per second (`ReqPerSec`). I will divide the total number of bytes received from the server by the total time taken to obtain bytes per second (`BytesPerSec`).

In the following charts, the red line connects the data points for Apache while the blue line connects the data points for NGINX.

![apache-nginx-time](/assets/images/benchmarking-apache-nginx/apache-nginx-time.png){:class="img-responsive"}
<p style="text-align:center;color:gray;font-size:80%;"><em>
Figure 2: Seconds taken for tests
</em></p>

![apache-nginx-mb-per-sec](/assets/images/benchmarking-apache-nginx/apache-nginx-mb-per-sec.png){:class="img-responsive"}
<p style="text-align:center;color:gray;font-size:80%;"><em>
Figure 3: Megabytes per second
</em></p>

![apache-nginx-req-per-sec](/assets/images/benchmarking-apache-nginx/apache-nginx-req-per-sec.png){:class="img-responsive"}
<p style="text-align:center;color:gray;font-size:80%;"><em>
Figure 4: Requests per second
</em></p>

These numbers indicate that, under a concurrency level of $$100$$, NGINX performs better than Apache on the AWS EC2 `t2-micro` instance that has single virtual CPU.

## Apache Prefork and Worker Multi-Processing Modules (MPMs) Comparison

In the above experiments, I was actually using a [prefork](https://httpd.apache.org/docs/2.4/mod/prefork.html) Apache web server, in which a pool of worker processes was created in advance, each ready and willing to accept one new HTTP connection. What if I switch to a [worker](https://httpd.apache.org/docs/2.4/mod/worker.html) Apache?

```console
Server MPM:     worker
  threaded:     yes (fixed thread count)
    forked:     yes (variable process count)
```

The Apache's [official document for MPMs](https://httpd.apache.org/docs/2.4/mpm.html) states that:

> The server can be better customized for the needs of the particular site. For example, sites that need a great deal of scalability can choose to use a threaded MPM like `worker` or `event`, while sites requiring stability or compatibility with older software can use a `prefork`.

According to [this NGINX's blog post](https://www.nginx.com/blog/nginx-vs-apache-our-view/):

> When a prefork Apache web server received an HTTP connection, one of the `httpd` worker processes grabbed and handled it. Each process handled one connection at a time, and if all of the processes were busy, Apache created more worker processes to be ready for a further spike in traffic. . . . The worker MPM replaced separate `httpd` processes with a small number of child processes that ran multiple worker threads and assigned one thread per connection. This was helpful on many commercial versions of Unix (such as IBM's AIX) where threads are much lighter weight than processes, but is less effective on Linux where threads and processes are just different incarnations of the same operating system entity.

Under my experimental settings, a worker Apache fails to match the performance of a prefork Apache:

![worker-mpm-stats](/assets/images/benchmarking-apache-nginx/worker-mpm-stats.png){:class="img-responsive"}
<p style="text-align:center;color:gray;font-size:80%;"><em>
Figure 5: Performance measurements for a worker Apache
</em></p>