---
title:  "SSL server certificate via Keychain Access in macOS Catalina"
---

If you ever tried to create a CA and sign an SSL server certificate using
macOS Keychain Access you'd be surprised to find that resulting certificate
is not a valid SSL certificate anymore due to new
[requirements in iOS 13 and macOS 10.15](https://support.apple.com/en-us/HT210176).

<!--more-->

# Issuing a CA Certificate

- Open _Keychain Access | Certificate Assistant | Create a Certificate Authority_
- Update _Name_
- Change _User Certificate_ to _SSL Server_ and do not override defaults

> You may need to clean up
  `~/Library/Application\ Support/Certificate\ Authority` as macOS expects
  generated CAs to have unique names.

## Overriding defaults

It's not recommended to override defaults, it's way too easy to create a CA
that macOS would not like. But in case you want to do it anyway:

> Note: While this set of settings works, it's unclear which parts of this
instruction are important and which are not. Try on your own peril.

- _Certificate Information_
  - _Validity Period_ has to be ≤824
  - _Sign your invitation_ should not be used
   - You can set any custom _Name (Common Name)_ or other fields there
- _Key Usage Extension For This CA_
  - Check _This extension is critical_
  - Check _Key Encipherment_
- _Key Usage Extension For Users of This CA_
  - Uncheck
- _Extended Key Usage Extension For This CA_
  - Check _This extension is critical_
  - Check _SSL Server Authentication_
- _Extended Key Usage Extension For Users of This CA_
  - Uncheck
- _Basic Constraints Extension For This CA_
  - Check _Use this certificate as a certificate authority_
- _Subject Alternate Name Extension For This CA_
  - Uncheck
- _Subject Alternate Name For Users of This CA_
  - Uncheck
- _Specify a Location for The Certificate_
  - _Keychain_ should be _local_
  - Check _Trust certificates signed by this CA_

## Double-check CA trust

If your newly created certificate is not trusted (blue plus icon),
you need to open it and set _Trust | When using this_ to _Always Trust_.

# Issuing an SSL Certificate

- Open _Keychain Access | Certificate Assistant | Create a Certificate_
- Update _Name_
- Change _Identity Type_ to _Leaf_
- Change _Certificate Type_ to _SSL Server_ and click _Override defaults_

## Overriding defaults

You _have_ to override defaults to make this certificate compatible with macOS
Catalina.

> Note: While this set of settings works, it's unclear which parts of this
instruction are important and which are not. Try on your own peril.

1. _Certificate Information_
   - _Validity Period_ can not be ≥825
   - You can set any custom _Name (Common Name)_ or other fields there
1. _Choose An Issuer_
   - Pick newly created CA (should be the default one)
1. _Key Usage Extension_
   - You can uncheck _Include Key Usage Extension_, but it's not required
1. _Extended Key Usage Extension_
   - Check _Include Key Usage Extension_
   - Uncheck _This extension is critical_
   - Don't check any other _Capabilities_ except _SSL Server Authentication_
1. _Basic Constraints Extension_
   - Leave unchecked
1. _Subject Altenate Name Extension_
   - You can uncheck _This extension is critical_, but it's not required
   - Input your DNS names, may include `*` at the beginning for wild cards
     and can have many domains comma-separated. This is a required field.
   - Do not use _iPAddress_ field
1. _Keychain_
   - _login_

