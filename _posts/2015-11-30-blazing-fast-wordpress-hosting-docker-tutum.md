---
layout: post
title: "Blazing fast WordPress hosting in minutes with Docker and Tutum"
date: 2015-11-30 04:20:00 -0500
categories: docker wordpress
---

Impossible, you say? Not so fast!

Containers have been around for years, but Docker brought them to the masses and market adoption in 2015 has been nothing short of astounding. That massive growth has created a lush ecosystem for companies to build on those open-source technologies. One of them is [Tutum](https://www.tutum.co/), a container-as-a-service platform whose best feature is arguably its independence on infrastructure providers. While they have native support for AWS, Azure, DigitalOcean, SoftLayer and Packet, *tutum-agent* can be installed on any [supported Linux distribution](https://support.tutum.co/support/solutions/articles/5000513678-bring-your-own-node).

The typical WordPress install simply puts Apache in front of PHP and exposes it directly to the Internet. Every time a request comes in, Apache spawns a new thread, WordPress makes its database calls and renders the page. Simple, but very resource intensive &mdash; with scaling coming at at the cost of memory. While high performance may often comes at the cost of complexity, containers dramatically reduce that barrier of entry, if not remove it altogether.

One of the ways to tackle the speed of a Web application is to add cache in front of the Web server. There are a few choices here, Varnish being one of the popular and well documented options, including community-built configuration files (VCLs) specifically for WordPress.

First up, the **MySQL** database using the [official image](https://hub.docker.com/_/mysql/) from Docker Hub. Its configuration, if we can call it that, is *really* straightforward. Simply define a *strong* root password, the path where data should be stored on the host and, well, that's it.

{% highlight bash %}
mysql01:
  image: 'mysql:5.6'
  autorestart: on-failure
  environment:
    - MYSQL_ROOT_PASSWORD=**USE_A_STRONG_PASSWORD**
  tags:
    - node-tag
  volumes:
    - '/host/path/to/mysql/data:/var/lib/mysql'
{% endhighlight %}

Next up is **WordPress** itself, also using the [official image](https://hub.docker.com/_/wordpress/). This container must be linked to *mysql01* and provided with a path containing WordPress installation. If there is no WordPress installation in the host path, the image will download the tagged version &mdash; 4.3 in this case.

{% highlight bash %}
wordpress01:
  image: 'wordpress:4.3-apache'
  autorestart: on-failure
  environment:
    - WORDPRESS_DB_NAME=wp_example
  links:
    - 'mysql01:mysql'
  tags:
    - node-tag
  volumes:
    - '/host/path/to/wordpress:/var/www/html'
{% endhighlight %}

As mentioned earlier, **Varnish** is an HTTP cache and also the key to high performance. This [image](https://hub.docker.com/r/benhall/docker-varnish/) comes courtesy of [Ben Hall](https://twitter.com/ben_hall) and is [built specifically for WordPress](http://blog.benhall.me.uk/2015/01/scaling-wordpress-varnish-docker/). It stands between the load balancer and Apache, serving up requests from memory whenever possible and forwarding them only when it cannot. In the best case scenario, Varnish is able to serve the entire site from cache. For read-only WordPress sites or unauthenticated users, this is exactly what will happen. However, if logged in users are being served personalized content, there is no way around hitting the database at some point. `VIRTUAL_HOST` is used by haproxy to route the request so make sure to change this to your own domain name.

{% highlight bash %}
varnish01:
  image: 'benhall/docker-varnish:latest'
  autorestart: on-failure
  environment:
    - 'VIRTUAL_HOST=example.com,www.example.com'
    - VARNISH_BACKEND_PORT=80
    - VARNISH_BACKEND_HOST=backend
  links:
    - wordpress01:backend
  ports:
    - '8080:80'
  tags:
    - node-tag
{% endhighlight %}

Finally, **haproxy** serves as the entrypoint, managing incoming HTTP requests and routing them to the proper container based on the host name. For those of you on HTTPS (hats off!), this is where you'll want to perform SSL/TLS termination. Tutum has crafted an [image](https://hub.docker.com/r/tutum/haproxy/) that load balances between linked containers and, provided it is deployed on Tutum, it will also reconfigure itself when containers are added, redeployed or stopped.

{% highlight bash %}
lb:
  image: 'tutum/haproxy:0.2'
  autorestart: on-failure
  deployment_strategy: high_availability
  links:
    - varnish01
  ports:
    - '80:80'
  roles:
    - global
  tags:
    - node-tag
{% endhighlight %}

Rather than deploying each of those containers one by one, Tutum supports **[Stacks](https://support.tutum.co/support/solutions/articles/5000583471-stack-yaml-reference)** as a form of container orchestration which is more or less equivalent to Docker Compose. This makes it possible define and deploy multiple services in a single click.

You might have noticed each of the services above have a `tags` property. This is what Tutum uses to decide which node to deploy the services to

Next up: blazing fast and *high-availability* WordPress.
