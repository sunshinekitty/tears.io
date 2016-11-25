---
layout: post
comments: true
title: Caddy and Github Pages
preview: It was not immediately obvious to me on how to setup Caddy as a proxy to Github Pages, this was mostly due to a misunderstanding on how Github Pages work, but thought I’d share a quick guide to hopefully clear up any questions.
---

It was not immediately obvious to me on how to setup Caddy as a proxy to Github Pages, this was mostly due to a misunderstanding on how Github Pages work, but thought I'd share a quick guide to hopefully clear up any questions.

## Configuring Github Pages

I won't go into much detail here, a good guide to get started is located [here](https://guides.github.com/features/pages/).  What's important is to remember what you set your **Custom domain** as.  This will be the site you need to serve via Caddy.  In this example my **Custom domain** is set as **tears.io**.

## Configure Caddy

{% gist 71c9663f96d82fd3b65bc1674a03611f %}

The transparent setting in the proxy directive is short-hand for:

```
header_upstream Host {host}
header_upstream X-Real-IP {remote}
header_upstream X-Forwarded-For {remote}
header_upstream X-Forwarded-Proto {scheme}
```

Note that the proxy directive is pointing to my main Github Pages page, and not to the sub-page of `/tears.io`.  This is on purpose, Github Pages knows to serve the normal site as long as the `Host` header is set to `tears.io` in this instance.

That wraps things up, this shows how this blog is hosted from Github Pages at [github.com/sunshinekitty/tears.io](https://github.com/sunshinekitty/tears.io).
