---
title: How to Enable Filesharing between Linux Mint 11 and Windows
---
This post will be rather short and to the point. For whatever reason, out of the box, Linux Mint 11's public folders in each user's home folder are not accessible to Windows clients and the process to make them accessible is easy enough but not obvious. The first step is to enable WebDAV by installing some Apache packages. The second step is to enable file sharing on the public folder. The final step is to enable execute permissions for everyone on your home folder.

Here are the terminal commands to install the required packages and set the appropriate permissions:

1. `$ sudo apt-get install apache2.2-bin libapache2-mod-dnssd`
2. `$ chmod 0711 ~`

Now go into the _Preferences_ menu and select -Personal File Sharing_. In the dialog that pops up, enable the checkbox next to _Share public files on network_ and close the dialog. Finally, right-click on the _Public_ folder in your home folder and enable the checkbox next to _Share this folder_ and enable the checkbox next to _Guest access (for people without a user account)_ and then close the dialog. You should now be able to access your Linux public folder from Windows.

Unfortunately it's defaults like this that hurt Linux usability on the desktop and I hope this gets fixed in future versions.
