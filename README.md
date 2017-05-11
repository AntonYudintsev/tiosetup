# Setting up Try it Online

## What is Try It Online

<https://tryitonline.net> is a community-maintained web site for hosting solutions to [code golf](https://en.wikipedia.org/wiki/Code_golf) puzzles presented on <http://codegolf.stackexchange.com/>. It can be used by anyone for free to quickly try out and share code snippets in a big number of practical and recreational programming languages.

## Setup Overview

These instructions are written to help set up a new instance of <http://tryitonline.net>.

TIO currently uses 2 domains which are hosted on one main server. There are also one or more load balanced arena servers, which are accessed from the main server via private network. The github repositiory for them is <https://github.com/TryItOnline/tryitonline.git>

- Main server split across 2 domains:
  - tryitonline.net - serves the front page, provides some assets for the rest of the sites
  - tio.run - TIO Nexus and TIO v2 PoC front end, short url for permalinks (permalinks are PoC only)
- Arena servers - sandbox where the actual code is executed, running on SeLinux and accessed by the tio.run via SSH

The main server setup instruction will get you to a single server which has both main server and arena server files on it. You will be able to run TryItOnline in this single server configuration, but this is not recommended. Instead you should follow arena setup instructions, and create one or more arena nodes. These arena nodes get synchronised with the main server setup using rsync. Once you have arena nodes initialized, you can point your main to them and disable using local arena files on mine (see below).

The setup makes use of [SeLinux](https://en.wikipedia.org/wiki/Security-Enhanced_Linux), support of which varies between different distributions of Linux.

The [talk.tryitonline.net](http://talk.tryitonline.net) domain is set up to redirect to Stack Exchange chat about the tryitonline service. Setting this domain up is not covered by this guide.

This setup runs on **Fedora 25**. It was tested on the Linode<https://www.linode.com/> hosting providers. *Note: as of the time of writing TryItOnline is hosted on Digital Ocean, but the move is planned to Linode soon.

It is assumed that you have a freshly provisioned instance(s) on Linode. All scripts are run as root. It might be possible to run the setup on another hosting provider, such as [Digital Ocean](https://www.digitalocean.com/) or [Vultr](https://www.vultr.com/), however minor image differences may require slightly different prep steps, which are not covered here. You will need minimum of 1024MB of memory in any case.

## Domain registration and certificates

In order to setup TIO, you will need the following sub-domains created within your domains. Note that these are the ones that TIO itself uses, yours, obviously will be different:

- tryitonline.net
- tio.run

You will need to provide your domain names to the setup scripts.

Our setup scripts use [LetsEncrypt](https://letsencrypt.org/) for SSL certificates, which is free service. You are welcome to use your own certificates, but you will need to update the setup scripts accordingly.
LetsEncrypt uses [Certbot](https://certbot.eff.org/) for generating SSL certificates and configuring apache to use them. In order for this to work, the domains that you are going to be using in your setup has to point to the the VM IP address that you are running the setup scripts on.
Thus, a recommended sequence would be following:

- Register one or more domains to use with TIO
- Provision a TIO main server machine
- Point the three domains (your versions of tryitonline.net and tio.run and) to the VM IP

After that Certbot will be able to validate the fact that it's you who are controlling these domains, and generate the certificates for you.
Note, that LetsEncrypt limits you to 5 sets of certificates per week for each domain combination you use. Thus, if you are going to run the setup scripts multiple times (for testing), it is advisable to `tar czf letsencrypt.tar.gz /etc/letsencrypt` after the first run of the setup scripts, so that you can reuse these certificates on the next run, and thus avoid hitting the rate limit.

## Generating ssh keys

There are three key pairs that is involved in TryItOnline setup:

- "apache" keypair which is used for authenticating main against arena when TryItOnline executes user's code
- "root" keypair which is used for authenticating main against arena for rsync, which is kicked off for either initial arena setup or for updating arena from main
- "host" keypair which is used in the both cases above to authenticate arena against main

Note, that main and all arenas will have all three keypairs above. These are in addition to keypairs you may use for your own SSH access to any of the servers.

During setup the apache and the root keypairs will get generated if not provided. The host keypair is always assumed to be present because having an openssh server usually implies that these are already generated.
Still, since during arena initialization (which is performed from main via SSH), the arena keys must match those on main, you may want to provide per-generated keys during provisioning so that you can be sure that the keys you are installing to the arena and those you are installing to main match.

Use `ssh-keygen` command on Linux to generate  private and public keys as required. You can use rsa or ed25519.pub keys.

## Dyalog APL

To run Dyalog APL you will need a licence and a 64-bit Dyalog installation image for Linux. TIO has such a licence. If you choose not to get a licence and thus do not have an installation image, the TIO installation script for Dyalog APL will generate errors which can be safely ignored. Dyalog is free for non-commercial usage; you can apply for a licence at <http://dyalog.com/prices-and-licences.htm>

## Setup scripts structure

Top level folders:

- `files` - those are various files used by setup scripts during setup process.
- `languages` - scripts for installing individual languages. Note that not all language are installed with these scripts. Some languages come from dnf, pip, npm or other sources.
- `private` - these are files that contain information that may vary from installation to installation. See below.
- `stage` - Those are the main setup scripts
- `misc` - miscellaneous scripts *not* used by the setup scripts. Mainly convenience for maintainers

- `bootstrap` - this is the script that needs to be run to start the setup process
- `run-scripts` - a utility script that executes all scripts in a specified directory, such as `stage`, `languages`.
- `setup-main` - script that is called at the end of `bootstrap`. It's a separate script for historical reasons and will be merged into bootstrap script in future.

## Configuring Linode

For both main an arena server you will need to switch to distribution supplied kernel (linode by default provides custom one) and enable SeLinux on your Fedora 25 image.
Once you created a Linode (1024MB of RAM, 20GB hard drive, or better), and deployed the Fedora 25 image you need to boot your linode, and wait until it boots up. Ssh in to make sure it did, or watch the boot progress in Lish.

Go to Linode Manager and edit your linode configuration profile. In the "Boot Settings" section select "GRUB 2" from the "Kernel" drop down, and click "Save changes". On
the "Remote Access" tab Click "Add a Private IP". Reboot your linode. Wait for it to back come up. It will take slightly longer than usual, as SeLinux relabeling and another reboot will happen automatically. Now you can follow the rest of the installation steps above.

## Setting up main server

Once you cloned the repository into `/opt/tiosetup`  you will need to provide content of `private/config` file in the `private` directory. Here is `config.default`, that you can use as a template:

```Bash
# Provide commit number or branch for `git checkout` command performed after cloning tryitonline repository
# This allows installing a historic version or a feature branch instead of the master latest
TryItOnlineCommit="master"

# To be able to use Dyalog APL, you need to download 64-bit Dyalog APL Classic and Unicode
# to /opt This requires a valid Dyalog license. If you have one, you can save your MyDyalog
# username in this file and run the misc/dldyalogapl script manually, before the remaining
# setup scripts. If you do not have a license, you can run the setup script without the
# Dyalog APL archive, but Dyalog APL won't be installed.
# DyalogUser=

# The following five settings are for the domain names used in the setup
# Please read more about them in accompanying documentation.

# Domain where https://tryitonline.net will be hosted
TRYITONLINENET=www2.tryitonline.nz

# Domain where https://tio.run will be hosted
TIORUN=tio2.tryitonline.nz

# Domain for <language>.tryitonline.net -> tio.run/nexus/<language> redirects.
# This setting is optional and may be left empty or removed entirely.
# LANGDOMAIN=

# This is your email used for LetsEncrypt certificate revocations
EMAIL=letsencrypt@tryitonline.nz

# Put your backed up let's encrypt certificates from previous TIO installation in
# private/letsencrypt.tar.gz and leave this setting alone
# Alternatively if you do not have the certs yet and would like to generate them
# change the line to
# UseSavedCerts="n"
# Note that LetsEncrypt limits you to 5 cert requests per week so you do not want
# to keep this setting saying n if your first attempt to install the mirror failed
# see accompanying documentation to find out how to back up generated letsencrypt
# certificates in this case
UseSavedCerts="y"

# You can provide your own SSH key pairs or let them be generated automatically.
# For the root (resp. apache) user, if either an RSA key pair (id_rsa and id_rsa.pub)
# or an ED25519 key pair (id_ed25519 and id_ed25519.pub) is found in private/root
# (resp. private/apache), they will be copied in the user's ssh folder. Otherwise, an
# ED25519 key pair will be generated automatically.
# Likewise, if either an RSA key pair (ssh_host_rsa and ssh_host_rsa.pub) or an
# ED25519 key pair (ssh_host_ed25519 and ssh_host_ed25519.pub) is found in private,
# they will replace the pre-generated host keys in /etc/ssh.

# This option should be left turned off ("n"). If it's turned on ("y") certificates
# and selinux configuration will be skipped during setup. This is useful for running
# an offline copy of TryItOnline in a docker image (which does no support selinux)
# Exposing an installation in offline mode to internet is a big security risk and should 
# never be done.
OfflineMode="n"
```

Below is a general scenario of starting main server setup:

```Bash
cd /opt
dnf install git nano screen openssl wget -y
git clone https://github.com/TryItOnline/tiosetup.git
cd tiosetup

mkdir private/apache
mkdir private/root

# put your previously saved letsencrypt.tar.gz to private/letsencrypt.tar.gz
# or make edit to edit private/config to read `UseSavedCerts="n"`

# create private/config

# add more scripts to /private to execute at the end of setup process if needed (don't forget to `chmod +x` them if you do)

# put your public key you generated earlier for connection to arena in private/apache/id_rsa.pub or private/apache/id_ed25519.pub
# put your private key you generated earlier for connection to arena in private/apache/id_rsa or private/apache/id_ed25519
# put your public key you generated earlier for rsync to arena in private/apache/id_rsa.pub or private/root/id_ed25519.pub
# put your private key you generated earlier for rsync to arena in private/apache/id_rsa or private/root/id_ed25519
# put your public key you generated earlier for host authentication in private/ssh_host_ed25519_key.pub
# put your private key you generated earlier for host authentication in private/ssh_host_ed25519_key
# if you have saved certs put them in private/letsencrypt.tar.gz

./bootstrap
```

Logs can be found in `/var/log/tioupd` directory. You can `tail` individual items from here for long-running sub-scripts. `misc/tiolog` script provide a simple bash script for tailing these.

Setup adds 127.0.0.1 as a linked arena, so once setup is finished, you should be able to run TryItOnline now.

## Setting up arena

You need to provision a fresh linode and configure kernel as described above. You need to add root/apache public keys so that the main server can initialise the arena.
The simplest way to do this is just to copy `/root/.ssh/authorized_keys` over from main. In addition you need to make sure that the host key for ssh (in `/etc/ssh/ssh_host_ed25519_key`) matches the corresponding public key from main.

Once all the above ready run on main:

```bash
tioinit arena_ip arena_name
```

Where `arena_name` is an arbitrary name that will be written to `/etc/hosts` file to correspond to the private ip of the arena linode given as `arena_ip`. E.g.:

```bash
tioinit 192.168.132.37 arena
```

When syncing has finished (it will take a few minutes), you might want to remove the main out of rotation:

```bash
arenactl disable runner@127.0.0.1
```

And then put newly create arena in rotation"

```bash
arenactl add runner@arena
arenactl enable runner@arena
```

## Dry run

On the main server, you can execute the following to test all languages on the local (main) arena.

```bash
tiodryrun
```

To specify an arena to run the tests against do:

```bash
tiodryrun -a arena
```

If setup was successful, you should see the following output after a few minutes.

```text
Testing X languages on arena arena...
Arena arena responded after Y seconds.
Result: X succeeded, 0 failed, 0 not tested
```

Failed tests will print the language name, the expected output, the actual output, and all pertinent information from STDERR. To see STDERR output (timings, potential warnings, etc.) of successful tests, add the `-v` or `--verbose` flag.

It is also possible to run the test for select languages. To do so, specify -l flag. Eg:

```bash
tiodryrun -a arena -l jelly
```

Another test utility is available at <https://github.com/TryItOnline/TioTests>. That one allows running the tests remotely from any client, not just from the main server.

## Docker

It is possible run similar  setup in a docker container, also a pre-build docker container available. This is intended for offline use only, as SeLinux specific commands cannot be run inside a docker container so it has SeLinux disabled. That makes it unsafe to expose on a non trusted networks, such as internet.

See <https://github.com/TryItOnline/tiodocker> for the details.
