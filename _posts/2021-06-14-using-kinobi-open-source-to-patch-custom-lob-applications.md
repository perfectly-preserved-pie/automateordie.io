# Context
We use JAMF at our company but had a problem with keeping our line-of-business (LOB) applications updated. [JAMF's Patch Management catalog](https://docs.jamf.com/jamf-app-catalog/Patch_Management_Software_Titles.html), while extensive, doesn't include applications hiding behind a vendor's paywall: VPN clients, NAC agents, AV/EDR agents, etc. 
In our case, one of those applications is Palo Alto GlobalProtect. While the application has its own auto-upgrade mechanism, I'm using it here as an example of what you can do with Kinobi Open-Source.

# External patch sources
Aside from JAMF's internal catalog (which is free), JAMF also allows you to configure an external patch source, which is really just a [webserver that can respond to specific API requests](https://www.jamf.com/jamf-nation/articles/497/jamf-pro-external-patch-source-endpoints). That JAMF article explains the API and JSON structure needed for a functioning external patch source. An external patch source will allow you to create a software title, definitons, etc. for _any_ application and therefore let you update _any_ application; you're not just limited to whatever's in the JAMF catalog. The downside is that _you_ have to define the application, its requirements, its versions, etc. (and keep them updated!).
To get a hint of what you _can_ (but sometimes don't need) to define, take a look at [Kinobi's page on the topic](https://mondada.atlassian.net/wiki/spaces/MSD/pages/553189450/Patch+Definitions).

# Introducing Kinobi Open-Source (not self-hosted, not cloud)
That JAMF article is pretty intimidating if you've never worked with APIs or JSON in general. Thankfully, we can stand on the shoulder of giants and use the solutions created by people who have already done the impressive work before us. One of these solutions is Kinobi Open-Source.
Kinobi is an external patch definition server that comes in 3 different flavors:
* Kinobi Cloud, a cloud-hosted patch definition server for Jamf Pro with access to a Kinobi subscription and JSON importer.
* Kinobi Self-Hosted, a self-hosted patch definition server for Jamf Pro with access to a Kinobi subscription and JSON importer.
* Kinobi Open-Source, an open-source patch definition server for Jamf Pro.

The first two (Cloud and Self-Hosted) require a paid subscription to use. The last one, Open-Source, is truly free and is what we'll be using here.

# Installing Kinobi Open-Source
## System Requirements
Installation is pretty simple; the installer is a .run script that can run on 
* Ubuntu LTS Server 14.04 or later (18.04 recommended)
* Red Hat Enterprise Linux (RHEL) 6.4 or later
* CentOS 6.4 or later

See the full system requirements [here](https://github.com/mondada/kinobi#standalone).

## Actually installing it
[There's already an installation guide provided by Mondada so I'll just link it here.](https://mondada.atlassian.net/wiki/spaces/MSD/pages/592216069/Kinobi+Open-Source)

### Don't install Kinobi on the same server hosting your JAMF Pro instance! (if you're not using JAMF Cloud)
If you install Kinobi on the JAMF Pro server, Kinobi won't start; it uses port 443 which is already in use on a JAMF Pro server so the port bind will fail. You can check the status of Kinobi's Apache webserver with `sudo systemctl status apache2.service`.
You'll see an error message similiar to "Bind: Address Already in Use".

For that reason, it's best to spin up a new VM/server and install only Kinobi on that.

# How to manually add a software title
And now we get to the meat of it 🍖
Mondada has [an excellent guide](https://mondada.atlassian.net/wiki/spaces/MSD/pages/553222153/Manual+Creation) on how to create your first software title.
In my case, I'll be creating one for Palo Alto GlobalProtect.

## GlobalProtect example
Click on the "New" button to start the process, then fill out the information specified.
* **Name**: this shows up in the JAMF GUI.
* **Publisher**: this shows up in the JAMF GUI.
* **Application Name**: optional. Skip it.
* **Bundle Identifier**: optional. Skip it.
* **Current Version**: What's the latest version of this software you have in your environment? Put that here. It'll be used in the patch reports as the latest version.
* **ID**: This is just an internal reference for Kinobi. You can make it anything you want but I tend to just put the application name.
![Starting the process with some basic information](https://i.imgur.com/1u6dsQy.png)

When you're finished, click Save. You've just created your first software title! 🎉 Now we need to add a "requirement".

### What's a Requirement (for a software title)?
Simply put, a requirement (per Kinobi) is "_Criteria used to determine which computers in your environment have this software title installed._"
The syntax, form, and structure is exactly the same as a smart group or advanced search in JAMF. Ever created one of those? Maybe you've created a smart group or advanced search based on things like
* ARM or x86_64 CPU architecture
* Presence of an installed application
* Operating System Version
* Extension Attribute value
* etc.

It's the same process for our software title requirement. You can make the requirement any of a number of different criteria but in general, *your requirement should be the easiest way to detect the presence of the application on a computer*; that'll usually be "Application Title". 
1. Click the "Requirements" tab.
2. Click the "Add" button to begin creating a requirement.
3. Click the drop-down menu and select "Application Title" and click Save.
![Requirements](https://i.imgur.com/mlJ5g8g.png)

4. Then change the **operator** to "is" and the **value** to "GlobalProtect.app"
![Criteria](https://i.imgur.com/XOWuSvI.png)


### Patches/Patch Definitions
Now that we've created the base software title and its requirement, we need to create some "sub-items" that Kinobi calls "patches" and JAMF calls "patch definitions". Because we're working with Kinobi, I'll refer to them as patches from now on. Both terms refer to the same thing: software title version information. For example, each of these GlobalProtect versions would be its own patch (and each patch can have its own possibly different requirements):
* GlobalProtect v5.0.8
* GlobalProtect v5.1.5
* GlobalProtect v5.2.6
* and so on.

#### Creating a patch in Kinobi
1. Click the "Patches" tab (next to the "Requirements" tab).
2. Click the "New" button.
3. Fill out the fields as shown:
* **Sort Order**: how should this appear in Kinobi. Purely cosmetic.
* **Version**: Version associated with this patch. 
* **Release Date**: optional. You can set this to the actual release date in your environment, the vendor's release date, or just leave it as default.
* **Standalone**: if this patch will need to be installed incrementally, select No.
* **Reboot**: Does the application patch need a reboot to complete the process?
* **Minimum Operating System**: Self-explanatory.

![New patch version](https://i.imgur.com/dlL6VKt.png)

## ⚠ A patch for **each** app version is needed otherwise they all show up as "Unknown"
This is important! You'll need to add a patch for _every version_ (other than the latest version, which you've already added) of your application that currently exists in your environment. In this case, I added patches for some of the older GlobalProtect versions:



If you _don't_ define patches for every version, they'll show up as an "Unknown" version in the JAMF Patch Management report:
![Undefined patch versions](https://i.imgur.com/WcF7k6F.png)


As you can see, that report isn't very helpful; although I defined the latest version (5.2.6) the other versions floating around show up as "Unknown". For this reason I strongly urge you to take the time and create a patch for each app version.

Anyways, at this point, you're done! 🎉 You've successfully installed Kinobi, connected to JAMF, and created your first software title & associcated patch(es).

## 🤔 Using Kinobi seems like a lot of clicking around in the GUI. Isn't there a way to to automate this?
You're right! It gets annoying especially if you have lots of apps. Luckily, Kinobi allows you to manually upload JSON definitions for patches.
_But how do you automatically generate those JSON files_? That's where [Patch Starter Script](https://github.com/brysontyrrell/Patch-Starter-Script) comes into play. It's a Python script that will spit out a fully defined JSON file for you for any application. Then you can just upload that JSON into Kinobi. But that deserves its own blog post in the future! I'll link to the new post once I've finished setting PSS up in my own environment and finish the write up.
