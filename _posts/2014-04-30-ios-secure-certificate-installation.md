---
layout: post
title:  "Install Certificates on iOS Device Securely via Custom Profile"
date:   2016-04-30
---

Installing certificates on iOS devices (at least on iPads and iPhones) can be done via a cable without using a less secure email-based transport. It's easy and free via [Apple Configurator 2](https://itunes.apple.com/us/app/apple-configurator-2/id1037126344).

# Saving certificates on disk

*Apple Configurator* can only load certificates from disk, so you have to export them from *Keychain* if they are stored there.

  - Open *Keychain Access* application: `open -a "Keychain Access"`
  - Select certificates and export them in `.p12` format: <kbd>Shift-Command-E</kbd>
  - As disk is a less secure storage using a password is recommended: `openssl -base64 10`

> When you save Apple Configurator's profile on disk this password is saved **in clear text**

# Create a custom profile containing certificates

  - Install and run [Apple Configurator 2](https://itunes.apple.com/us/app/apple-configurator-2/id1037126344) from the AppStore
  - Create a new profile: <kbd>Command-N</kbd>
  - Set a human-readable name on a *General* page. This is what you'll see on your iPhone when you browse your profiles
  - Add certificates on a *Certificates* page. You'll need password you used during certificate export to properly load it.
  - Close and save your profile (it even supports iCloud disk now!)
  
# Install profile containing certificates on a device

  - Connect your device via *USB cable*
  - In *Apple Configurator* select your device and open *Profiles* tab, add profile you just created
  - You'll need to confirm profile installation on a device as device is not *managed*.
  - Profile itself is not signed, even if it contains only trusted and properly signed certificates

Voilà! Certificates are on your device bundled in one profile with your custom name.
