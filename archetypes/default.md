---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: true
ShowToc: true
cover:
    image: "<image path/url"
    # can also paste direct link from external site
    alt: "<alt text>" #short written description of an image for accessibility, if image cannot be viewed
    caption: "<text>"
    relative: false # To use relative path for cover image, used in hugo Page-bundles
    responsiveImages: true # Set false to reduce generation time and size of the site
    linkFullImages: true # Set true to enable hyperlinks to the full image size on post pages
categories:
  - "category"
tags:
  - "tag"
params:
    ShowBreadCrumbs: true
    ShowShareButtons: true
    ShowPostNavLinks: true
---
