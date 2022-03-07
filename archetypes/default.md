---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
draft: true
ShowToc: true
cover:
    image: "<image path/url"
    # can also paste direct link from external site
    # ex. https://i.ibb.co/K0HVPBd/paper-mod-profilemode.png
    alt: "<alt text>" #short written description of an image for accessibility, if image cannot be viewed
    caption: "<text>"
    relative: false # To use relative path for cover image, used in hugo Page-bundles
    responsiveImages: true # Set false to reduce generation time and size of the site
    linkFullImages: true # Set true to enable hyperlinks to the full image size on post pages
params:
    ShowBreadCrumbs: true
    ShowShareButtons: true
    ShowPostNavLinks: true
---
