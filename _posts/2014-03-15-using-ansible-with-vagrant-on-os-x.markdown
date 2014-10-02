---
title: Using Ansible with Vagrant on OS X
---
This weekend I decided to switch-over a project of mine from using Vagrant's shell provisioner to Ansible's. For those not familiar, Vagrant is a tool that is designed to create virtual machines for developing against. Ansible is a provisioning and orchestration tool that manages deploying and configuring software on multiple machines. The set up was a little trickier than I would've liked.

The overall steps involved were installing Ansible on the host, which caught me off-guard as I was expecting it to just be required on the guest. This involves doing some compiling. Once it was installed, I needed to tweak the defaults in Vagrant somewhat to get everything to play nicely. The steps required are:

1. Ensure you have an up-to-date XCode install with the command-line developer tools
2. Install Pip if you haven't already (`sudo easy_install pip`)
3. Install Ansible with Pip. Normally this would just be `sudo pip install ansible`. This step is way harder than it needs to be since Apple recently shipped an update to Clang configured to error out when it receives command line arguments it doesn't understand. Unfortunately, Python on OS X was built with a few of these arguments and extensions built against it require those same arguments. The workaround I used for this was to set the ARCHFLAGS environment variable during the Pip install run to not raise that error. On the command line it looks like, `sudo ARCHFLAGS=-Wno-error=unused-command-line-argument-hard-error-in-future pip install ansible`.
4. In your `Vagrantfile`, make sure you set `sudo` to `true` in your Ansible provisioner configuration. You will get a lot of permission errors if you don't.
