[<img width="250" height="100" src="https://habet.dev/wp-content/uploads/2021/12/Screen-Capture_select-area_20211219074622.gif"/>](https://habet.dev/)



# Self-hosting and securing web services out of your home with Argo Tunnel

Table of Contents

- [Why a tunnel?](#why-a-tunnel)
- [Cloudflare’s Argo Tunnel](#cloudflares-argo-tunnel)
- [Pre-requisites](#pre-requisites)
- [1\. Install](#1-install)
- [2\. Authenticate](#2-authenticate)
- [3\. Create tunnel](#3-create-tunnel)
- [4\. Create a configuration file](#4-create-a-configuration-file)
- [5\. Modify your DNS zone](#5-modify-your-dns-zone)
- [7\. Test the Tunnel](#7-test-the-tunnel)
- [8\. Convert to a system service](#8-convert-to-a-system-service)
- [Final Thoughts](#final-thoughts)

## Why a tunnel?

- Opening up a port on a home network isn’t the greatest idea. It is a security risk, a hole waiting for a hacker/bot to sneak in using a known vulnerability.
- Many [ISP](https://en.wikipedia.org/wiki/Internet_service_provider)‘s and routers prevent you from opening up ports 80 and 443.
- Another challenge you will face is setting up a dynamic DNS.
- Finally, when your domain is pointing to your home IP address it is very easy to determine your home location.

## Cloudflare’s Argo Tunnel

This is where [Cloudflare’s](https://www.cloudflare.com/) argo tunnel comes in. It builds a tunnel to cloudflares network. Now all request to your domain get directed through this tunnel to your web-server. all without opening any ports! and your domain simply points to cloudflare.

![cloudflare logo](https://habet.dev/wp-content/uploads/2021/12/cf-logo-v-rgb-rev.png)

## Pre-requisites

- Assumes you already have a domain name configured to use Cloudflare’s DNS service. This is a totally free service.
- Assumes you already have a Linux server with [Nginx](https://nginx.org/en/) and [Let’s Encrypt](https://letsencrypt.org/), listing on ports 80 and 443.

## 1\. Install

First we will get [cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation) installed as a package to get everything going. Then we will transfer it over to a service that automatically runs on boot in the background and establishes a tunnel.

```
$ wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
$ sudo dkpg -i cloudflared-linux-arm64.deb
```

*Note this link is for a raspberry pi, for other binaries see [here](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation).

This will create a folder in the home directory `~/.cloudflared`.

## 2\. Authenticate

Next, we need to authenticate with Cloudflare.

```
$ cloudflared tunnel login
```

This will generate a URL which will take you to login into your dashboard on Cloudflare.

## 3\. Create tunnel

```
$ cloudflared tunnel create <NAME>
```

Replace &lt;NAME&gt; with any name of your choice.

Running this command will:

- Create a tunnel by establishing a persistent relationship between the [name you provide](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-useful-terms#tunnel-name) and a [UUID](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-useful-terms#tunnel-uuid) for your tunnel. At this point, no connection is active within the tunnel yet.
- Generate a [tunnel credentials file](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-useful-terms#credentials-file) in the [default `cloudflared` directory](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-useful-terms#default-cloudflared-directory).

Confirm that the tunnel has been successfully created by running:

```
$ cloudflared tunnel list
```

## 4\. Create a configuration file

Now, create a [configuration file](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-useful-terms#configuration-file) in your `.cloudflared` directory. Using your favorite editor create a file `config.yml`, the content should look like this:

```
tunnel: <Tunnel-UUID>
credentials-file: /home/username/.cloudflared/<tunnel-UUID>.json
originRequest:
  originServerName: mydomain.com
ingress:
  - hostname: mydomain.com
    service: https://localhost:443
  - service: http_status:404 
```

A couple of things to note, here:

- Once the tunnel is up and traffic is being routed, Nginx will present the certificate for `mydomain.com` but `cloudflared` will forward the traffic to `localhost` which causes a certificate mismatch error. This is corrected by adding the `originRequest` and `originServerName` modifiers just below the credentials-file
- [Cloudflare’s docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/configuration/ingress) only provide examples for HTTP requests, and also suggests using the url `http://localhost:80`. Although Nginx can handle 80 to 443 redirects, our ingress rules and ARGO will handle that for us. It’s not necessary to include any port 80 stuff.
- if you want to host additional services via subdomain, just simply list them with port 443, like so:

```
 - hostname: subdomain1.mydomain.com
    service: https://localhost:443
  - hostname: subdomain2.mydomain.com
    service: https://localhost:443
```

Just insure the last line is `- service: http_status:404`.

## 5\. Modify your DNS zone

Now, we need to setup a CNAME for the TLD and any services we want. The `cloudflared` app handles this easily. The format of the command is:

```
$ cloudflared tunnel route dns <UUID or NAME> <hostname> 
```

Do this for each service you want (i.e., subdomain1, subdomain2, etc) hosted through ARGO.

![cloudflare dashboard](https://habet.dev/wp-content/uploads/2021/12/dns.png)

Cloudfalres DNS dashboard

## 7\. Test the Tunnel

Run the tunnel to proxy incoming traffic from the tunnel to any number of services running locally on your origin.

```
$ cloudflared tunnel run <UUID or NAME>
```

The above command as written (without specifying a config.yml path) will look in the default cloudflared configuration folder `~/.cloudflared` and look for a config.yml file to setup the tunnel.

If everything’s working, the end of the output should be:

```
...
...
<timestamp> INF Connection <redacted> registered connIndex=0 location=ATL
<timestamp> INF Connection <redacted> registered connIndex=1 location=IAD
<timestamp> INF Connection <redacted> registered connIndex=2 location=ATL
<timestamp> INF Connection <redacted> registered connIndex=3 location=IAD
```

Now, try to access your website and your service from outside your network – for example, a smart phone on cellular connection is an easy way to do this. If your webpage loads, SUCCESS!

## 8\. Convert to a system service

You’ll notice if you Ctrl+C out of this last command, the tunnel goes down! That’s not great! We want it up all the time. To do that we turn cloudflared into a service.

```
$ sudo cloudflared service install
```

Move the files to /etc/cloudflared

```
$ sudo mv ~/.cloudflared/* /etc/cloudflared/
```

Check ownership with `ls -la`, should be `root:root`.

Now we need to fix the config file, replace the line:

```
$ credentials-file: /home/username/.cloudflared/<tunnel-UUID>.json
```

with

```
$ credentials-file: /etc/cloudflared/<tunnel-UUID>.json
```

Then, start the system service with the following command:

```
$ sudo systemctl start cloudflared
```

And start on boot with:

```
$ sudo systemctl enable cloudflared
```

Check the status with:

```
$ sudo systemctl status cloudflared
```

The output should be similar to that shown in Step 7 above. You can safely delete your `~/.cloudflared` directory.

## Final Thoughts

We now have a web-server serving the internet without opening any ports on our network.

Credits: Reddit user : [u/highspeed_usaf](https://www.reddit.com/r/homelab/comments/pnto6g/how_to_selfhosting_and_securing_web_services_out/) and the cloudflare [docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide).

Published December 22, 2021By [admin](https://habet.dev/author/habetadmen/)

Categorized as [Self Hosted](https://habet.dev/category/self-hosted/), [Technology](https://habet.dev/category/technology/) Tagged [argo tunnel](https://habet.dev/tag/argo-tunnel/), [cloudflare](https://habet.dev/tag/cloudflare/), [DNS](https://habet.dev/tag/dns/), [self-hosted](https://habet.dev/tag/self-hosted/)



[Previous post<br>Raspberry Pi Encrypted Boot with SSH](https://habet.dev/raspberry-pi-encrypted-boot-with-ssh/)


## Categories

- [Self Hosted](https://habet.dev/category/self-hosted/)
- [Technology](https://habet.dev/category/technology/)
- [Uncategorized](https://habet.dev/category/uncategorized/)

- [Twitter](https://twitter.com/habet_dot_dev)
- [Reddit](https://www.reddit.com/user/abe-101/)
- [Email](mailto:admin@habet.dev)
- [Github](https://github.com/abe-101)
- [Linkedin](https://www.linkedin.com/in/abe101)
- [Tumblr](https://abe-101.tumblr.com/)
- [HN](https://news.ycombinator.com/user?id=abe-101)

![Abe's Portfolio](https://habet.dev/wp-content/uploads/2021/12/Screen-Capture_select-area_20211219074622.gif)

Proudly hosted under the computer desk on a [Raspberry Pi!](https://raspberrypi.org/).
