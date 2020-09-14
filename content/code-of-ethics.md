---
title: "Code of Ethics"
date: 2020-09-10T22:53:28-07:00
draft: false
---

I believe that you have the right to know how my code honors your privacy and guarantees equal access to all users regardless of ability. In this document, I’ve made every reasonable effort to disclose the mechanics of my website that may affect you and explain my rationale for the way this site is built.

If you think something is missing here, or if you believe I'm not adhering to these principles consistently, please reach out to me. I will make correcting the issue my top priority.

## Progressive enhancement

This website prioritizes the principle of **Progressive Enhancement.** Any user, regardless of web browser, should be able to access and consume the core functionality of this website. Users with more recent browsers or devices may notice upgraded interactions, but the content is fundamentally the same.

## Privacy, tracking, and analytics

Thanks to Netlify, this site always uses HTTPS. It utilizes a single privacy-respecting analytics tool, **Simple Analytics.** You can verify this by taking a look at my [source code](https://github.com/samlinville/).

Simple Analytics doesn’t use cookies or collect any personal data. It respects your browser’s Do Not Track preferences, and doesn’t employ any kind of fingerprinting through private data. [Read this document from Simple Analytics](https://docs.simpleanalytics.com/what-we-collect) to understand more about the data that is collected when you visit my website.

While I’m sure that Netlify logs some data on their servers when you visit a site that they host, I do not have access to these logs and do not know what information is collected.

### A note on third-party embedded media

Sometimes, I embed content and media from other websites, like YouTube or SoundCloud. Because this content is externally hosted, I often don’t have control over the way that content collects your data. While I don’t have access to the data they collect, I see it as my responsibility to minimize the invasion of your privacy when you are viewing content through my website. Where possible, I have enabled privacy modes on embedded content.

In the future, I plan to block these embedded content blocks from rendering on page load. Instead, you'll be able to choose whether to load those pieces of content individually.

## Security compliance

Maintaining a web server is hard work! To eliminate the possibility that I'll forget to apply an important security update or rotate keys on time, I've decided that this website will be hosted on a static site platform— Netlify, in my case. My Netlify account uses strict multi-factor authentication to prevent someone from gaining unauthorized access.

I have made every reasonable effort to minimize my reliance on third-party code, but I do use a few reputable libraries to make my life easier and your experience better. If you’re curious about which code libraries have been included in my website, you can take a look at my [source code](https://github.com/samlinville/).

## Performance

All assets in this site are minified and compressed to the greatest practical extent.

Furthermore, I do my best to ensure that your device downloads appropriately-sized images to prevent unnecessary data ingress on your device.

## Accessibility

I am committed to ensuring that all users can access my content equally, regardless of ability.

This site makes every reasonable effort to use semantic HTML markup that allows screen readers to properly navigate and interpret the content. Text content on this website also adheres to WCAG 2.0 AA standards for contrast.

Images on this website are always captioned descriptively via the `alt` attribute.

If you believe that any of my content falls short of these standards for accessibility, please reach out to me. I will make fixing the error my highest priority.