---
author: dbrower
category: practices
filename: 2016-11-21-moving-item-downloads-from-the-ruby-loop.md
layout: post
tagline: A (request) path less taken
title: Moving Item Downloads From the Ruby Loop
tags: 'ruby, hydra, nginx, golang, design'
---

In most web applications, handling the download of a file is fairly standard:
the application receives a download request, say `GET /item/1`, determines whether the download is permitted,
and, if allowed, copies the data into the response.
For most [Hydra](https://projecthydra.org/) applications that means copying data out of Fedora.
There are two problems with that in our situation, especially when downloading large files.
First, it ties up the application instance by making it babysit a download.
This is a big deal for us since our application is in Ruby and each application instance is precious.
Second, our default application server, Unicorn, will kill requests that take longer than 30 seconds to complete.
Downloading a large file, especially to clients with poor internet connections, can easily take longer than
that to complete, and end up killed.

We solved it by moving the second piece of the download request—the copying of bits from Fedora to the response—to a second, non-Ruby, download application.
Download requests still begin with the Ruby Hydra application determining whether the download is allowed.
If it is, the copying of bits is handed off to our download proxy app.
We can do this in a way that is transparent to the client since we front everything with Nginx,
and use a special Nginx directive to do the hand-off.
(As far as I'm aware, Apache does not have a similar capability).

The hand-off takes care of both of our problems with the original setup.
The Ruby application code only spends as much time as it takes to do the authorization on the request,
allowing that application instance to process other requests without waiting for the download to finish.
Second, our download proxy doesn't have a 30 second timeout like our Ruby Unicorn workers, and will not
kill a download taking longer than that.

This post describes how we did this in exhaustive detail.
I hope other people find it useful and implement similar designs.
There are three pieces to it: our download application, configuring Nginx, and altering the Hydra application.


## Download Proxy Application

We made a tiny Go application to set the correct response headers and proxy content from Fedora.
We named the app [disadis](https://github.com/ndlib/disadis), and it was one of our pilot Go projects.
When we did this in 2014, as well as today, we are using Fedora 3.x.
(I would expect setting this up to use Fedora 4 to be similar.)
I would also like to note that it is probably possible to remove the application all together and have Nginx
talk with Fedora directly.
We decided having the application gave us more control over the response.

The application listens on port 4000 for requests of the form `/{id}`.
When a request happens, the program gets the datastream contents of `/objects/und:{id}/datastreams/content/content` from Fedora
using a configured fedora user and password; it sets the headers to have the correct filename; and streams the content back to the client.

We run the Ruby code and the Go code as two separate applications on the server.
A schematic of the request process would look like this.

    Client ---> Nginx ----> Ruby
                      \---> Go   ---> Fedora

## Nginx Configuration

The hand-off uses the Nginx [`X-Acell-Redirect`](https://www.nginx.com/resources/wiki/start/topics/examples/x-accel/)
directive, which lets an application do an internal redirect to another one,
completely invisible to the client, who sees only the final response to the request.

We created an internal location in our Nginx configuration for the proxy application.
Here is the relevant configuration section:

    location ^~ /download-content/ {
        internal;
        proxy_intercept_errors on;
        error_page 401 403 404 500 = @app;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
        proxy_redirect   off;
        proxy_buffering  off;
        proxy_pass http://127.0.0.1:4000/;
    }

In this block we set up the path `/download-content/` as an internal route.
The specific route is not particularly important, so long as the Ruby application redirects to it.
Being an internal route is important, since it means Nginx will not allow clients to request resources from it directly;
the only way to arrive here is via an internal redirection.
The block then says that all errors should be handled by the application,
and disables buffering the response, which is important for streaming downloads.
Finally, it specifies `localhost:4000` to handle all requests to this path.
Because of the trailing slash on the `proxy_pass` directive, when the redirect happens,
Nginx will strip off the initial `/download-content/`, so internal redirects to
`/download-content/12345` will become requests to `localhost:4000/12345`.
Finally, localhost:4000 is where our download proxy is listening.


## Changes to Ruby Application

When the Ruby application wants to start a download, it returns a response
with the header `X-Accel-Redirect` set to the path to redirect to.
In our case that path is `/download-content/{id}`.
Since the response body is not important, we only send back a head request.
The code is similar to the following.

    if download_proxying_enabled?
      response.headers['X-Accel-Redirect'] = "/download-content/#{asset.noid}"
      head :ok
    end

This section determines whether the proxy application is being used
(e.g., we don't necessarily use it on local development).
If it is, then we are then we invoke the Nginx internal redirect and return.
Nginx handles it from here by then forwarding the redirected request to the download application.

This is an extremely useful design, and I hope others in the Hydrasphere will also find it useful.
