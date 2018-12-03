---
title:  "Fronting Ubiquiti's UniFi Controllers with Nginx with SSL"
---

All Ubiquiti controllers (mFI, UniFi Controller and UniFi Video Controller)
insist on using HTTPS for all connections, which may break the experience
in Safari when one uses default out-of-the-box self-signed certificates.

Solution is simple: use Nginx to deal with SSL for most of the cases,
the rest is to use SSL via Java to make secure sockets work.


> File extensions are _very_ confusing when one is dealing with certificates.
In short if you use _OpenSSL_:
  - _PEM_ means ASCII plaintext, _DER_ means binary format, these only
    apply to `-inform` and `-outform` parameters
    (quoting [this article](http://info.ssl.com/article.aspx?id=12149))
  - [PKCS#8](https://wiki.openssl.org/index.php/Manual:Pkcs8(1))
    is a format for private keys, Java can read its _DER_ form. Use
    `-topk8` (to PK8 i.e. to PKCS#8) to output it. Use `-nocrypt`
    to disable key encryption enabled by default.
  - [PKCS#12](https://wiki.openssl.org/index.php/Manual:Pkcs12(1))
    is a format for a key bag, where several keys and
    certificates may be stored. Output is controlled with
    `-nodes` (no DES i.e. no encryption),
    `-nocerts` (skips public keys), and `-nokeys` (skips private keys)
  - [X.509](https://wiki.openssl.org/index.php/Manual:X509(1))
    is the default format for keys and certificates, can be read by Nginx.
    Can be directly concatenated in its _PEM_ form

<!--more-->

## Installing Nginx and Configuring SSL

> I assume that one uses NVR Ubiquiti box for all the controllers. That box
uses Debian, but most of these instructions would just apply to Ubuntu as well.

NVR has `nvr-webui` deb package, which has a dependency on `nginx-light`,
which has all the facilities to make all of this work. Otherwise one would
`apt-get install nginx` to get one on the box.

The easiest way to make Nginx to work is to add all the required keys and
certificates into a single decrypted X.509 PEM file, where private key
goes first.
I had `NVR_SSL.p12` PKCS#12 SSL certificate keys file and
`SSL_CA.cer` X.509 intermediate CA certificate (both binary files).

```sh
(
  openssl pkcs12 -nodes -in NVR_SSL.p12
  openssl x509 -inform der -in SSL_CA.cer
) > /etc/nginx/NVR_SSL_Chain.pem
```

> NVR has an Nginx 1.2.1 Light version preinstalled. A lot of Nginx config
options related to SSL and proxying were added much later on.
This article may inadvertently rely on
[newer Debian's version from backports]({% post_url 2017-09-08-ubuquiti-nvr-unifi-mfi-install %}#add-ubiquitis-debian-repositories).
Install with: `apt-get -t wheezy-backports install nginx-common nginx-light`

### Configuring Nginx reverse proxy

 - Ubiquiti insists on using HTTPS ports, so we proxy pass to HTTPS ports
 - Nginx by default does not validate proxy SSL certificates
 - Ubiquiti software relies on WebSockets a lot
   (see [Nginx & WebSockets](https://www.nginx.com/blog/websocket-nginx/))
 - Ubiquiti software uses X443 ports for HTTPS and X080 ports for HTTP

Overall it's a standard Nginx SSL reverse-proxy configuration:

```txt
cat > /etc/nginx/sites-enabled/ssl-proxy <<EOF

server {
        listen                  443 ssl;
        server_name             mfi.home.timothybasanov.com;

        ssl_certificate         /etc/nginx/NVR_SSL_Chain.pem;
        ssl_certificate_key     /etc/nginx/NVR_SSL_Chain.pem;
        ssl                     on;
        ssl_protocols           TLSv1.2;

        location / {
                proxy_pass              https://localhost:6443;
                proxy_set_header        Host $host;
                proxy_set_header        Upgrade $http_upgrade;
                proxy_set_header        Connection "upgrade";
                proxy_http_version      1.1;
        }
}
server {
        listen                  443 ssl;
        server_name             nvr.home.timothybasanov.com;

        ssl_certificate         /etc/nginx/NVR_SSL_Chain.pem;
        ssl_certificate_key     /etc/nginx/NVR_SSL_Chain.pem;
        ssl                     on;
        ssl_protocols           TLSv1.2;

        location / {
                proxy_pass              https://localhost:7443;
                proxy_set_header        Host $host;
                proxy_set_header        Upgrade $http_upgrade;
                proxy_set_header        Connection "upgrade";
                proxy_http_version      1.1;
        }
}
server {
        listen                  443 ssl;
        server_name             unifi.home.timothybasanov.com;

        ssl_certificate         /etc/nginx/NVR_SSL_Chain.pem;
        ssl_certificate_key     /etc/nginx/NVR_SSL_Chain.pem;
        ssl                     on;
        ssl_protocols           TLSv1.2;

        location / {
                proxy_pass              https://localhost:8443;
                proxy_set_header        Host $host;
                proxy_set_header        Upgrade $http_upgrade;
                proxy_set_header        Connection "upgrade";
                proxy_http_version      1.1;
        }
}

EOF
```

### Fix for a UniFi Video controller's secure WebSockets

 - UniFi Video uses `wss://hostname:7446` for secure web sockets
 - Direct WSS URL is passed from a server via XHR API call
 - We can not find and replace it
 - Nginx can not intercept it as port is already bound
 - _Solution:_ We need to add our SSL certificates to UniFi-Video server

> To proxy this call via Nginx one needs to either have a separate IP
for Nginx server or to fix the url passed from API in-flight. I did not want
to deal with the additional IP address. As for in-flight replacement
it requires a module from `nginx-full` deb package,
which conflicts with `nginx-light`,
which is required for `nvr-webui` which I still rely on.

Ubiquiti published all the instructions in their
[UniFi Video 3.7.0 release notes](https://community.ubnt.com/t5/UniFi-Video-Blog/UniFi-Video-3-7-0-Release/ba-p/1934006):

  1. Stop the service: `/etc/init.d/unifi-video stop`
  1. Clean up the outdated key store:
     `rm /usr/lib/unifi-video/data/ufv-truststore`
  1. Enable support for custom certificates, an experimental feature:
     `echo ufv.custom.certs.enable=true >> /usr/lib/unifi-video/data/system.properties`
  1. Create a directory for new certificates:
     `mkdir /usr/lib/unifi-video/data/certificates`
  1. Generate X.509 DER certificates file:
     ```sh
     # Generate it from the original files
     (

       openssl pkcs12 -nodes -nokeys -in NVR_SSL.p12
       openssl x509 -inform der -in SSL_CA.cer
     ) | openssl x509 -outform DER \
       > /usr/lib/unifi-video/data/certificates/ufv-server.cert.der

     # Alternatively read back Nginx configs
     openssl x509 -outform DER -in /etc/nginx/NVR_SSL_Chain.pem \
       > /usr/lib/unifi-video/data/certificates/ufv-server.cert.der
     ```
  1. Generate PKCS#8 DER private key file:
     ```sh
     # Generate it from the original files
     openssl pkcs12 -nodes -nocerts -in NVR_SSL.p12 \
       | openssl pkcs8 -topk8 -nocrypt -outform DER \
       > /usr/lib/unifi-video/data/certificates/ufv-server.key.der

     # Alternatively read back Nginx configs
     openssl pkcs8 -topk8 -nocrypt -outform DER -in /etc/nginx/NVR_SSL_Chain.pem \
       > /usr/lib/unifi-video/data/certificates/ufv-server.key.der
     ```
  1. Fix up the permissions:
     ```sh
     chmod 600 /usr/lib/unifi-video/data/certificates
     chown -R unifi-video:unifi-video /usr/lib/unifi-video/data/certificates
     ```
  1. Start the service: `/etc/init.d/unifi-video start`
  1. Verify `less /var/log/unifi-video/server.log` log to not to have
     exceptions about a key import format. If there are issues, start
     from the first step all over
  1. Clean up: `rm -rf /usr/lib/unifi-video/data/certificates`

> There are several things that Ubiquiti may fix to make our life easier:
  - Allow certificates import directly via Web UI
  - Allow binding to `localhost` and using HTTP instead of HTTPS
  - Add support for `X-Remote-IP` headers, now
    all logins are logged coming from Nginx's IP address `127.0.0.1`
  - Make their deb packages depend on `nginx-light` **or** `nginx-full`

Done. Now everything should work just fine. I verified it to work
even in Safari!
