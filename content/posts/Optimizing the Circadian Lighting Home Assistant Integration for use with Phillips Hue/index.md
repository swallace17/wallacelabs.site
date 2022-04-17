---
title: "Optimizing the Circadian Lighting Home Assistant Integration for use with Phillips Hue"
date: 2022-04-17T03:55:50Z
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
  - "Home Automation"
tags:
  - "Phillips Hue"
  - "Home Assistant"
  - "Circadian Lighting"
  - "Scripting"
params:
    ShowBreadCrumbs: true
    ShowShareButtons: true
    ShowPostNavLinks: true
---
Before getting too into the weeds, lets talk a bit about home smart lighting. In my ongoing journey to create an ever more integrated smart-home, I have opted to utilize the Phillips Hue platform for my lighting. Generally speaking, I recommend against using smart bulbs, in favor of smart switches from Lutron's Caseta platform. In my home though, I go against my own advice. I do so because of how much I value the dynamic color, and color temperature functions offered by the Hue ecosystem. Yes, the bulbs are expensive, and their fixtures and other accessories generally even more so. You also run into the issue of light switches needing to always be toggled on, such that the bulbs always have power-- whether the LED's are lit up or not--  in order to talk to the bridge and allow the smart features to function. Hue [has a $40 product](https://www.philips-hue.com/en-us/p/hue-philips-hue-wall-switch-module/046677571160) which removes that irritation, but of course, that requires spending even more money on this ecosystem. You really gotta commit-- and I have-- but take that as a cautionary tale. If you can live without dynamic color and color temperature, smart switches are really the way to go (and don't waste your time on anything other than [Lutron's Caseta line](https://www.casetawireless.com/us/en)).

With all of that being said- if you, like me, use Hue for your lighting, you may have found yourself wishing that you could set your lights to automatically adjust their color temperature throughout the day. Warmer tones at night, and cooler ones during the day. Its an increasingly popular concept. In a perfect world, what I would like is for lights to be cool and bright during the day, become warmer as the sun is setting, and gradually become a bit dimmer as the night draws on. Doesn't seem too difficult, right?

For years, Phillips has offered a basic version of this functionality by utilizing the [Time-Based light](https://labs.meethue.com/formulas/huelabs/feel-better-with-light) formula (formerly "feel better with light") in the "My Labs" portion of the Hue app. This functionality is a bit limited though- it switches through a few pre-determined scenes at different times of day, rather than acting as a smooth transition from cool to warm as the day goes on. 

Apple, with iOS 14 at WWDC 2020, announced a HomeKit feature called Adaptive Lighting. Initially, it seemed like this feature was going to be amazing. In typical Apple fashion, however, it launches with basic functionality and little to no customization options. If you use HomeKit, Apple's Adaptive Lighting feature works with Hue, but with Adaptive Lighting, color temperature changes are inherently tied to changes in brightness. Basically, if you turn the brightness down, the light becomes warmer. If you make a light bright, the color temperature of the light becomes cooler. Also not what I'm looking for.

Since 2019, the only method I have found that even comes close to meeting my needs is a Home Assistant integration called [Circadian Lighting](https://github.com/claytonjn/hass-circadian_lighting). 

> If you are not familiar with Home Assistant, and you have a vested interest in home automation, you'll want to get familiar. In short, it ties together all the various home automation platforms from Apple, Google, Amazon, etc, and lets them all coexist and integrate together under one roof. There's a learning curve, but the payoff is worth it, and if you're reading this to begin with, its likely right up your alley. 

Circadian lighting does it all, and it very customizable to boot. It ties in to the Home Assistant platform, and allows me to automate all my lights slowly transitioning their color temperature throughout the day based on the movement of the sun. I first started using it in 2019, a week or so after I moved into my first apartment, configuring it at 2am the night before my wedding. Good times! I've been using Circadian ever since, and haven't found anything better. 

> The only thing that has gotten close is an extremely similar integration called [Adaptive Lighting](https://github.com/basnijholt/adaptive-lighting). It has a few extra features, but nothing that impacts my usage. I believe it started as a fork of Circadian Lighting, and both are actively maintained. I'd recommend checking out both, and seeing what will best fit your needs, but for now I'm still using Circadian Lighting. 

# Circadian Lightings Only Hiccup
I've only had one hiccup with Circadian Lighting, and solving it is what this post is all about. Circadian Lighting only smoothly transitions the color temperature of lights which are currently turned on. If you turn on a light, it will initially turn on to whatever color temperature it was set to last (or to a default value I guess, depends on how you have it configured). After a few seconds, Home Assistant sees that the light has been turned on, and Circadian Lighting kicks in and the light changes to the proper color temperature. Its far from the end of the world- but those few seconds of delay began to grate on me. Every time I turned a light on, I thought to myself "one of these days, I gotta figure out how to fix that." 

In October of 2019, my brother-in-law and I worked out a solution in [this Github issue](https://github.com/claytonjn/hass-circadian_lighting/issues/3) on the topic, along with another user, Bluefries. It was an exciting moment when we got it working, late one night. Long story short though, the modification involved changing some code in Circadian Lighting, the developer wanted to implement in a different way (for reasons that make sense, but are beyond the scope of this discussion), so a pull request was not submitted. Eventually, our modification broke with an update to Circadian Lighting. 

Robert forked the project, and I believe still has the modification working, but I'd ideally like to solve this issue without having to maintain an ongoing fork of the project. This is where the Home Assistant platform becomes very helpful. 

In summary- I've managed to fix this issue for myself by writing a Home Assistant automation which runs whenever I switch on one of my Hue-compatible light switches. The automation is triggered by a light switch being toggled, then it determines a color temperature and brightness value based on the home assistant sensor entity created by the Circadian Lighting integration, and finishes by setting the appropriate light home assistant entities (provided by the Phillips Hue integration) to the calculated color temperature and brightness. All of this happens outside of the Circadian Lighting integration itself, so it should work regardless of future updates, etc. 

There are downsides to this approach, namely occasional latency. Consider the flow of data that has to happen for a light switch to turn on here: 

(mindnode image)
Physical button pressed -- detect button in home assistant -- trigger automation -- automation reads values from Circadian Lighting sensor entity -- automation triggers corresponding value changes to light entities in the Phillips Hue Integration -- Phillips Hue Integration sends light value updates to Phillips Hue Bridge via API -- Phillips hue Bridge communicates with lights via Zigbee, lights update to appropriate values. 

If its not readily apparent- thats quite the communication change to turn on a light. Usually it happens near immediately, but occasionally there is a bit of a delay-- maybe up to a second or two. Irritating. It also sometimes fails outright, and I'll have to toggle the switch a second time. Lastly, if you toggle the switch back and forth quickly, it can get overloaded, and you'll have to wait a few seconds, then flip the switch again for changes to be made. Honestly, in day to day use, this rarely ends up being an issue, but its often enough that I'm wanting to figure out some way to optimize it further. 

The benefit of the original solution, which my brother-in-law Robert continues to use in a fork of Circadian Lighting, is that is significantly cuts down on the communication chain laid out in the image above. Instead of querying the circadian lighting sensor at the moment the light switch is toggled, a script is constantly monitoring the sensor, and updates a Hue Scene stored on the Hue Bridge with corresponding values via the Hue API whenever a change is detected. That way, whenever a light switch is toggled, it triggers the light to turn on to a particular scene, and the process is done. Much more efficient. 

So, consider with me- is there a perfect compromise?  Ideally, I would like the efficiency of Robert's approach, with the flexibility of mine. Minimal communication chain, as we've discussed in the two images above, without having to make any changes to Circadian Lighting's code and maintain an ongoing fork. Bonus points if its relatively flexible and could be expanded to the Adaptive Lighting integration as well. Can that happen? I think so. 

I'm envisioning configuring a Home Assistant automation which monitors the Circadian Lighting Sensor for changes. Whenever it sees a change, it would either work with the Hue API directly to update a scene with those values, or it would trigger a script which does the same thing. From there, the switch itself would be configured via Hue to simply turn the light on to that scene when toggled. 

If its that easy, why haven't I done it? Well, things are easy in theory. I am far from an expert on Home Assistant development, and I'm not yet whether its possible to get an automation to communicate directly with the Hue Bridge API. I think it should be possible via Restful commands, but its going to require some experimentation, and at least a few hours of trial and error. One of these days I'll get around to it, but in the meantime, I hope the method I've detailed here may be helpful to someone. Happy hacking! 