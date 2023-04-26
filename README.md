# Host Gitea on Google Cloud in Minimal Ubuntu LTS (Example)

In over one year hosting Gitea on Google Cloud, I have enjoyed the self-hosted Git service.

Creating an instance, setting it up, and securing the service was an amazing adventure. Enhancing the instance was, well, tricky but rewarding. Gitea itself is, indeed, (arguably!) painless: I have only encountered two issues, issues which I think even an enthusiast can fix with a little knowledge of SQLite3.

What follows is documentation of how I got Gitea up and running on Google Cloud. Hopefully, it can help others who want to host Gitea on Google Cloud, too. Perhaps, one could even adapt my work to host Gitea with other cloud providers.

Enjoy!

## Table of Contents

* [Part I: Create an Instance](#part-i-create-an-instance)
* [Part II: Set Up the Instance](#part-ii-set-up-the-instance)
* [Part III: Additional Measures to Secure Gitea](#part-iii-additional-measures-to-secure-gitea)
* [Parts IV-VI: Enhance the Instance](#part-iv-enhance-the-instance)
* [Part VII: Upgrading Packages, Collections and Ubuntu, and Retesting Functionality](#part-vii-upgrading-packages-collections-and-ubuntu-and-retesting-functionality)
* [Part VIII: Troubleshooting](#part-viii-troubleshooting)
* [Part IX: Privacy](#part-ix-privacy)
* [Part X: Options](#part-x-options)

## Part I: Create an Instance

Let's get some formalities *squared away*...

> Use Google's Chrome web browser for the best experience. Google Cloud webpages tend to time out or fail to load in Safari.

### Enable 2-Step Verification

[https://myaccount.google.com/security &#128279;](https://myaccount.google.com/security)

### Create a New Project

[https://console.cloud.google.com/iam-admin/iam &#128279;](https://console.cloud.google.com/iam-admin/iam)

### Create a Billing Account, and Link It to the Project

[https://console.cloud.google.com/billing/ &#128279;](https://console.cloud.google.com/billing/) <!--Cloud Shell commands currently in beta: https://cloud.google.com/sdk/gcloud/reference/beta/billing/projects/link -->

### Enable APIs

- [x] Compute Engine API: [https://console.cloud.google.com/apis/library/compute.googleapis.com &#128279;](https://console.cloud.google.com/apis/library/compute.googleapis.com)
- [x] Domains API: [https://console.cloud.google.com/marketplace/product/google/domains.googleapis.com &#128279;](https://console.cloud.google.com/marketplace/product/google/domains.googleapis.com)
- [x] DNS API: [https://console.cloud.google.com/marketplace/product/google/dns.googleapis.com &#128279;](https://console.cloud.google.com/marketplace/product/google/dns.googleapis.com)

Now, let's work on creating the instance...

### Compute Engine

#### Access It

Activate Cloud Shell

>  Click on the command prompt icon at the top-right corner of your web browser.

Click on the down arrow next to "Terminal," and select the auto-generated project ID.

#### Open Ports

Port 80

```bash
gcloud compute firewall-rules create default-allow-http \
	--direction INGRESS \
	--rules tcp:80 \
	--action ALLOW
```

Port 443

```bash
gcloud compute firewall-rules create default-allow-https \
	--direction INGRESS \
	--rules tcp:443 \
	--action ALLOW
```

Port 3000

```bash
gcloud compute firewall-rules create gitea-setup \
	--direction INGRESS \
	--rules tcp:3000 \
	--action ALLOW
```

> Port 80 will redirect HTTP requests to the HTTPS address, and port 3000 will only be used to set up Gitea.

#### Create an Instance

Check for the latest release of Minimal Ubuntu LTS

```bash
gcloud compute images list \
  --filter ubuntu-minimal
```

> Example:
>
> NAME: ubuntu-minimal-2204-jammy-v20220928 \
> PROJECT: ubuntu-os-cloud \
> FAMILY: ubuntu-minimal-2204-lts \
> DEPRECATED: \
> STATUS: READY

Create the instance

```bash
gcloud compute instances create gitea \
	--machine-type e2-small \
	--image ubuntu-minimal-2204-jammy-v20220928 \
	--image-project ubuntu-os-cloud \
	--scopes compute-rw,cloud-platform \
	--tags http-server,https-server \
	--shielded-secure-boot \
	--zone us-west1-b
```

>  e2-micro could also work. See [Gitea system requirements &#128279;](https://docs.gitea.io/en-us/#system-requirements). \
>  Additional machine types: [https://cloud.google.com/compute/all-pricing &#128279;](https://cloud.google.com/compute/all-pricing) \
>  Additional zones (and regions): [https://cloud.google.com/compute/docs/regions-zones &#128279;](https://cloud.google.com/compute/docs/regions-zones)

Reserve the external IP address

```bash
gcloud compute addresses create gitea-address \
	--addresses 34.168.233.62 \
	--region us-west1
```

### Cloud Domains

[https://console.cloud.google.com/net-services/domains/registrations/list &#128279;](https://console.cloud.google.com/net-services/domains/registrations/list)

Register `mydomain.dev` (or any domain that equally requires an SSL certificate)

### Cloud DNS

[https://console.cloud.google.com/net-services/dns/ &#128279;](https://console.cloud.google.com/net-services/dns/)

Add a Type A DNS record

TTL: `360` `minutes` \
IPv4 Address: `34.168.233.62`

### Back to the Compute Engine

Reconnect to Cloud Shell

#### Back Up the Instance 

Create a Snapshot

```bash
gcloud compute snapshots create snapshot-1 \
	--project myproject \
	--source-disk gitea \
	--description "created instance" \
	--source-disk-zone us-west1-b \
	--storage-location us
```

> If you ever need to restore it, go to [https://console.cloud.google.com/compute/instances &#128279;](https://console.cloud.google.com/compute/instances). \
> Stop the instance; edit it by replacing its boot disk with the snapshot; and re-start the instance.
>
> *In fact, trying that now might be a good idea; learn now to mitigate any potential downtime later.*
>
> <!--make sure source disk is the disk the instance is using-->

## Part II: Set Up the Instance

The instance is created. Now, let's work on setting up the instance...

### Compute Engine (Continued)

#### Connect to the Instance

[https://console.cloud.google.com/compute/instances &#128279;](https://console.cloud.google.com/compute/instances)

Click on "SSH," and unblock the pop-up window. (You may have to reclick on "SSH.")

##### Check and Upgrade Packages

Check for the latest packages

```bash
sudo apt update
```

Upgrade them

```bash
sudo apt upgrade
```

> If asked to reboot, run: `sudo reboot`. \
> Wait a moment, then retry connecting to the instance.

##### Install and Set Up Packages

###### Nano

Install a command line text editor

```bash
sudo apt install nano
```

> We will use nano to set up packages.
> Feel free to see an [overview of nano's shortcuts &#128279;](https://www.nano-editor.org/dist/latest/cheatsheet.html).

###### Caddy

Install a web server

[https://caddyserver.com/docs/install#debian-ubuntu-raspbian &#128279;](https://caddyserver.com/docs/install#debian-ubuntu-raspbian)

> We will use caddy to obtain the required SSL certificate (Part I) and redirect the HTTP requests to the HTTPS address (Part I).

Set up the web server

```bash
sudo nano /etc/caddy/Caddyfile
```

Input:

```bash
mydomain.dev {
	# root * /usr/share/caddy
	# file_server
	reverse_proxy localhost:3000
}
```

Restart the web server

```bash
sudo systemctl restart caddy
```

###### Gitea

Install the Git service

[https://gitlab.com/packaging/gitea/ &#128279;](https://gitlab.com/packaging/gitea/)

> Run the last command with `sudo`. That is, `sudo systemctl enable --now gitea` \
> If you prefer installing Gitea manually, consult [https://docs.gitea.io/en-us/install-from-binary/ &#128279;](https://docs.gitea.io/en-us/install-from-binary/)  <!--I used to install and upgrade the binary, but it's tricky to install and upgrade.--> \
> Gitea is available as a snap package, unfortunately you cannot modify the logo, home page or theme.

Set up the service

[http://34.168.233.62:3000 &#128279;](http://34.168.233.62:3000)

Database Type: `SQLite3` \
Server Domain: `mydomain.dev` \
Gitea Base URL: `https://mydomain.dev/`
<!--also changed site title: VCS Portal-->

*Email Settings*

SMTP Host: `smtp.gmail.com` \
SMTP Port: `465` \
Send Email As: `example@gmail.com` \
SMTP Username: `example@gmail.com` \
SMTP Password: `***` \
&check; Enable email notifications

> Port 465 uses recommended Implicit TLS. See [https://docs.gitea.io/en-us/email-setup/ &#128279;](https://docs.gitea.io/en-us/email-setup/) \
> For the SMTP Password, use an [App Password &#128279;](https://support.google.com/accounts/answer/185833?hl=en). This requires two-step verification.
> <!--"Send Email As" is redundant-->

*Server and Third-Party Service Settings*

&cross; Enable OpenID Sign-In \
&check; Disable Self-Registration

*Administrator Account Settings*

Administrator Username: `john.doe` \
Password: `***` \
Confirm Password: `***` \
Email Address: `example@gmail.com`

> Please use a strong password.
> Administrator email address *can* be different than SMTP username.

Test Caddy and Gitea

Check [https://mydomain.dev &#128279;](https://mydomain.dev)

> You may have to reload the web browser tab. \
> <b>Caution: <u>Do NOT log into mydomain.dev.</u> The link is purely an example. Use your own domain.</b>

Check email notifications

Sign In > Site Administration > Configuration > Send Testing Email 

> For creating additional user accounts, see Part IX.

Back to Cloud Shell

Create another snapshot...

## Part III: Additional Measures to Secure Gitea

Enabling two-step verification, registering a domain that requires an SSL certificate, disabling OpenID sign-in, disabling self-registration, and using a strong administrator password help secure Gitea by restricting access to it. 

However, we can do better...

### Compute Engine (Continued)

#### Close Port 3000

Reconnect to Cloud Shell

```bash
gcloud compute firewall-rules delete gitea-setup
```

#### Block Brute Force Attacks

Back to the SSH browser window

Enable logs

```bash
sudo nano /etc/gitea/app.ini
```

Input:

```ini
[log]
MODE = file
```

Restart the service

```bash
sudo systemctl restart gitea
```

Check for logs

```bash
sudo cat /var/lib/gitea/log/gitea.log
```

> Time stamps use Coordinated Universal Time.

Install iptables

```bash
sudo apt install iptables
```

Install crowdsec <!--I used to install fail2ban-->

[https://docs.crowdsec.net/docs/getting_started/install_crowdsec &#128279;](https://docs.crowdsec.net/docs/getting_started/install_crowdsec)

> Run the installation commands with `sudo`.

Enable Gitea support

```bash
sudo cscli collections install LePresidente/gitea
```

Parse Gitea logs

```bash
sudo nano /etc/crowdsec/acquis.yaml
```

Append:

```yaml
filenames:
	- /var/lib/gitea/log/gitea.log
labels:
	type: gitea
```

Reload crowdsec

```bash
sudo systemctl reload crowdsec
```

Test it

Go to [https://mydomain.dev &#128279;](https://mydomain.dev). Create several failed authentication attempts until the website times out. Check `sudo cscli decisions list`, and you should see an ID (e.g., 9098) with your IP address banned. Either wait, or unban yourself: `sudo cscli decisions delete --id 9098`. Re-attempt to sign in.

#### Mitigate the Risk of Cross-Regional Outages

Reconnect to Cloud Shell

```bash
gcloud compute instances add-metadata gitea \
	--metadata VmDnsSetting=ZonalOnly
```

Back to the SSH browser window

```bash
sudo dhclient -v -r
```

#### Mitigate Snapshot Restoration Error

Periodically, if one attempts to restore a snapshot they will encounter an error message: 'Operation type [insert] failed with message "The zone...does not have enough resources available to fulfill the request. Try a different zone, or try again later.'

Reconnect to Cloud Shell

```bash
gcloud compute reservations create my-reservation \
	--zone=us-west1-b \
	--vm-count=1 \
	--machine-type=e2-small 
```

#### Checklist

Double-check that all of these measures have been taken...

- [x] 2-Step verification (Google account)
- [x] stable image (Ubuntu Server LTS)
- [x] secure boot (enabled when created instance)
- [x] reserved external IP address (did after created instance)
- [x] HTTPS only (.dev domain requires an SSL certificate)
- [x] DNSSEC (Cloud DNS)
- [x] create snapshots (doing after completing each part)

> We will schedule snapshots in Part VI.

- [x] practice restoring instance from snapshot (did in Part I)
- [x] SSL certificate (Caddy)
- [x] reverse proxy (Caddy)
- [x] SMTP Implicit TLS (port 465)
- [x] &cross; OpenID sign-in
- [x] &cross; self-registration
- [x] strong administrator password
- [x] closed port 3000
- [x] protected from brute-force attacks (crowdsec)
- [x] mitigated risk of cross-regional outages (Part II)
- [x] mitigated snapshot restoration error

In fact, I like that Ubuntu packages&mdash;except for Node.js (Part IV)&mdash;are relatively up-to-date, not *bleeding edge* / potentially unstable as on Arch Linux and not potentially outdated as on Debian or RHEL.

> Keep in mind...
>
> - These measures are not exhaustive
> - You don't have to customize the logo, favicon, home page, and theme (Parts IV-V)

Back to Cloud Shell

Create another snapshot...

## Part IV: Enhance the Instance

PHEW! The instance is finally set up. Now, let's work on enhancing the instance...

### Compute Engine (Continued)

> Caution: The following instructions will not work with Gitea's snap package. See Part II for workable installation instructions.

#### Customize the Logo and Favicon

Back to the SSH browser window

Install Node.js

[https://github.com/nodesource/distributions/blob/master/README.md &#128279;](https://github.com/nodesource/distributions/blob/master/README.md)

> You will need the latest LTS version of Node.js, and look for "Node.js LTS" \
> The version of Node.js packaged with Ubuntu will be outdated.
>
> *The latest LTS version is available as a snap package, unfortunately when I installed it and ran `make generate-images` (below), I received an error: "Not implemented: HTMLCanvasElement.prototype.getContext" (see also [issue #20157 &#128279;](https://github.com/go-gitea/gitea/issues/20157)).* 

Prepare logo and favicon you want

~~Start with the logo. Export the PNG or JPG/JPEG image file to a SVG vector file. Then, scale the vector file to, say 48px x 48px, and export that. Name these vector files as `logo.svg` and `favicon.svg`, respectively.~~

Focus on the logo. Export the PNG or JPG/JPEG image file to a SVG vector file. Name the vector file as `logo.svg`. Then, duplicate the file and rename the duplicate as `favicon.svg`.


> Suggestion: Use vector graphics software (e.g., Inkscape or Affinity Designer). For Inkscape, export image files to <b>Inkscape SVG</b>. For Affinity Designer, export image files to <b>SVG (for export)</b>. Pixelmator Pro, an image editor, also works by ~~exporting image files~~ converting areas in image files to shapes, grouping the shapes and exporting the group to simply <b>SVG</b>.

Copy the logo and favicon you want from your local computer to the instance

> Click on UPLOAD FILE at the top edge of the pop-up window. \
> Upload logo.svg and favicon.svg.

Install make

```bash
sudo apt install make
```

Clone Gitea's source repository

```bash
git clone https://github.com/go-gitea/gitea.git gitea-source
```

Replace the logo and favicon in the cloned repository's assets directory with yours

```bash
mv logo.svg gitea-source/assets/ && \
mv favicon.svg gitea-source/assets/
```

Generate new images for your logo and favicon

```bash
cd gitea-source/ && \
make generate-images
```

> You can ignore any "deprecated" warnings.

Be patient! Generating the images may take several minutes.

Locate the images

```bash
cd public/img
```

> Other SVG export formats could work, as well, but with Inkscape SVG or SVG (for export) the `make generate-images` command will generate the images completely. With other export formats, the command may not do so, for example with Affinity Designer's SVG (digital - high quality) preset, the command did not generate logo.png and favicon.png.

Create an "img" directory in the working directory of your instance, and move everything into it

```bash
export WD=/var/lib/gitea/ && \
sudo mkdir -p $WD/custom/public/img/ && \
sudo mv * $WD/custom/public/img/
```

Restart the service

```bash
sudo systemctl restart gitea
```

Check [https://mydomain.dev &#128279;](https://mydomain.dev)

> If you are using either the Firefox or Chrome desktop web browser, the favicon should change.
>
> However, if you are using the Safari desktop web browser, you will need to quit Safari, empty the favicon cache, and re-launch Safari. \
> To empty the favicon cache, launch Finder > Go > Go to Folder... > `~/Library/Safari/Favicon Cache/`. Select all items in the folder, move them to the trash, and empty the trash.

Back to Cloud Shell

Create another snapshot...

## Part V: Enhance the Instance (Cont.)

We have customized the logo and favicon. Now, let's customize the home page and theme...

### Compute Engine (Continued)

> Caution: Again, the following instructions will not work with Gitea's snap package. See Part II for workable installation instructions.

#### Customize the Home Page

Back to the SSH browser window

Clone Gitea's source repository

> See Part IV.

Checkout the latest release

```bash
cd gitea-source/ && \
git checkout f48fda8eefa4d47e335f01ac92366b9373950e0e
```

> Locate the latest release by going to [https://github.com/go-gitea/gitea/releases &#128279;](https://github.com/go-gitea/gitea/releases) (e.g., v1.17.3) \
> Copy its commit hash (e.g., f48fda8eefa4d47e335f01ac92366b9373950e0e)
>
> *Make sure that the Gitea service version (Part II) matches the release version. Otherwise, either upgrade the Gitea service (Part II) or checkout an earlier release.*

Modify the "home" template

```bash
nano templates/home.tmpl
```

Leave only the logo, app name, and app description:

```xml
<!--
	<div class="ui stackable middle very relaxed page grid">
	...
	</div>
	<div class="ui stackable middle very relaxed page grid">
	...
	</div>
-->
```

<!--also input a disclaimer-->

Create a "templates" directory in the working directory of your instance, and move the "home" template into it

```bash
export WD=/var/lib/gitea/ && \
sudo mkdir $WD/custom/templates/ && \
sudo mv templates/home.tmpl $WD/custom/templates/
```

Restart the service

```bash
sudo systemctl restart gitea
```

Check [https://mydomain.dev &#128279;](https://mydomain.dev) 

#### Customize the Theme

Select one from [Awesome Gitea &#128279;](https://gitea.com/gitea/awesome-gitea/src/branch/main/README.md#user-content-themes) (e.g., Red Silver, or a fork of it)

Clone its repository <!--cloned my own fork-->

```bash
git clone https://github.com/iamdoubz/Gitea-Red-Silver.git
```

Navigate to the public/css directory

```bash
cd Gitea-Red-Silver/public/css
```

> You can ignore the public/img directory, unless you want to overwrite any customized logo and favicon.

Create an "css" directory in the working directory of your instance, and move the CSS file into it

```bash
export WD=/var/lib/gitea/ && \
sudo mkdir -p $WD/custom/public/css/ && \
sudo mv theme-redsilver.css $WD/custom/public/css/
```

As stated in the README, make the theme selectable, and make it the default

```bash
sudo nano /etc/gitea/app.ini
```

Input:

```ini
[ui]
THEMES = auto,gitea,arc-green,redsilver
DEFAULT_THEME = redsilver
```

> You can also remove the original theme color by inputting: `THEME_COLOR_META_TAG = none` \
> Otherwise, when you scroll down or up, areas above or below each webpage will still be colored green.

Restart the service

```bash
sudo systemctl restart gitea
```

Check [https://mydomain.dev &#128279;](https://mydomain.dev)

Sign In > Settings > Appearance > redsilver > Update Theme 

Back to Cloud Shell

Create another snapshot...

## Part VI: Enhance the Instance (Cont.)

We have customized the home page and theme. Now, let's apply finishing touches...

### Compute Engine (Continued)

#### Schedule Snapshots

Back to Cloud Shell

Create snapshot schedule

```bash
gcloud compute resource-policies create snapshot-schedule gitea-snapshots \
    --project myproject \
    --region us-west1 \
    --max-retention-days 14 \
    --on-source-disk-delete keep-auto-snapshots \
    --daily-schedule \
    --start-time 18:00 \
    --storage-location us \
    --description "regular and automatic back up"
```

> Start time is in Coordinated Universal Time (UTC).
>
> 18:00 UTC is late morning in California.

Attach the schedule to the instance

```bash
gcloud compute disks add-resource-policies gitea \
    --resource-policies gitea-snapshots \
    --zone us-west1-b
```

> Snapshots make Gitea's [backup and restore commands &#128279;](https://docs.gitea.io/en-us/backup-and-restore/) redundant. I am grateful for this, as well, because for me the backup command never completely worked (e.g., avatars and repo-avatars were never backed up).

## Part VII: Upgrading Packages, Collections and Ubuntu, and Retesting Functionality

We have applied finishing touches. Now, let's discuss additional resources...

### Compute Engine (Continued)

#### Upgrading Nano, Caddy, Gitea, Iptables, Crowdsec, Crowdsec Collections, Node.js, Make, etc.

Back to the SSH browser window

```bash
sudo apt update
sudo apt upgrade
sudo cscli collections upgrade --all
sudo systemctl reload crowdsec
```

> If asked to reboot, run: `sudo reboot`. \
> Wait a moment, then retry connecting to the instance.

##### Excluding Ubuntu Packages

Useful, if you want to skip one newer package version: \
`sudo apt-mark hold [package]`

To re-commence upgrades: \
`sudo apt-mark unhold [package]`

#### Upgrading Ubuntu Releases

> Not for Ubuntu LTS point releases!

Back to Cloud Shell

Open port 1022

```bash
gcloud compute firewall-rules create recovery \
	--direction INGRESS \
	--rules tcp:1022 \
	--action ALLOW
```

Back to the SSH browser window

```bash
do-release-upgrade
```

Back to Cloud Shell

Close port 1022

```bash
gcloud compute firewall-rules delete recovery
```

After upgrading any packages, collections or the operating system, retest functionality...

#### Retesting the Domain

Check [https://mydomain.dev &#128279;](https://mydomain.dev)

#### Retesting Brute Force Protection

Back to the SSH browser window

Reset Crowdsec's metrics

```bash
sudo systemctl reload crowdsec
```

Create a failed authentication attempt at [https://mydomain.dev &#128279;](https://mydomain.dev)

Check Crowdsec's metrics

```bash
sudo cscli metrics
```

> "gitea-logs" should be parsed

Except for two issues (Part VIII), simply retesting the domain and brute force protection have worked for me, but *your mileage may vary*.

## Part VIII: Troubleshooting

If you run into any issues...

### Compute Engine (Continued)

#### Troubleshooting the Domain

Back to the SSH browser window

##### Check Gitea

```bash
sudo systemctl status gitea --no-pager --full
```

If the problem(s) exists here, consult the example below. (I will add additional examples, if any new problems arise.)

###### Example: *"UNIQUE constraint failed: webauthn_credential.lower_name, webauthn_credential.user_id"*

Let's say the lower_name = yubikey (and user_id = 21)

SOLUTION

Install sqlite3

```bash
sudo apt install sqlite3
```

Change user

```bash
sudo su gitea
```

Edit the database

```bash
sqlite3 /var/lib/gitea/data/gitea.db
```

Remove the row in the database with that lower_name (and user_id)

```sqlite
SELECT lower_name FROM webauthn_credential;

DELETE FROM webauthn_credential
WHERE lower_name = yubikey;
```

> You could also have used the user_id, but lower_name will delete both.

Exit the database

```sqlite
.quit
```

Otherwise, check to see if anyone submitted an issue for the problem(s): [https://github.com/go-gitea/gitea/issues &#128279;](https://github.com/go-gitea/gitea/issues). Gitea support staff may reply to the issue. If no issue is submitted, you can either submit one yourself, join Gitea's [Discord chat &#128279;](https://discord.com/invite/Gitea), or participate in their [Discourse forum &#128279;](https://discourse.gitea.io).

You can also check Gitea's logs

```bash
sudo cat /var/lib/gitea/log/gitea.log
```

If the problem(s) exists elsewhere, then check Caddy...

###### Check Caddy

```bash
sudo systemctl status caddy --no-pager --full
```

> You can safely ignore any "context canceled" errors. See ["Aborting with incomplete response" &#128279;](https://caddy.community/t/aborting-with-incomplete-response/10990).

In over one year hosting Gitea on Google Cloud, though, I have only encountered two issues, both of which involved Gitea and occurred after upgrading it (one required removing a webauthn_credential, and the other required a downgrade)...neither involved Caddy, nor Ubuntu, nor any other package. (I just started using Crowdsec.)

#### Troubleshooting a Security Vulnerability

EXAMPLE

["gitattributes parsing integer overflow" &#128279;](https://github.com/git/git/security/advisories/GHSA-c738-c5qq-xg89)

SOLUTION

Install software-properties-common

```bash
sudo apt install software-properties-common
```

Add Ubuntu Git Maintainers' Personal Package Archive (PPA)

```bash
sudo add-apt-repository ppa:git-core/ppa
```

Run

```bash
sudo apt update && \
sudo apt install git
```

#### Troubleshooting Images Not Loading and Being Logged Out

MP4 files do not load properly, and&mdash;for whatever reason&mdash;you will be logged out.

SOLUTION

Replace MP4 files with GIF files.

## Part IX: Privacy

Protect users...

### Compute Engine (Continued)

[https://mydomain.dev &#128279;](https://mydomain.dev) 

#### Hide Administrator Visibility ~~and Activity~~

Administrator visibility ~~and activity~~ can reveal ~~users and their activity~~ them self

- [x] Administrator Account > Settings > Profile > User visibility: Private
~~- [x] Administrator Account > Settings > Profile > &check; Hide the activity from the profile page~~ (redundant)

#### Hide Regular Users' Visibility ~~and Activity~~

Regular users' visibility ~~and activity~~ can reveal themselves ~~and their activity~~

- [x] Administrator Account > Site Administration > User Accounts > Create/Edit User Account > User visibility: Private

> Users who want to be discoverable by private users can use Limited visibility. ~~Just be sure you hide activity from your profile page.~~ (optional) \
> Users who want to be discoverable by ALL users (even external ones) can use Public visibility.

> Users who want their repositories to be discoverable and forkable by private users must use Limited visibility and make their repositories Public. \
> Users who want their repositories to be discoverable and forkable by ALL users (even external ones) must use Public visibility and make their repositories Public.

#### Disable Stars

Starring repositories can reveal users ~~and their activity~~

Edit Gitea's configuration file

```bash
sudo nano /etc/gitea/app.ini
```

Input:

```ini
[repository]
DISABLE_STARS = true
```

Restart Gitea

```bash
sudo systemctl restart gitea
```

#### Test Privacy Settings

Create a dummy user account(s).

Log into it to see what other users see.

~~#### Caution~~

~~Following other users can also reveal users and their activity~~

~~- [x]  Caution users about following other users~~

~~(Fixed in Gitea 1.17.4. See also go-gitea/gitea#21849 &#128279;. Of course, followed users can still inadvertently reveal followers.)~~

#### Suggestion

- [x] ALL users enroll in Two-Factor Authentication

## Part X: Options

Quality of life improvements...

### Compute Engine (Continued)

#### Autocompletion

*Can help you remember commands and run commands more quickly*

Back to the SSH browser window

Create and edit a file named .inputrc

```bash
nano $HOME/.inputrc
```

Input exactly:

```bash
"\e[A": history-search-backward
"\e[B": history-search-forward
```

Start a new session

```bash
bash
```

Test autocompletion by typing some characters you previously typed, then pressing the up/down arrow key.

Repeat these steps for other users (e.g., gitea) <!--also did root-->

> For the gitea user, first run the command: `sudo su gitea` \
> To return to your service account, run: `exit`

#### Enable Push to Create for Users

*Can help if users already have a local repository set up*

Edit Gitea's configuration file

```bash
sudo nano /etc/gitea/app.ini
```

Input:

```ini
[repository]
ENABLE_PUSH_CREATE_USER = true
```

Restart Gitea

```bash
sudo systemctl restart gitea
```

All done!

<!-- also removed snapd -->

## Extended Resources

* Affinity Designer: [https://affinity.serif.com/en-us/designer/ &#128279;](https://affinity.serif.com/en-us/designer/)
* Gitea Documentation: [https://docs.gitea.io/en-us/ &#128279;](https://docs.gitea.io/en-us/)
* Google Cloud Documentation: [https://cloud.google.com/docs &#128279;](https://cloud.google.com/docs)
* Inkscape: [https://inkscape.org &#128279;](https://inkscape.org)
* Minimal Ubuntu: [https://canonical.com/blog/minimal-ubuntu-released &#128279;](https://canonical.com/blog/minimal-ubuntu-released)

## Future Research

* Potentially hosting Gitea on Google Cloud in Minimal Ubuntu LTS <u>on ARM</u>

## Known Issue

- [ ] For Gitea 1.19.0+, Safari users who have multiple tabs open (e.g., Gitea repo in one tab and Gitea wiki in another) may be abruptly logged out. ~~Current workaround: Disable "Prevent cross-site tracking."~~ (Workaround failed.) Replacing MP4 files with GIF files partially helped, but users still eventually get logged out. Clearing browsing history temporarily works. (See [issue #24176 &#128279;](https://github.com/go-gitea/gitea/issues/24176). ~~Mitigating issue by using Google Chrome.~~ Clicking on "Remember This Device" at login worked. A fix is coming: [pull request #24330](https://github.com/go-gitea/gitea/pull/24330)