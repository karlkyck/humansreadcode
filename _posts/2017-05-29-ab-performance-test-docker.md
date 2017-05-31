---
layout: post
title: "Performance Testing using ab & Docker"
author: karl
date: 2017-05-29
comment: true
categories: [Articles]
published: true
noindex: false
---
Whenever possible I use official Docker images to run applications, services, and tools so I don't have to install and maintain a myriad of different applications on my development machine.
The Apache HTTP server benchmarking tool `ab` is a handy utility you can use to quickly performance test your HTTP services.
It comes bundled as part of the Apache HTTP server installation, but why bother installing it when you can just use the official Docker image.
 
## Docker Run
To run the latest Docker image for the Apache HTTP server on linux:

```
docker run -dit --name httpd-ab -v /var/www/html:/usr/local/apache2/htdocs/ httpd
```

This will pull down the latest official Apache HTTP server Docker image, start the `httpd` server, and name the container `httpd-ab`.
The volume `-v /var/www/html:/usr/local/apache2/htdocs/` is used to serve HTML from the `/var/www/html` directory on the host machine.

## Docker Exec

You can access the `ab` command line tool on the running container by running the following command on linux:

`docker exec -it httpd-ab bash`

This will drop you into a bash session running on the container.
You can now run the `ab` command to performance test your application.

## Example Command
Here's an example command you can run to performance test a server:

`ab -c 350 -n 20000 example.com/`

This command will execute 20000 HTTP GET requests to example.com maxing out at 350 simultaneous requests.
