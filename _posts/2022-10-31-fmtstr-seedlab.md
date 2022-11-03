---
layout:       post
title:        "SEED Labs 2.0: Format String Attack Lab Writeup"
category:     "Computing Systems"
tags:         system-security docker reverse-shell
permalink:    /fmtstr-seedlab/
---

For general overview and the setup package for this lab, please go to [SEED Labs official website](https://seedsecuritylabs.org/Labs_20.04/Files/Format_String/). The lab assignment was conducted using SEED virtual machine configured on a AWS EC2 instance along with containers. The set-up instructions for Docker Compose can be found [here](https://github.com/seed-labs/seed-labs/blob/master/manuals/docker/SEEDManual-Container.md).

The **Format Function**{: style="color: red"} is an ANSI C conversion function, like <code>printf</code>, <code>fprintf</code>, which converts a primitive variable of the programming language into a human-readable string representation. The **Format String**{: style="color: red"} is the argument of the Format Function and is an ASCII Z string which contains text and format parameters, like:
```c
printf("The magic number is: %d\n", 1911);
```
The **Format String Parameter**{: style="color: red"}, like <code>%x</code> or <code>%s</code>, defines the type of conversion of the format function[^1].

<!-- excerpt-end -->

|---
| | **Buffer Overflow** | **Format String**
|:-|:-|:-
| **Public since** | mid 1980's | June 1999
| **Danger realized** | 1990's | June 2000
| **Number of exploits** | a few thousands | a few dozen
| **Considered as** | security threat | programming bug
| **Techniques** | evolved and advanced | basic techniques
| **Visibility** | sometimes very difficult to spot | easy to find

*Table 1: Buffer Overflow vs. Format String Vulnerabilities, from [TESO Group's 2001 paper](https://cs155.stanford.edu/papers/formatstring-1.2.pdf)*[^2]
{:.table-caption}

<br />
## Table of Contents
{:.no_toc}
* TOC 
{:toc}
<br />

## Warm-up

Consider the following vulnerable C program:
```c
/***** fmtstr.c *****/
#include <stdio.h>

int main(int argc, char **argv)
{
    char buf[200];
    int val = 1;
    int buflen = 0;
    int strlen = 0;
    
    printf("buf is at: %p.\n", buf);
    printf("val is at: %p.\n", &val);
    if (argc != 2) {
        printf("usage: %s [user string]\n", argv[0]);
        return 1;
    }
    
    buflen = sizeof buf;
    strlen = snprintf(buf, buflen, argv[1]);
    if (strlen < 0 || strlen >= buflen) {
        printf("User string is not completely written!\n");
    }
    printf("buffer is %s.\n", buf);
    printf("val is %d/%#x (@ %p).\n", val, val, &val);
    return 0;
}
```

I received the following warning message after compilation:
```console
$ touch fmtstr.c
$ vim fmtstr.c
$ gcc -m32 fmtstr.c -o fmtstr
fmtstr.c: In function ‘main’:
fmtstr.c:18:5: warning: format not a string literal and no format arguments [-Wformat-security]
   18 |     strlen = snprintf(buf, buflen, argv[1]);
      |     ^~~~~~
```
which states that the third argument of <code>snprintf()</code> is not recognized by the format function as an appropriate ASCII Z string with format arguments and thus the program is vulnerable to malicious user string input. The safe way of using the format function is:
```c
snprintf(buf, sizeof buf, "%s", argv[1]);
```

Run the program a few times and then **turn off ASLR** to simplify the tasks:
<p style="color:gray; font-size:80%;">
ASLR makes guessing the exact addresses difficult; guessing addresses is one of the critical steps of the format-string attack.
</p>
```console
$ ./fmtstr
buf is at: 0xffd142d4.
val is at: 0xffd142c8.
usage: ./fmtstr [user string]
$ ./fmtstr AAAA
buf is at: 0xff96e864.
val is at: 0xff96e858.
buffer is AAAA.
val is 1/0x1 (@ 0xff96e858).
$ ./fmtstr %8$llx
buf is at: 0xff8f8dc4.
val is at: 0xff8f8db8.
User string is not completely written!
buffer is .
val is 1/0x1 (@ 0xff8f8db8).
$ echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
0
```

Try to construct an input that triggers a segmentation fault:
```console
$ ./fmtstr $(perl -e 'print "A"x300')
buf is at: 0xffffd444.
val is at: 0xffffd438.
User string is not completely written!
buffer is AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.
val is 1/0x1 (@ 0xffffd438).
$ ./fmtstr %s%s%s%s
buf is at: 0xffffd564.
val is at: 0xffffd558.
Segmentation fault (core dumped)
```
For the first case, 199 <code>A</code>'s were printed out.

Try to find the beginning of my input string on the stack:
```console
$ ./fmtstr ABCD--%x--%x--%x--%x--%x--%x--%x--%x--%x--%x--%x
buf is at: 0xffffd544.
val is at: 0xffffd538.
buffer is ABCD--5655624c--0--100--40--ffffd6d4--40--200--1--c8--0--44434241.
val is 1/0x1 (@ 0xffffd538).
```

Replace the final <code>%x</code> first with <code>%hx</code>, then with <code>%hhx</code>:
```console
$ ./fmtstr ABCD--%x--%x--%x--%x--%x--%x--%x--%x--%x--%x--%hx
buf is at: 0xffffd544.
val is at: 0xffffd538.
buffer is ABCD--5655624c--0--100--40--ffffd6d4--40--200--1--c8--0--4241.
val is 1/0x1 (@ 0xffffd538).
$ ./fmtstr ABCD--%x--%x--%x--%x--%x--%x--%x--%x--%x--%x--%hhx
buf is at: 0xffffd544.
val is at: 0xffffd538.
buffer is ABCD--5655624c--0--100--40--ffffd6d4--40--200--1--c8--0--41.
val is 1/0x1 (@ 0xffffd538).
```
The four characters "<code>ABCD</code>" is stored on the stack in an order that "<code>D</code>" is at the highest address and "<code>A</code>" is at the lowest address. With "<code>%x</code>", four bytes of content are printed out at the start address of the input string, which should be read from right to left; with "<code>%hx</code>", only the lowest two bytes of content are printed out; with "<code>%hhx</code>", only the lowest one byte of content is printed out.

## Environment Setup

### The <code>format.c</code> Program

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/ip.h>

/* Changing this size will change the layout of the stack.
 * Instructors can change this value each year, so students
 * won't be able to use the solutions from the past.
 * Suggested value: between 10 and 400  */
#ifndef BUF_SIZE
#define BUF_SIZE 100
#endif


#if __x86_64__
  unsigned long target = 0x1122334455667788;
#else
  unsigned int  target = 0x11223344;
#endif 

char *secret = "A secret message\n";

void dummy_function(char *str);

void myprintf(char *msg)
{
#if __x86_64__
    unsigned long int *framep;
    // Save the rbp value into framep
    asm("movq %%rbp, %0" : "=r" (framep));
    printf("Frame Pointer (inside myprintf):      0x%.16lx\n", (unsigned long) framep);
    printf("The target variable's value (before): 0x%.16lx\n", target);
#else
    unsigned int *framep;
    // Save the ebp value into framep
    asm("movl %%ebp, %0" : "=r"(framep));
    printf("Frame Pointer (inside myprintf):      0x%.8x\n", (unsigned int) framep);
    printf("The target variable's value (before): 0x%.8x\n",   target);
#endif

    // This line has a format-string vulnerability
    printf(msg);

#if __x86_64__
    printf("The target variable's value (after):  0x%.16lx\n", target);
#else
    printf("The target variable's value (after):  0x%.8x\n",   target);
#endif

}


int main(int argc, char **argv)
{
    char buf[1500];


#if __x86_64__
    printf("The input buffer's address:    0x%.16lx\n", (unsigned long) buf);
    printf("The secret message's address:  0x%.16lx\n", (unsigned long) secret);
    printf("The target variable's address: 0x%.16lx\n", (unsigned long) &target);
#else
    printf("The input buffer's address:    0x%.8x\n",   (unsigned int)  buf);
    printf("The secret message's address:  0x%.8x\n",   (unsigned int)  secret);
    printf("The target variable's address: 0x%.8x\n",   (unsigned int)  &target);
#endif

    printf("Waiting for user input ......\n"); 
    int length = fread(buf, sizeof(char), 1500, stdin);
    printf("Received %d bytes.\n", length);

    dummy_function(buf);
    printf("(^_^)(^_^)  Returned properly (^_^)(^_^)\n");

    return 1;
}

// This function is used to insert a stack frame between main and myprintf.
// The size of the frame can be adjusted at the compilation time. 
// The function itself does not do anything.
void dummy_function(char *str)
{
    char dummy_buffer[BUF_SIZE];
    memset(dummy_buffer, 0, BUF_SIZE);

    myprintf(str);
}
```

The vulnerable program <code>format.c</code> used in this lab can be found in the <code>server-code</code> folder, which reads data from the standard input, and then passes the data to <code>myprintf()</code> to print out the data. The program will run on a server with the root privilege, and its standard input will be redirected to a TCP connection between the server and a remote user. Therefore, the program actually gets its data from a remote user. If users can exploit this vulnerability, they can cause damages.

```console
$ make
gcc -o server server.c
gcc -DBUF_SIZE=100 -z execstack  -static -m32 -o format-32 format.c
format.c: In function ‘myprintf’:
format.c:44:5: warning: format not a string literal and no format arguments [-Wformat-security]
   44 |     printf(msg);
      |     ^~~~~~
gcc -DBUF_SIZE=100 -z execstack  -o format-64 format.c
format.c: In function ‘myprintf’:
format.c:44:5: warning: format not a string literal and no format arguments [-Wformat-security]
   44 |     printf(msg);
      |     ^~~~~~
$ make install
cp server ../fmt-containers
cp format-* ../fmt-containers
```

The warning message is generated by a countermeasure implemented by the <code>gcc</code> compiler against format string vulnerabilities. It should be noted that the program needs to be compiled using the "<code>-z execstack</code>" option, which allows the stack to be executable. Our ultimate goal is to inject code into the server program's stack, and then trigger the code.

### The <code>server.c</code> Program

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <time.h>
#include <sys/socket.h>
#include <netinet/ip.h>
#include <arpa/inet.h>
#include <signal.h>
#include <sys/wait.h>

#define PROGRAM "format"
#define PORT    9090

int socket_bind(int port);
int server_accept(int listen_fd, struct sockaddr_in *client);
char **generate_random_env();

void main()
{
    int listen_fd;
    struct sockaddr_in  client;

    // Generate a random number
    srand (time(NULL));
    int random_n = rand()%2000; 
   
    // handle signal from child processes
    signal(SIGCHLD, SIG_IGN);

    listen_fd = socket_bind(PORT);
    while (1){
        int socket_fd = server_accept(listen_fd, &client);

        if (socket_fd < 0) {
            perror("Accept failed");
            exit(EXIT_FAILURE);
        }

        int pid = fork();
        if (pid == 0) {
            // Redirect STDIN to this connection, so it can take input from user
            dup2(socket_fd, STDIN_FILENO);

            /* Uncomment the following if we want to send the output back to user.
             * This is useful for remote attacks. 
            int output_fd = socket(AF_INET, SOCK_STREAM, 0);
            client.sin_port = htons(9091);
            if (!connect(output_fd, (struct sockaddr *)&client, sizeof(struct sockaddr_in))){
               // If the connection is made, redirect the STDOUT to this connection
               dup2(output_fd, STDOUT_FILENO);
            }
            */ 

            // Invoke the program 
            fprintf(stderr, "Starting %s\n", PROGRAM);
            //execl(PROGRAM, PROGRAM, (char *)NULL);
            // Using the following to pass an empty environment variable array
            //execle(PROGRAM, PROGRAM, (char *)NULL, NULL);
            
            // Using the following to pass a randomly generated environment varraible array.
            // This is useful to slight randomize the stack's starting point.
            execle(PROGRAM, PROGRAM, (char *)NULL, generate_random_env(random_n));
        }
        else {
            close(socket_fd);
        }
    } 

    close(listen_fd);
}


int socket_bind(int port)
{
    int listen_fd;
    int opt = 1;
    struct sockaddr_in server;

    if ((listen_fd = socket(AF_INET, SOCK_STREAM, 0)) == 0)
    {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }

    if (setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)))
    {
        perror("setsockopt failed");
        exit(EXIT_FAILURE);
    }

    memset((char *) &server, 0, sizeof(server));
    server.sin_family = AF_INET;
    server.sin_addr.s_addr = htonl(INADDR_ANY);
    server.sin_port = htons(port);

    if (bind(listen_fd, (struct sockaddr *) &server, sizeof(server)) < 0)
    {
        perror("bind failed");
        exit(EXIT_FAILURE);
    }

    if (listen(listen_fd, 3) < 0)
    {
        perror("listen failed");
        exit(EXIT_FAILURE);
    }

    return listen_fd;
}

int server_accept(int listen_fd, struct sockaddr_in *client)
{
    int c = sizeof(struct sockaddr_in);

    int socket_fd = accept(listen_fd, (struct sockaddr *)client, (socklen_t *)&c);
    char *ipAddr = inet_ntoa(client->sin_addr);
    printf("Got a connection from %s\n", ipAddr);
    return socket_fd;
}

// Generate environment variables. The length of the environment affects 
// the stack location. This is used to add some randomness to the lab.
char **generate_random_env(int length)
{
    const char *name = "randomstring=";
    char **env;

    env = malloc(2*sizeof(char *));

    env[0] = (char *) malloc((length + strlen(name))*sizeof(char));
    strcpy(env[0], name);
    memset(env[0] + strlen(name), 'A', length -1);
    env[0][length + strlen(name) - 1] = 0;
    env[1] = 0;
    return env;
}
```

This is the main entry point of the server, which listens to port <code>9090</code>. When it receives a TCP connection, it invokes the <code>format</code> program, and sets the TCP connection as the standard input of the <code>format</code> program. This way, when <code>format</code> reads data from <code>stdin</code>, it actually reads from the TCP connection, i.e. the data are provided by the user on the TCP client side.

### Container Setup

All the containers will be running in the background. To run commands on a container, we often need to get a shell on that container. We first need to use the "<code>docker ps</code>" command to find out the ID of the container, and then use "<code>docker exec</code>" to start a shell on that container.

The <code>docker-compose.yml</code> file:
```yaml
version: "3"
  
services:
    fmt-server-1:
        build: 
            context: ./fmt-containers
            args:
                ARCH: 32 
        image: seed-image-fmt-server-1
        container_name: server-10.9.0.5
        tty: true
        networks:
            net-10.9.0.0:
                ipv4_address: 10.9.0.5

    fmt-server-2:
        build: 
            context: ./fmt-containers
            args:
                ARCH: 64
        image: seed-image-fmt-server-2
        container_name: server-10.9.0.6
        tty: true
        networks:
            net-10.9.0.0:
                ipv4_address: 10.9.0.6

networks:
    net-10.9.0.0:
        name: net-10.9.0.0
        ipam:
            config:
                - subnet: 10.9.0.0/24
```

```console
$ dcup -d
Building fmt-server-1
Step 1/6 : FROM handsonsecurity/seed-ubuntu:small
small: Pulling from handsonsecurity/seed-ubuntu
da7391352a9b: Already exists
14428a6d4bcd: Already exists
2c2d948710f2: Already exists
5d39fdfbe330: Pull complete
56b236c9d9da: Pull complete
1bb168ce59cc: Pull complete
588b6963c007: Pull complete
Digest: sha256:53d27ec4a356184997bd520bb2dc7c7ace102bfe57ecfc0909e3524aabf8a0be
Status: Downloaded newer image for handsonsecurity/seed-ubuntu:small
 ---> 1102071f4a1d
Step 2/6 : COPY server    /fmt/
 ---> f975b401bcf4
Step 3/6 : ARG ARCH
 ---> Running in 5956c753e5ed
Removing intermediate container 5956c753e5ed
 ---> e24615f8b588
Step 4/6 : COPY format-${ARCH}  /fmt/format
 ---> 8bc77a17bb69
Step 5/6 : WORKDIR /fmt
 ---> Running in 81d04dd58397
Removing intermediate container 81d04dd58397
 ---> 2987e71b4a66
Step 6/6 : CMD ./server
 ---> Running in e307627655f7
Removing intermediate container e307627655f7
 ---> 5f3ba64a89a4

Successfully built 5f3ba64a89a4
Successfully tagged seed-image-fmt-server-1:latest
WARNING: Image for service fmt-server-1 was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Building fmt-server-2
Step 1/6 : FROM handsonsecurity/seed-ubuntu:small
 ---> 1102071f4a1d
Step 2/6 : COPY server    /fmt/
 ---> Using cache
 ---> f975b401bcf4
Step 3/6 : ARG ARCH
 ---> Using cache
 ---> e24615f8b588
Step 4/6 : COPY format-${ARCH}  /fmt/format
 ---> b576ccf0bb52
Step 5/6 : WORKDIR /fmt
 ---> Running in a6744edf74f2
Removing intermediate container a6744edf74f2
 ---> 6ee5e26695b8
Step 6/6 : CMD ./server
 ---> Running in 8b9a83764c6b
Removing intermediate container 8b9a83764c6b
 ---> 8cc3521aa5c1

Successfully built 8cc3521aa5c1
Successfully tagged seed-image-fmt-server-2:latest
WARNING: Image for service fmt-server-2 was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating server-10.9.0.6 ... done
Creating server-10.9.0.5 ... done
$ docker-compose down
Stopping server-10.9.0.5 ... done
Stopping server-10.9.0.6 ... done
Removing server-10.9.0.5 ... done
Removing server-10.9.0.6 ... done
Removing network net-10.9.0.0
```

## Task 1: Crashing the Program

## References

[^1]: See [owasp.org](https://owasp.org/www-community/attacks/Format_string_attack).

[^2]: "Exploiting Format String Vulnerabilities" by TESO, Version 1.2, September 1, 2001.