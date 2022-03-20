---
title: "Exploring Google Workspace Upload Limits"
date: 2022-03-20T15:56:39Z
draft: true
ShowToc: true
cover:
    image: "<image path/url"
    # can also paste direct link from external site
    # ex. https://i.ibb.co/K0HVPBd/paper-mod-profilemode.png
    alt: "<alt text>" #short written description of an image for accessibility, if image cannot be viewed
    caption: "750Gb a day?"
    relative: false # To use relative path for cover image, used in hugo Page-bundles
    responsiveImages: true # Set false to reduce generation time and size of the site
    linkFullImages: true # Set true to enable hyperlinks to the full image size on post pages
params:
    ShowBreadCrumbs: true
    ShowShareButtons: true
    ShowPostNavLinks: true
---
## Overview
Google Workspace Enterprise plans (successor to G Suite Business plans) include unlimited Google Drive storage for licensed users, but [there is a strict 750Gb limit](https://apps.google.com/supportwidget/articlehome?hl=en&article_url=https%3A%2F%2Fsupport.google.com%2Fa%2Fanswer%2F172541%3Fhl%3Den&product_context=172541&product_name=UnuFlow&trigger_context=a) on data a licensed user can upload to their account within 24 hours. This limit is generally quite reasonable and should not pose an issue. The vast majority of people today do not have internet connections sufficient to upload 750Gb of data within 24 hours [even if they wanted to](https://www.speedtest.net/global-index). That being said, some people do have sufficient connections, and I'm happy to count myself among them (Thanks, [altafiber!](https://www.cincinnatibell.com/)). 

With a 250Mb/s upload speed, I have personally run into situations where I happen to need a lot of data, say 800Gb (just over the daily limit), uploaded ASAP. With my connection, I should be able to upload that data in a little more more than 7.5 hours, but instead I'm stuck waiting at least 24. Not the end of the world, but frustrating nonetheless. Recently, I used the time I was stuck waiting due to this limit to explore if there were any ways around it. Turns out, there's a couple!

## The Quick Way
The easiest solution is to compress all the data you want to upload to a single file. Zip, Tar, Rar, whatever, just create a compressed archive of your data. There's a million tools out there to make that happen (including those built-in to your OS), but I recommend [7-Zip](https://www.7-zip.org/) on Windows, and [Keka](https://www.keka.io/en/) on macOS. Once you have created your archive, Google Drive will allow you to upload a single file ([up to 5Tb in size](https://apps.google.com/supportwidget/articlehome?hl=en&article_url=https%3A%2F%2Fsupport.google.com%2Fa%2Fanswer%2F172541%3Fhl%3Den&product_context=172541&product_name=UnuFlow&trigger_context=a)) without cutting you off after you've uploaded 750Gb. As soon as that upload completes, assuming 750Gb worth of that file were uploaded within the last 24 hours, you will have triggered your limit and be unable to upload anything else for ~24 hours. Its difficult to tell exactly when the system will unlock and allow uploads again, but hey, at least you got your data uploaded ASAP. 

## The Interesting Way
What if, for whatever reason, you don't want to compress your data to an archive? Maybe you don't have enough disk space to store your data and a zip of your data at the same time. There is another way! It involves the scripted use of rclone and Google Workspace service accounts. I do not necessarily recommend this work-around, and highly doubt Google would smile upon it, but it does exist, is technically interesting, and I'm hardly the first to write about it-- so lets discuss. 

The general premise of this work-around is embracing the fact that a single account cannot upload more than 750Gb in a day. No way around that fact. So if a single account can't do it, just use multiple! Simple enough in theory, and not that much more complicated to implement. Also, while this method does theoretically enable abuse of the service, I do not think utilizing it is inherently so. In my opinion, this method can be utilized ethically so long as it is utilized sparingly, and for data which does not exceed 5Tb in size. Google will, after all, accept 5Tb within a day anyway, so long as you compress first. This just saves you a bit of trouble. 

> ### Note 
> I am hardly the first to discover or publicize this method. There are [projects](https://gist.github.com/korjjs/2c7b256825c5c70fe5b2c33980413d95) floating around online which enable automating this work-around to great effect, allowing uploading of up to 75Tb a day (100x the usual limit). While I find these projects interesting academically, they are more complex to implement than the method I've detailed below. Furthermore, their entire architecture seems to be built around enabling mass abuse of the Google Workspace Enterprise storage system. Unless you plan to abuse that system en masse (and dare Google to terminate your account while you're at it), I don't think there is much practical need for the additional complexity these projects introduce. 

### Implementation
Implementing this method for yourself is relatively simple. First, [install rclone](https://rclone.org/downloads/). It's available for many Operating Systems, including Windows, macOS, and Linux. Once rclone is installed, go ahead and login to your Google Workspace admin panel at [admin.google.com](https://admin.google.com/). 

From the admin panel, navigate to Directory-->Groups, then click "Create Group". 

{{< figure src=groups.png align=center >}}

On the ``Create Group`` screen, add a name for the group, an email for the group, and set your account to be the group owner. Lastly, check the box to make it a security group. 