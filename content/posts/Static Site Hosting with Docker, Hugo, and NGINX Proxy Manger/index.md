---
title: "Static Site Hosting for Free with Docker, Hugo, and NGINX Proxy Manger"
date: 2022-03-03T13:35:19-05:00
draft: true
ShowToc: true
cover:
    image: "cover.png"
    # can also paste direct link from external site
    alt: "text"
    caption: "My mission to build a site and achieve simple DIY web-hosting" #short written description of an image for accessibility, if image cannot be viewed
    relative: false # To use relative path for cover image, used in hugo Page-bundles
    responsiveImages: true
    linkFullImages: true
categories:
 - "tutorials"
 - "Web Hosting"
 - "Docker" 
tags:
  - "Hugo"
  - "NGINX Proxy Manager"
  - "Docker Compose"
  - "Dynamic DNS (DDNS)"
params:
    ShowBreadCrumbs: true
    ShowShareButtons: true
    ShowPostNavLinks: true
---
## Overview
It seems appropriate that my first post on this site be an exploration of its own technical underpinnings. That being said, before I get too far into the weeds, a bit of context. When I set out to build my site I had a few goals in mind. Those goals can be reasonably summed up as wanting to build a site from markdown files in Github, and self-host it using docker containers. Simple enough, but lets unpack that a bit:

* Self-Host
  * Run all the website infrastructure in my home, and avoid paying a hosting provider like Squarespace or Wix, as nice as those tools can be.
* Write all my posts in [markdown](https://daringfireball.net/projects/markdown/)
  * Markdown enables easy text-formatting, and allows me to do all my writing in any number of great, markdown-supporting text-editors on basically any computing platform.
* Run all infrastructure in containers
  * Running my web-hosting infrastructure entirely inside docker containers allows me to easily change what computer is running my website in the event that doing so becomes necessary or desirable. Migrating to another computer I own, to a Virtual Private Server (VPS) I rent, or to the Cloud with Azure or AWS container hosting-- it all becomes trivial.
* Store the site as a repository in Github
  * Site files being a Github repo provides me with backups and quick visibility into my site's version history. Additionally, it works alongside the docker containers to keep my site mobile. If I decide to migrate hosts in the future, I won't need to worry about transferring my site files manually. With them in Github, all I would need to do is initiate a git clone of my site's repo on the new host, and the site files are ready to go!

With those goals in mind, I eventually landed on using docker containers of [Hugo](https://gohugo.io) (to build my site), and [NGINX Proxy Manager](https://nginxproxymanager.com) (to host it). If you would like to do something similar, and build a site like the one you are currently viewing, the below should be helpful! With that out of the way, lets begin.

## Requirements
In order to successfully build and host a site using this method, the below are prerequisites:

1. **Own a domain name.** Mine is [wallacelabs.tech](https://wallacelabs.tech). If you do not already own one, I recommend purchasing from [Hover](https://www.hover.com) or [Google Domains](https://domains.google). As part of owning a domain, you will have control of your domain's DNS records. If you somehow own a domain but do not have the ability to manage its DNS records, you'll have to get that worked out before proceeding.
2. **Own an internet-connected computer.** This computer will be your web-server. Ideally, that computer will be a server in the sense that it is always on, and dedicated to server functions. While you could host your website on your everyday use computer, I would not recommend it. In terms of actual hardware though? It can be anything from a tiny Raspberry Pi to a massive EPYC Server. The important thing is that it can run docker, is connected to the internet, and is always on.
3. **Be able to manage port-forwarding rules on your network's router/gateway.** Port-forwarding on your router, along with setting your domain's DNS records, are the two keys required to map traffic from the internet to the web-server hosting your site.
4. **Either a Static WAN IP, or a Dynamic DNS configuration.** This is required in order for your domain's DNS rules to reliably point traffic to the router on your local network. Most home internet connections do not have static WAN IP's, so you'll likely need to configure a Dynamic DNS (DDNS) service. Stated simply, DDNS is a way of automatically informing your domain's DNS servers of any changes to your home network's Public IP Address. [Google Domains has a highly convenient DDNS service built in](https://support.google.com/domains/answer/6147083?hl=en), if you use them as your domain registrar. If you do not use Google Domains, I would strongly recommend using [DuckDNS](https://www.duckdns.org/why.jsp). 

{{< figure src=flow.png align=center caption="When your website is accessed from the internet, the traffic will follow this general path. Most of these requirements exist in order to make sure that traffic can reliably traverse this path, allowing your site to be online." >}}

If you can check all those boxes, you should be good to go.

## Phase One - Getting Your Site Running Locally
Before worrying about domain DNS records, web hosting, and port-forwarding rules, we're going to focus on getting a site running on your local network. First, we'll setup the web-server.

### Installing Docker
In order to setup containers for Hugo and NGINX Proxy Manager, you will need Docker installed on your web-server. Docker runs on Windows, macOS, and Linux, so the Operating System (OS) you use is mostly up to you. It is, however, worth mentioning that less setup is required on Linux and macOS. To run Docker in an ideal manner on Windows, you will need to configure Windows Subsystem for Linux (WSL) and subsequently, a Linux Distribution of your choice to run on WSL. This is not *overly complicated*, but it is outside the scope of this discussion. A post for another day, perhaps.

> **A quick aside:** *If you plan on using this system as a general purpose home server, in addition to using it as a web-host, it is definitely worth considering the security implications of doing so. By using a computer for web hosting, you are inherently opening up this system to the internet, and there is a lot to be said for the practice of isolating web-hosts on your network through any variety of means. A thorough consideration of these potential risks (and the network hardening measures which might be taken to mitigate them) would be a large digression, and thus is outside the scope of this discussion. Another post for another day, perhaps. For now though, suffice it to say that our risk factor here is not particularly large, due primarily to the fact that we are discussing hosting a static site, not a dynamic one. Had I opted to host a dynamic site using Wordpress, or something of the like, I would be singing a different tune. For additional reading on the security implications of static vs. dynamic sites, [see here](https://srandby.org/digital-writing/index.html).*

Begin by [installing the Docker Engine](https://docs.docker.com/engine/install/) using the instructions specific to your OS. Once Docker is installed, you can move on to setting up a Hugo container.

### Setting up a Hugo container

To create my site, I utilized a container based on the [Hugo Docker Image](https://hub.docker.com/r/klakegg/hugo/). Hugo's Docker-Hub image is highly configurable, offering many different tags for specialized integration into custom web-publishing workflows. For my use, all I really wanted was basic container which, when run, provides an interactive shell environment where I can run Hugo commands. The below docker compose file will accomplish this:

{{< gist swallace17 2ce37da57497e37ec36110c24eb20668 >}}

To utilize this compose file to setup your own Hugo container, first, go ahead and download it. Then, open it in your text-editor of choice and update the volume mapping from ``*PATH TO SITE*`` in line 6 to wherever you would like your site's git repo to live on your host system. I would recommend creating a folder called ``site`` at the root level of your host. If you go that route, just change ``*PATH TO SITE*`` to ``/site`` and you're good to go. Lastly, open up a Terminal/Powershell session, navigate to whichever folder ``docker-compose.yml`` has been downloaded to, and run the following command:

```bash
docker-compose up
```

You should see the following in your Terminal session:

{{< figure src=compose-terminal.png align=center >}}

Now, open up Docker Desktop and you should see your container running, like this:

{{< figure src=compose-DD.png align=center caption="Note that Docker Desktop is only currently avalible for macOS and Windows, though a Linux version [is in Tech Preview](https://docs.docker.com/desktop/linux/), should you wish to try it." >}}

With the Hugo container setup and ready to go, you can now use it to create your Hugo site-template.

### Creating your Hugo-Site

Now we're in business. Click the "CLI" button in Docker Desktop to open up a CLI session on your Hugo container. This is how you'll interact with the containerized Hugo program moving forward.

Your CLI session will have opened up inside the container's ``/src`` folder by default. The ``/src`` folder is a volume mapped to the host. Whatever data the container puts in this folder, you will be able to access from the host, and vice versa. In docker containers, data inside a volume map like this one is also the only data that will persist after the container is stopped or restarted (i.e. the ``/src`` folder is the only thing that does not get deleted when the container stops). The rest of the container is rebuilt from scratch every time the container starts. Given you probably want your Hugo-site to not be deleted after you create it, you will want to create it inside the ``/src`` folder. Furthermore, whenever you run Hugo commands, you'll want to run them in this folder (Hugo commands have to be run against a Hugo-site, and this is where your Hugo-site will be located).

 So, with a CLI session opened at ``/src``, run the below command. When you do, a blank Hugo site template will be initialized as a git repo inside the ``/src`` folder. Just be sure to change ``*YOUR_SITE_NAME*`` to whatever you would like to call the repo containing your site.

```bash
hugo new site *YOUR_SITE_NAME*
```

The Hugo-site will be generated, and you will see the below message in your CLI session:

{{< figure src=site-creation-success.png align=center caption="Success!" >}}

Next up, as indicated in that success message, you'll need to [pick a theme](https://themes.gohugo.io) and install it. There are tons to choose from! If you are curious, the site you are viewing now is running on [PaperMod](https://themes.gohugo.io/themes/hugo-papermod/). Installing your chosen theme is as simple as following the instructions provided by your theme of choice. These instructions usually consist of installing the theme as a git-submodule. This method is convenient as it allows you to easily update your theme by initiating a git pull on that submodule. Put more simply- you'll have a one-click method of updating to a new version of your chosen theme in the future, should you want to do so. As you follow installation instructions, just make sure to run any necessary commands inside the Hugo container's CLI. With a blank Hugo site template created and your theme installed, the foundation of your site is in place.

### Creating your first post
With your site created, you can now create a quick test post. Before that though, a brief rabbit-trail about Leaf Bundles.

#### To Leaf-Bundle, or not to Leaf-Bundle

> In a moment, you'll run a command that will create the first post on your newly created site. That command is going to create the post as something Hugo calls a [Leaf Bundle](https://gohugo.io/content-management/page-bundles/). I recommend creating all your posts as a leaf bundle because, among other benefits, it allows you to bundle post-images inside the same folder as as the markdown text-file of the post itself. If you do not create your posts as leaf bundles, any images for your posts will be lumped together, completely disorganized, inside a folder named ``static`` at the root level of your site. Not ideal. The below diagrams should help visualize this:

**Leaf Bundle Post Structure**
>```text
>content/
>└──posts
>    └── YOUR_POST_NAME
>        ├── image1.png
>        ├── image2.png
>        └── index.md

**Non-Leaf Bundle Post Structure**
>```text
>content/
>├── posts
>│   ├── my-post.md
>│   └── my-other-post.md
>└── static
>    ├── my-post_image1.png
>    ├── my-post_image2.png
>    ├── my-post_image3.jpg
>    ├── my-other-post_image1.png
>    └── my-other-post_image2.jpg

As you accumulate more and more posts, the disorganized pile of images inside ``/static`` will continue to get larger and more unwieldy. I highly prefer the structure provided by the leaf bundle method, and I think you will too. Now, back to the business of creating your first post.

#### Creating a post (As a Leaf-Bundle!!)
Run the below command in the Hugo container CLI to create a test post on your site:

```bash
hugo new content/posts/*YOUR_POST_NAME*/index.md
```

If you wish, feel free to modify index.md to add a quick "hello world" message, or something of the like, but do not feel obligated. Your first post has been created, whether it has anything written in it or not. With that, time to generate your site!

### Generating your Site

One last command to run now.

```bash
hugo
```

With that, your hugo site has been converted from a pile of markdown documents, images, themes, and the like (that only Hugo can understand) to an actual HTML website that can be displayed by any web browser. You'll find the generated website inside the ``public`` folder created at the root-level of your site's repo. 

>  This folder is what your web-server will be pushing out to the internet whenever someone accesses your website. Your web-server will only serve the HTML files that are in this server, so any time you want to update your site, or add a new post, you will need to run the ``hugo`` command again to generate your site fresh.

### Hosting your Site! (Locally)
Before setting up your newly generated site to be accessed over the internet, we're going to preview it on your local network. Hugo includes a built-in web-host function for this exact purpose. To take advantage of this, run the below command (again, inside the Hugo Container's CLI):

```bash
hugo server
```

With luck, you'll be greeted with the following message:

![Local Hosting Success](hugo-server.png)

Open [http://localhost:1313](http://localhost:1313) in your browser of choice to view your site! As you may have noticed in the above image, the nice thing about previewing your site this way is that Hugo generates the preview in something called "Fast Render Mode." Fast Render Mode means you don't have to run the ``hugo`` command and regenerate your site every time you want to see a change displayed in your browser. Using the local preview, if you are editing a post, just save any changes you've made to the ``index.md`` file and you will see them immediately reflected in your browser. Live previewing! 

> If you were not so lucky, you may have gotten an error in your CLI session that says something like "Check your Hugo installation: you need the extended version to build...". This is because some themes require an extended version of the Hugo docker image in order to build successfully. Remember earlier, when I said the Hugo image is highly configurable, has many tags, blah blah blah? Here you have a great illustration of that fact. If this happens to you, change line 3 of ``docker-compose.yml`` from ``image: klakegg/hugo:latest`` to ``image: klakegg/hugo:ext-alpine``. Shut down your container, relaunch it using ``docker-compose up``, and you should be good to go.

## Phase Two - Hosting your Site on the Web!
Believe it or not, this should be the easy part. We're going to spin up another container, this time for [NGINX Proxy Manager (NPM)](https://nginxproxymanager.com/). NGINX Proxy Manager is an awesome web-based GUI built on top of the [NGINX web server](https://www.nginx.com/). It is generally used for securely exposing services running on your network to the internet. For example, if you wanted to securely access your [Home Assistant Server](https://www.home-assistant.io/) over the internet, rather than strictly on your home network, or via VPN. In that scenario, you already have a web-server operating a given service or web-app locally, you are just using NPM as a reverse proxy to access that service over the internet. So how can NPM be used to host your site? With a little bit of tricky configuration, NPM can be used to host static files to the web-- which is exactly what Hugo static sites are!

As preparation for that, lets quickly configure DNS rules for your domain, and add port forwarding rules to your router.

### Configuring your Domain and Router

Begin by logging into your domain registrar's website, then pull up the page for managing your domain's DNS records. For me, on Google Domains, it looks like this: 

{{< figure src=gDomains-DNS.png align=center caption="" >}}

You will want to create a DNS A record resolving ``yourDomain.com`` to your [public IP address](https://whatismyipaddress.com/). Ideally, this address will be static, but most home connections are not. With that in mind, I'd recommend configuring Dynamic DNS (DDNS) via [Google Domains](https://support.google.com/domains/answer/6147083?hl=en) or [Duck DNS](https://www.duckdns.org/why.jsp), as earlier stated. Without DDNS in place, your site will go down any time the public IP on your home network changes (usually when you reboot your modem). With this DNS record in place, anytime someone accesses ``yourDomain.com``, your domain registrar's nameservers will resolve the request to your home network's router/gateway. 

All thats left is to setup your router/gateway to send that traffic to your webserver, completing the chain. With that in mind, login to the management interface of your router/gateway, navigate to the Port-Forwarding management page, and add a rule. This will be different on every router, but for me, on a Ubiquiti UniFi based network, it looks like this: 

{{< figure src=pf-rule.png align=center caption="Make sure to set 'Forward IP' to the IP address of your web-server. Also, double check and make sure your web-server is setup with a static IP address while you're at it. Don't want that changing on you!" >}}

You're going to need to create two rules here-- one for HTTP traffic coming in over port 80, and another for HTTPS on port 443. You're configuring these rules to route any traffic coming into your network over port 80 or 443 to your web-server. With those rules in place, you should see something like this: 

{{< figure src=pf.png align=center caption="Port-forwarding complete. Almost done!" >}}

With port forwarding complete, you've completed the chain such that all traffic going to your domain will reach your web-server, like in this image we looked at in the beginning: 

{{< figure src=flow.png align=center caption="Traffic mapping complete!" >}}


### Setting up an NGINX Proxy Manager container

We're nearing the end! Now that
1. Your HTML site is sitting in the ``public`` folder
2. Your router and domain DNS are configured to point HTTP and HTTPS traffic to your web-server

All thats left is to configure NGINX Proxy Manager to actually serve your website! To do that, use the below ``docker-compose.yml`` to create an NPM container on your host. Same drill as before, download the file, and you'll need to make a couple edits before running.

{{< gist swallace17 ac84eccb6fdca3c9b62ee4c162b1ba3a >}}

First, you'll need to chose where on your host system you would like NPM data and LetsEncrypt certificates to live. I recommend creating a folder named ``container_app_data`` at the root of your host, with sub-folder ``NPM``. Do that, then just change ``*PATH*`` in lines 10 and 11 of ``docker-compose.yml`` to ``/container_app_data/NPM``, and docker will automatically create sub-folders for Data and Lets Encrypt at ``/container_app_data/NPM/data`` and ``/container_app_data/NPM/letsencrypt`` respectively.

Lastly, you'll need to change ``*PATH TO SITE*`` in line 12. Set this to be the same as whatever you chose for ``*PATH TO SITE*`` when creating the Hugo container. Giving NPM access to ``*PATH TO SITE*/public`` will allow it to access your generated site (inside the Hugo container's public folder).

With these modifications in place, open a Terminal/Powershell session at the location of ``docker-compose.yml`` and run the command:

```bash
docker-compose up
```
If everything is setup correctly in ``docker-compose.yml``, the container will build and you'll be greeted by the following new container in Docker Desktop:

{{< figure src=npm-dd.png align=center caption="Two containers now!" >}}

### Configuring NPM for Static-Site Hosting

With the container running, open up a browser on your host machine and navigate to [http://localhost:81](http://localhost:81). When you do, you should see the NPM login page:

{{< figure src=npm-browser.png align=center caption="" >}}

To login, use the default username and password:

```text
Email:    admin@example.com
Password: changeme
```

You will then be prompted to setup new login credentials. As with anything, make sure to setup a unique, strong password (I recommend [1Password](https://1password.com/) to help with this). Once you've setup your new login setup, click on "SSL certificates" in the tab bar, then "Add SSL Certificate." On this screen, type in your domain name, email and "I agree to the Let's Encrypt Terms of Service", like this:

{{< figure src=create-cert.png align=center caption="" >}}

Before clicking save, make sure to click the button to "Text Server Reachability". You should see the following message if successful:

{{< figure src=domain-reachable.png align=center caption="" >}}

Success here demonstrates traffic from the internet is reaching your web server correctly. Congratulations! You must have setup your domain DNS records and port-forwarding rules successfully. If you don't get a success message here, you might need to go back and double-check those steps. Click the save button, and your SSL certificate will be added: 

{{< figure src=ssl-success.png align=center caption="" >}}

With your SSL certificate created, all thats left is to configure a Proxy Host, and your site is live! Click the ``Hosts`` tab, then ``Proxy Hosts``. On the Proxy Hosts page, click the ``Add Proxy Host`` button. The New Proxy Host screen will pop up, on the details tab. Set it up like this: 

{{< figure src=proxyhost.png align=center caption="Obviously be sure to use your actual domain name, not ``yourDomain.com`` and ``www.yourDomain.com``" >}}

Then click on the SSL tab and select the SSL cert you just created, like this: 

{{< figure src=ph-ssl.png align=center caption="" >}}

Lastly, click the Advanced tab, and paste the below as ``Custom Nginx Configuration``.  

```text
location / {
  root /site/;
}
```
like this: 

{{< figure src=ph-advanced.png align=center caption="This is the 'tricky configuration' I mentioned earlier. [Thanks to Burke Azbill writing on dimentionquest.net for figuring this out!](https://dimensionquest.net/2021/02/host-static-site-on-npm/)" >}}

This little bit of custom configuration is having NGINX host the static content contained inside the NPM container's ``/site`` folder. That folder just so happens to container your website, as, when you modified the ``docker-compose.yml`` for the NPM container, you mapped it to the same location as the ``public`` folder your website was generated in. Save this config, and your website should be live! Open up a browser, navigate to your domain name, and there it will be! With your site created, all you have to do now is fill it with content. Congratulations!

## Phase Three - Wrapping Up
Lets wrap up with connecting your site up to a Github repo, and some things you can do to enable some general quality-of-life improvements as you manage your site.

### Configure Additional Development Environments
This may sound counterintuitive, but I would recommend setting up as additional development environment on your everyday computer to do most of your site development on--setting up the site, testing changes, writing posts, etc. Then, whenever you are ready to publish, you push your changes up to your Github repo, and pull them down on the web server. Boom, published. This is how I personally develop for my site most of the time. On my laptop, I use the Atom text editor
