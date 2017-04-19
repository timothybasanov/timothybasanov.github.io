---
title:  "Enable secure signed email on macOS and iOS"
---

Both *macOS* and *iOS* support secure signing for your email messages. It's easy to setup and it "just works".

If you ever receive a signed message from somebody you can send them *encrypted email*. Only your recipient will be able to read it. Even if your email account is compromised, this still holds true.

<!--more-->

# Getting free secure email certificate

Almost every certificate authority provides free secure email certificate, you can use [COMODO Secure Email Certificate](https://secure.comodo.com/products/frontpage?area=SecureEmailCertificate), it's as good as any.

  - Fill the form, provide your name and email
  - You'll get a confirmation email with a link for a certificate key
  - Download it and open to import into *Keychain*. Choose *Apache* if there is a choice of different certificate types

> You may get import errors, verify if your certs are imported anyway

# Done..?

That's it. Your email is now secured on your Mac.

> You have to restart your *Mail.app* to load these keys

  - When you create new email two additional buttons are visible: lock to encrypt email and a certificate to sign it. Emails are signed by default
  - To encrypt an email to somebody you have to receive a signed message from her first

> If you encrypt an email *subject* is still sent in clear text. Even you will not see a body of that email. When you'll receive encrypted mail you will only be able to read it on a device with certificate in a *Keychain*. Online? No luck.

# Enabling it on your iOS device

You'll need to install certificate on your iOS device first: [Install Certificates on iOS Device Securely via Custom Profile]({{ site.url }}/2016/04/26/ios-secure-certificate-installation.html). Include both your and COMODO's certificates in a profile.

  - Go to *Settings / Mail, Contacts, Calendars / Accounts / Your Account / Mail / Advanced*
  - Enable *S/MIME* and *Sign* with the right certificate (if you have many email addresses)

That's it. Now emails are signed properly when sent from this device. You will have an option to enable encryption per message later.
