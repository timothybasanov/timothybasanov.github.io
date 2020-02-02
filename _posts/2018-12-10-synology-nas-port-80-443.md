---
title:  "Opening ports 80 and 443 on a Synology NAS via UI"
---

Synology NAS's OS DSM 6.x does not allow one to directly use
ports 80 and 443. There are several different articles
flowing around that try to cover it, but they often
require SSH access to a box and may not be compatible
with future versions of DSM.
Turns out there is "one weird trick" to make it all work
via built-in Nginx server relying on "Host" headers from HTTP requests.

Verified on DS718+ DSM 5.2.1 with  Ubiquity mFI controller
and Homebridge (for HomeKit) running within Docker containers on Synology.
As a side effect now I can use DSM without ugly port numbers (e.g. :5001)
via a redirection below.

<!--more-->

> I did not have Web Station installed and I did not have Synology-managed
Apache/PHP servers. I have a hope that this would work for most other
Synology setups, but I have not tried it myself. Redirecting Synology's
own host to a different port (:5000) may break applications like Photo Station,
but I do not use them and have not verified if they were broken.
I have only tried iOS _DS Finder_ app and it worked.

## Synology NAS Ports & Applications

These are main ports of interest:
 - 80: Runs Synology-controlled Nginx to redirect HTTP users to :5000
 - 443: Same Nginx to redirect HTTPS users to :5001
 - 5000: Main Synology DSM UI on HTTP
 - 5001: Main Synology DSM UI on HTTPS

> There is a lot of things going on with :80 and :443 on Synology.
Some "applications" (e.g. photo station and friends) may use this port
instead of :5000 or :5001. I've tried to reconfigure it via SSH
and Nginx config file, but there
are many auto-generated config files so it's too fragile for my taste.

> If you noticed that you're still being redirected to port :5001, check
that you disabled _Automatically redirect HTTP connections to HTTPS
 _Control Panel \| Network \| DSM Settings_. While you're connecting to
 port :443, Nginx proxies your connection to :5000, which may force DSM to send
 you a 302 redirect to :5001.

### Using Reverse Proxy to Control Nginx on :80/:443

 * Open _Control Panel \| Application Portal \| Reverse Proxy_
 * Create a new proxy rule
 * Set _General_ options:
   - Source Protocol: _HTTPS_
   - Source Hostname: `your-synology-hostname-you-use-in-a-browser`
   - Source Port: `443`
   - Destination Protocol: _HTTP_
   - Destination Hostname: `localhost`
   - Destination Port: `5000`
 * Set _Custom Header_ options:
   - Choose _Create | ▼ | WebSocket_ to create: `Upgrade` `$http_upgrade`
     and `Connection` `$connection_upgrade`
 * Check that _Advanced Settings_ has _HTTP 1.1_ enabled

> If you don't specify _Source Hostname_ it shows a misleading error:
"This port number is reserved for system use only. Please enter a different
number." It's only allowed to redirect traffic via a Host header.

> I recommend to have a separate mapping for port 80 as well, even if you
would never use it. Otherwise if some application redirects you to HTTP
version of a web site it would load default contents, which is DSM Web UI
and this is super-confusing.

Surprisingly this effectively updates Nginx configuration
in `/var/tmp/nginx/app.d/server.ReverseProxy.conf` with (simplified):

```javascript
server {
        listen                  443 ssl;
        server_name             your-synology-hostname-you-use-in-a-browser;

        ssl_certificate         /usr/syno/etc/certificate/ReverseProxy/$GUID/fullchain.pem;
        ssl_certificate_key     /usr/syno/etc/certificate/ReverseProxy/$GUID/privkey.pem;

        location / {
                proxy_pass              http://localhost:5000;
                proxy_set_header        Host $host;
                proxy_set_header        Upgrade $http_upgrade;
                proxy_set_header        Connection "upgrade";
                proxy_http_version      1.1;
        }
}
```

> You only need `Upgrade` and `Connection` headers if web application uses
web sockets, I'm adding it just in case. You also need to explicitly
have `Host` header, otherwise some apps tend to redirect you incorrectly.

### Adding HTTPS Certificates

You may have noticed that above a `fullchain.pem` and `privkey.pem` files
are mentioned. They are managed separately in
_Control panel | Security | Certificate_ panel.

You need to add your own certificate and _Configure_ to connect
_your-synology-hostname-you-use-in-a-browser_ with a cert you've uploaded.
This way you can support as many virtual hosts on a Synology NAS as you need
as long as you've configured your DNS to point to it.

Synology is very picky about certificates:
 * Only a certificate, a key and an (optional) intermediate CA are supported
 * Exactly one key and/or certificate per file
   - This prevents you from using more than one intermediate CA and makes it
     difficult to use your own CA authority if you have complex chains
   - To make things harder when something goes wrong error messages are cryptic
     at best
 * Certificates and keys are only accepted in X.509 PEM form
   - Use `openssl x509 -inform der -in certificate.cer` to convert
     certificate from X.509 DER form (`*.cer`) produced by
     Keychain Access in macOS
   - Keys are only accepted in X.509 PEM form, use
     `openssl pkcs12 -nodes -in key.p12` to convert from PKCS#12 DER
     form produced by Keychain Access in macOS
   - [Read my other post on Nginx/Certs/SSL](https://timothybasanov.com/2017/09/24/ubiquiti-nvr-unifi-mfi-nginx-ssl-proxy.html)
