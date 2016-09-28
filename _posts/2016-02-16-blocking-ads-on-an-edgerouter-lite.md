---
layout: post
comments: true
title: Blocking (Hulu) ads on an EdgeRouter Lite
preview: I recently picked up a Ubiquiti EdgeRouter Lite, of course the first thing I wanted to do was make sure there weren't any ads on my network.  I found [this article](https://help.ubnt.com/hc/en-us/articles/205223340-EdgeMAX-Ad-blocking-content-filtering-using-EdgeRouter) which did the bulk of the work for me, unfortunately for me some [shows](http://southparkstudios.com/) I watch are streamed through Hulu.
---

I recently picked up a Ubiquiti EdgeRouter Lite, of course the first thing I wanted to do was make sure there weren't any ads on my network.  I found [this article](https://help.ubnt.com/hc/en-us/articles/205223340-EdgeMAX-Ad-blocking-content-filtering-using-EdgeRouter) which did the bulk of the work for me, unfortunately for me some [shows](http://southparkstudios.com/) I watch are streamed through Hulu.  Fortunately there is a way to get around this, Hulu expects an ad video to play but using dnsmasq you can redirect requests to their ad servers to a local server which can play a short clip of your choosing.

This guide is going to include the steps in the article I linked above, in addition to taking the steps to setup a fake ad-video-playing server.

## Configure the 'blackhole' web server
Our requirements for the webserver are for it to redirect any requests to `/path/*` (for mobile/consoles) to a short video clip.

For the purpose of this guide I'm going to assume you have a vm or machine that you can run nginx on, I prefer not to run an http server on the router to avoid adding any additional load.  The commands in this article are geared towards RHEL/Centos, but the configuration files should be inter-changeable between distros.

### Install nginx

```bash
yum install -y nginx
```

### Configure nginx

Place the following in `/etc/nginx/conf.d/adblock.conf`:

```
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;

    location = / {
        try_files $uri /404.html;
    }

    location ~ /.*(.avi|.fla|.mp4|.ogg|.mpeg|.mpg|.wmv|.flv) {
        try_files $uri /media/black.mp4;
    }
}
```

Remove the following from `/etc/nginx/nginx.conf`:

```
    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
```

Start/Restart and Enable nginx

```bash
systemctl restart nginx
systemctl enable nginx
```

Now we need to get our dummy black video.  I googled for a 5 second black clip and found [this](https://www.youtube.com/watch?v=xE0Bp6ENy_8) then converted it to .mp4 using [this](http://www.clipconverter.cc/).

Download the file to `media/black.mp4`

```bash
mkdir /usr/share/nginx/html/media
wget -O /usr/share/nginx/html/media/black.mp4 "http://link/to/file"
```

Verify in  your browser that things work as expected, any URL's matching `/path/*` should redirect you to a video, anything else a 404 page.

## Configure the router
I expanded on the script in the article referenced above.  This one is written in python and grabs more ad lists and also adds in some redirects for hulu ad URL's to your server you configured in the section above.  Place the following script in `/config/user-data/update-adblock-dnsmasq.py`.  You need to replace `video_server` with the IP address of the server you configured above.  You can add any websites you want to whitelist to the `ignore_list` section.

{% gist 458fb2c899effe53954b018f00c0bccd %}

Now make the script executable

```bash
chmod +x /config/user-data/update-adblock-dnsmasq.py
```

Try executing it

```bash
/config/user-data/update-adblock-dnsmasq.py
```

Verify it's working from a machine on your network, your output should be similar:

```bash
curl secure-us.imrworldwide.com/path/

curl measuremap.com
```

Add in a cron job to update the general ad blocking every Sunday at 4AM:

```bash
sudo crontab -e
```

```
<...>
0 4 * * 6  /config/user-data/update-adblock-mdns.py
```

All set!
