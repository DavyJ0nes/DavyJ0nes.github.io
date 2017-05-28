+++
categories = ["how-to"]
comments = false
date = "2015-09-24T12:04:06+01:00"
draft = false
showcomments = true
showpagemeta = true
slug = ""
tags = ["debian", "cli"]
title = "Creating a Private Package repository for Debian"

+++


--------

**Caveat**

A lot of this article was nicked from a brilliant [Digital Ocean article](https://www.digitalocean.com/community/tutorials/how-to-use-reprepro-for-a-secure-package-repository-on-ubuntu-14-04).

Go check out all their [tutorials](https://www.digitalocean.com/community/tutorials) and while you're there why not [sign up!](https://cloud.digitalocean.com/registrations/new)

--------

## Why bother creating your own repository?

So you want to host your own .deb packages to make is simple for users to install without the need for git or building from source?

Read on...

This guide goes through how to set up a Debian flavoured package repository. We are using `reprepro` to install the packages, `gpg` for security and either `nginx` or `apache` to serve it to the world.


## Create and publish a Signing Key with GPG

#### 1. Generate a Master Key
    
We firstly need to install a tool to help with generating entropy

`apt-get install rng-tools`
   
Start rngd dameon manually using:
`rngd -r /dev/urandom`
   
By default rngd looks for a special device to retrieve entropy from /dev/hwrng. Some VMs do not have this device. To compensate we use the pseudo random device /dev/urandom by specifying the -r directive. 

Now we can generate the master key by invoking the gpg command like so:
`gpg --gen-key`

```
Please select what kind of key you want:
 (1) RSA and RSA (default)
 (2) DSA and Elgamal
 (3) DSA (sign only)
 (4) RSA (sign only)
Your selection? 1
```

By selecting the first option, gpg will generate first a signing key, then an encryption key.

```
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
```
    
The Debian project reccomends using 4096 bits for a signing key and Bigger is always better!
 
```
Please specify how long the key should be valid.
       0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y
```
    
Master keys do not normally have expiration dates, but set this value as long as you expect to use this key. If you only plan to use this repository for only the next 6 months you can specify 6m. 0 will make it valid forever.
    
```    
You need a user ID to identify your key; the software constructs the user ID
from the Real Name, Comment and Email Address in this form:
    "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"
    
Real name: Tom Celic
Email address: t@c.com
Comment: You know...
You selected this USER-ID:
    "Tom Celic (You know...) <t@c.com>"
    
Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
```
    
You will need to input a passphrase now. Ensure you **memorise this** as there is no way to recover a gpg key.

```
You need a Passphrase to protect your secret key.

Enter passphrase: (hidden)
Repeat passphrase: (hidden)
```

It will now create a load of entropy to generate the key. Once it is done you will have your master key.

```
gpg: key 231E2BB7 marked as ultimately trusted
public and secret key created and signed.
   
gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
pub   4096R/231E2BB7 2015-09-22
      Key fingerprint = AB5A 8BBB 2EFE F49B D36D  9554 BB48 C227 231E 2BB7
uid                  Tom Celic (You know...) <t@c.com>
sub   4096R/9C5F50AF 2015-09-22
```

Make a note of your signing key ID (231E2BB7 above). We will need this to create a subkey for it.


#### 2. Generate a Subkey to Sign Packages

Now we'll create a second signing key so that we don’t need the master key on this server. Think of the master key as the root authority that gives authority to subkeys. If a user trusts the master key, trust in a subkey is implied.

In the terminal execute:

`gpg --edit-key 231E2BB7`

You will then get the following output:

```
gpg (GnuPG) 1.4.16; Copyright (C) 2013 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

pub  4096R/231E2BB7  created: 2015-09-22  expires: never       usage: SC  
                     trust: ultimate      validity: ultimate
sub  4096R/9C5F50AF  created: 2015-09-22  expires: never       usage: E   
[ultimate] (1). Tom Celic (You know...) <t@c.com>
```

We now want to add a new key. type `addkey`

```
gpg> addkey
Key is protected.

You need a passphrase to unlock the secret key for
user: "Tom Celic (You know...) <t@c.com>"
4096-bit RSA key, ID 231E2BB7, created 2015-09-22

gpg: gpg-agent is not available in this session
```

We then want an RSA sign-only key that is 4096 bits long and doesnt expire. *ideally we would want to put an expiration on this but do not have process in place to look into maintainance*:

```
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
.....+++++
.+++++

pub  4096R/231E2BB7  created: 2015-09-22  expires: never       usage: SC  
                     trust: ultimate      validity: ultimate
sub  4096R/9C5F50AF  created: 2015-09-22  expires: never       usage: E   
sub  4096R/3648C586  created: 2015-09-22  expires: never       usage: S   
[ultimate] (1). Tom Celic (You know...) <t@c.com>
```

In the output above, the SC from our master key tells us that the key is only for signing and certification. The E means the key may only be used for encryption. Our signing key can be correctly seen with only the S. 

And finally we want to save. `gpg> save`


#### 3. Detach Master Key fro Subkey

The point of creating the subkey is so we don't need the master key on our server, which makes it more secure. Now we'll detach our master key from our subkey. We will need to export the master key and subkey, then delete the keys from GPG's storage, then re-import just the subkey.

First let's use the `--export-secret-key` and `--export` commands to export the whole key. Remember to use your master key's ID!

```
gpg --export-secret-key 231E2BB7 > private.key
gpg --export 231E2BB7 >> private.key
```

By default --export-secret-key and --export will print the key to our console, so instead we pipe the output to a new file (private.key). Make sure to specify your own master key ID, as noted above. 

**Make a copy of the private.key file somewhere safe**

**ONce you have backed up this file, kill it**

`rm private.key`

Now export your public key and your subkey. Make sure to change the IDs to match the master key and the second subkey that you generated (don't use the first subkey)

```
gpg --export 231E2BB7 > public.key
gpg --export-secret-subkeys 3648C586 > signing.key
```

Now we have backups of all keys we can remove the master key from the server.

`gpg --delete-secret-key 231E2BB7`

And then reimport only our signing subkey

`gpg --import public.key signing.key`

Now we want to clean up the keys:

`rm public.key signing.key`

And now we should publish the signing keyto keyserver.ubuntu.com

`gpg --keyserver keyserver.ubuntu.com --send-key 2648C586`

This command publishes your key to a public storehouse of keys – in this case Ubuntu’s own key server.
This allows others to download your key and easily verify your packages. 


## Set up Repository With Reprepro

### 1. Install and configure Reprepro

You can get reprepro from most debian flavoured repos

```
apt-get update
apt-get install reprepro
```

Configuration for Reprepro is repository-specific, meaning you can have different configurations if you make multiple repositories. Let's first make a home for our repository.

Make a dedicated folder for this repository and move to it.

```
mkdir -p /var/repos/
cd /var/repos/
```
Create a configuration directory
`mkdir conf && cd conf/`

Create two config file (options and distributions)
`touch options distributions`

Edit options with the following
```
ask-passphrase
verbose
```

Edit the distributions file with the following:

```
Origin: Ubuntu
Label: LiveLink 615 Repository
Codename: trusty
Components: main
Architectures: i386 amd64
SignWith: 3648C586
```

The Codename directive directly relates to the code name of the released Debian distributions and is required. This is the code name for the distribution that will be downloading packages, and doesn't necessarily have to match the distribution of this server. For example, the Ubuntu 14.04 LTS release is called trusty, Ubuntu 12.04 LTS is called precise, and Debian 7.6 is known as wheezy. This repository is for Ubuntu 14.04 LTS so trusty should be set here.

The Components field is required. This is only a simple repository so set main here. There are other namespaces such as “non−free” or “contrib” – refer to apt-get for proper naming schemes.

Architectures is another required field. This field lists binary architectures within this repository separated by spaces. This repository will be hosting packages for 32-bit and 64-bit servers, so i386 amd64 is set here. Add or remove architectures as you need them.

To specify how other computers will verify our packages we use the SignWith directive. This is an optional directive, but required for signing. The signing key earlier in this example had the ID 3648C586, so that is set here. Change this field to match the subkey’s ID that you generated. 


### 2. Add a Package with Reprepro

`reprepro -b /var/repos includedeb trusty package-name.deb`

enter passwords


### 3. Listing and Deleting

We can list the managed packages with the list command followed by the codename. For example: 

```
reprepro -b /home/vhosts/repositories/ list trusty

trusty|main|i386: example-helloworld 1.0.0.0
trusty|main|amd64: example-helloworld 1.0.0.0
```

To delete a package, use the remove command. The remove command requires the codename of the package, and the package name. For example:

`reprepro -b /home/vhosts/repositories/ remove trusty example-helloworld`


### 4. Make the Repository Public

This depends on whether the server you are hosting this from is running apache2 or nginx. Both methods will be explained

#### 1. Apache

Install Apache if not available

```
apt-get update
apt-get install apache2
```

Create new VirtualHost in `/etc/apache2/sites-available`

`vim repository.example.com`

Input the following config:

```
<VirtualHost *:80>
ServerName repository.example.com
ServerAlias www.repository.example.com
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-agent}i\" %I %O" combinedio
DocumentRoot /home/vhosts/repositories/

<Directory /home/vhosts/repositories/>
       Options Indexes FollowSymLinks MultiViews
        AllowOverride None
        Order allow,deny
        allow from all
</Directory>

<Directory "/home/vhosts/repositories/db/">
  Order allow,deny
  Deny from all
</Directory>

<Directory "/home/vhosts/repositories/conf/">
  Order allow,deny
  Deny from all
</Directory>

ErrorLog /home/vhosts/repositories/log/repo_error.log
LogLevel warn
CustomLog /home/vhosts/repositories/log/repo_access.log combinedio
```

Enable the site (this creates a symlink from sites-available to sites-enabled)

`a2ensite repository.example.com`

Reload the apache2 config, check it, then restart apache

```
service apache2 reload

apache2ctl configtest

service apache2 restart
```

#### 2. nginx

Install Ngnix if not available

```
apt-get update
apt-get install ngnix
```

Now let's create a new host in /etc/nginx/sites-available

`vim repository.example.com`

And enter the following config bits

```
server {
    listen       80;
    server_name  www.repository.example.com repository.example.com;
    root   /home/vhosts/repositories/;

    access_log /home/vhosts/repositories/log/repo.access.log
    error_log /home/vhosts/repositories/log/repo.error.log
    
    location ~ /(db|conf) {
      deny        all;
      return      404;
    }
}
```

Restart nginx

`service nginx restart`

Sort out DNS

Done, we are all good to go.


## Install Package from the Repo!

We are now ready to test that our new repo can be pulled from.

On the new server, download your public key to verify the packages from your repository. Recall that you published your key to keyserver.ubuntu.com.

This is done with the apt-key command.

`apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 2DB65792`

This command downloads the specified key and adds the key to the apt-get database. The adv command tells apt-key to use GPG to download the key. The other two arguments are passed directly to GPG. Since you uploaded your key to “keyserver.ubuntu.com” use the --keyserver keyserver.ubuntu.com directive to retrived the key from the same location. The --recv-keys <key ID> directive specifies the exact key to add.

Now add the repository's address for apt-get to find. You'll need your repository server's IP address from the previous step. This is easily done with the add-apt-repository program. 

`add-apt-repository "deb http://repository.example.com/ trusty main"`

The repository location should be set to the location of your server. We have an HTTP server so the protocol is http://. The example’s location was repository.example.com. Our server’s code name is trusty. This is a simple repository, so we called the component "main".

After we add the repository, make sure to run an apt-get update. This command will check all the known repositories for updates and changes (including the one you just made). 

Now run `apt-get update` and you will be good to go.


## References

- https://www.digitalocean.com/community/tutorials/how-to-use-reprepro-for-a-secure-package-repository-on-ubuntu-14-04
- https://www.debian-administration.org/article/286/Setting_up_your_own_APT_repository_with_upload_support
- https://www.debian-administration.org/article/513/Restrict_Access_To_Your_Private_Debian_Repository
- http://vincent.bernat.im/en/blog/2014-local-apt-repositories.html
- http://santi-bassett.blogspot.co.uk/2014/07/setting-up-apt-repository-with-reprepro.html
