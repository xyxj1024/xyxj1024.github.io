---
layout:             post
title:              "Serving External Static Files with Pure Node.js"
category:           "Web Applications and Cybersecurity"
tags:               web-development javascript nodejs cse503-assignment
permalink:          /blog/nodejs-static-file-server
---

When implementing [the multi-room chat server project](https://classes.engineering.wustl.edu/cse330/index.php?title=Module_6) for Washington University CSE 503S (Spring 2023), I encountered the issue of loading external static files (e.g., CSS stylesheets, JavaScript programs, and images, etc.) into the web page. This post documents a simple solution inspired by [this MDN article](https://developer.mozilla.org/en-US/docs/Learn/Server-side/Node_server_without_framework).

<!-- excerpt-end -->

First of all, assume we have the `index.html` file and the `server.js` file under the same directory (i.e., the working directory where we install Node.js packages and run the `node server.js` command). Create a `public` or `static` directory under the working directory, which will contain several subdirectories including `css`, `img`, and `js`. For example, a directory tree might look like:

```console
$ tree .
.
├── README.md
├── index.html
├── public
│   ├── css
│   │   └── style.css
│   ├── img
│   │   ├── default.png
│   │   └── favicon.png
│   └── js
│       └── client.js
└── server.js

5 directories, 7 files
```

Within our `index.html`, include the static files in the following way:

```html
<link rel="icon" type="images/png" href="/img/favicon.png">
<link rel="stylesheet" type="text/css" href="/css/style.css">

<script type="text/javascript" src="/js/client.js"></script>
```

Basically, in order to serve anything underneath our `public` directory, we need something like this:

```javascript
let app = http.createServer(function (req, res) {
    let filename = path.join(__dirname, "public", url.parse(req.url).pathname);
    (fs.exists || path.exists) (filename, function (exists) {
        if (exists) {
            fs.readFile(filename, function (err, data) {
                ...
            });
        } else {
            ...
        }
    });
});
```

Use the following code at the beginning of our `server.js`:

```javascript
const http = require("http"),
      fs = require("fs"),
      path = require("path");

/* this is the port we used in the course */
const port = 3456;

const file = "index.html";
/*
 * Listen for HTTP connections.
 * This is essentially a miniature static file server that 
 * only serves our "index.html" on port 3456: 
 */
const server = http.createServer(function (req, res) {
    /* 
     * This callback runs when a new connection is 
     * made to our HTTP server.
     */
    if (req.url === "/") {
        fs.readFile(file, "UTF-8", function (err, data) {
            /* 
             * This callback runs when the index.html 
             * file has been read from the filesystem.
             */
            if (err) return res.writeHead(500);
            res.writeHead(200, { "Content-Type": "text/html" });
            res.end(data);
        });
    } else if (req.url.match("\.js")) {
        let scriptPath = path.join(__dirname, 'public', req.url);
        let fileStream = fs.createReadStream(scriptPath, "UTF-8");
        res.writeHead(200, { "Content-Type": "text/javascript" });
        fileStream.pipe(res);
    } else if (req.url.match("\.css")) {
        let cssPath = path.join(__dirname, 'public', req.url);
        let fileStream = fs.createReadStream(cssPath, "UTF-8");
        res.writeHead(200, { "Content-Type": "text/css" });
        fileStream.pipe(res);
    } else if (req.url.match("\.png")) {
        let imgPath = path.join(__dirname, 'public', req.url);
        let fileStream = fs.createReadStream(imgPath);
        res.writeHead(200, { "Content-Type": "image/png" });
        fileStream.pipe(res);
    } else {
        res.writeHead(404, { "Content-Type": "text/html" });
        res.end("No Page Found!");
    }
});
server.listen(port);
```

We can view those static files directly in the browser. For example, to view the `default.png` file, go to: `http://localhost:3456/img/default.png`. In my case, since I was using an AWS EC2 instance, I would replace `localhost` with my AWS public IPv4 address. Note that we are running our file server on port `3456` as opposed to the default web server port, which is `80`. Make sure to create a custom TCP rule for port `3456` under "Inbound" of the corresponding security group.