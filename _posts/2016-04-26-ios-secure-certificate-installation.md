---
title:  "Install Certificates on iOS Devices Securely via a Custom Profile"
---

Installing certificates on iOS devices (at least on iPads and iPhones) can be done via a cable without using a less secure email-based transport. It's easy and free via [Apple Configurator 2](https://itunes.apple.com/us/app/apple-configurator-2/id1037126344).

<!--more-->

> As of iOS 10.3 newly installed root CA certificates need to be manually enabled in
_Settings | General | About | Certificate Trust Settings_

## Saving certificates on disk

*Apple Configurator* can only load certificates from disk, so you have to export them from *Keychain* if they are stored there.

  - Open *Keychain Access* application: `open -a "Keychain Access"`
  - Select certificates and export them in `.p12` format: <kbd>⇧⌘E</kbd>
  - As disk is a less secure storage using a password is recommended: `openssl rand -base64 12`

## Create a custom profile containing certificates

  - Install and run [Apple Configurator 2](https://itunes.apple.com/us/app/apple-configurator-2/id1037126344) from the AppStore
  - Create a new profile: <kbd>⌘N</kbd>
  - Set a human-readable name on a *General* page. This is what you'll see on your iPhone when you browse your profiles
  - Add certificates on a *Certificates* page
  - Close and save your profile (it even supports iCloud disk now!)

> If you don't specify `.p12` password in a configurator's profile you can ignore errors shown. When you choose to save a configurator's profile on disk this password is saved **in clear text**

## Install profile containing certificates on a device

  - Connect your device via *USB cable*
  - In *Apple Configurator* select your device and open *Profiles* tab, add profile you just created
  - You'll need to confirm profile installation on a device as device is not *managed*.
  - Profile itself is not signed, even if it contains only trusted and properly signed certificates

Voilà! Certificates are on your device bundled in one profile with your custom name.
