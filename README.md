# CloudPanel certificate scripts

These scripts are intended to help generate and install *root domain and wildcard* SSL certificates, using incron to trigger [acme.sh](https://github.com/acmesh-official/acme.sh/tree/master) when a user generates a site in [CloudPanel](http://cloudpanel.io).

It's intended for a configuration where the acme.sh user is not the root user, is set up to use Cloudflare DNS and uses LetsEncrypt to generate certificates.

## Prerequisites
Assuming you have already installed CloudPanel, you will need to install acme.sh following the instructions on the repo, and install incron via your package manager (i.e. `sudo apt install incron`).

You will also need to have generated a certificate already via acme.sh, using your Cloudflare API key. All domains that you want to create in CloudPanel will need to be added to this API key via `Manage Account > Account API tokens`, as well as have DNS set up on Cloudflare, otherwise the certificate generation will fail.

## Usage

Grab the scripts:

```
git clone https://github.com/papalozarou/cloudpanel-certificate-scripts.git
```

You will need to edit `clp-common` to add the user name for the acme.sh user.

Create a folder within your acme.sh user's home directory, to store stub files:

```
mkdir ~/.clp-sites-enabled
```

If you change the name of this directory, you will need to update `clp-common` to reflect this.

Copy the scripts to `/usr/local/bin`:

```
sudo cp ~/cloudpanel-cert-scripts/* /usr/local/bin
```

You may need to check that files are executable, via something like `ls -lna /usr/local/bin | grep clp*`, though `clp-common` does not need to be as it only contains shared variables and is imported into the other three files.

Add the following to the root users incrontab, via `sudo incrontab -e`, replacing the `$ACME_USER` placeholder with the name of the acme.sh user:

```
/etc/nginx/sites-enabled/ IN_CREATE /usr/local/bin/clp-generate-domain-stub $#
/home/$ACME_USER/.acme.sh/ IN_CREATE  /usr/local/bin/clp-install-certificate $#
```

Then add the following to the acme.sh users incrontab, via `incrontab -e`, again replacing the `$ACME_USER` placeholder with the name of the acme.sh user:

```
/home/$ACME_USER/.clp-sites-enabled/  IN_CREATE /usr/local/bin/clp-issue-certificate $#
```

If things don't work, check `journalctl` for clues.

## How it works

1. The root user watches, via incron, for domains being created by CloudPanel in `/etc/nginx/sites-enabled`. When a new site is created, `clp-generate-domain-stub` creates a domain stub file in `~/.clp-sites-enabled` in the acme.sh user's home directory.
2. The acme.sh user watches this folder, via incron, `~/.clp-sites-enabled` for a domain stub file. When a new stub file is created, `clp-issue-certificate` checks to see if a certificate has already been created for that domain. If it has not, a root domain and wildcard certificate will be generated via LetsEncrypt. If it has, the script exits.
3. The root user watches, via incron, the `.acme.sh` directory for newly created certificates folders. When a new certificate folder is created, `clp-install-certificate` waits 50s, to give LetsEncrypt time to generate the full set of certificate files, then installs the certificate to the correct domain in CloudPanel. Lastly it deletes the domain stub file within `~/.clp-sites-enabled`.

## Copying certs to local machines

If you want to use generated certificates for services within your local network you can download `clp-common` and `clp-certificate-transfer`. You will need to edit `clp-common` to add the user name for the remote acme.sh user as before.

`clp-certificate-transfer` is opinionated about how you are setup. And probably not in a good way. So you will likely need to edit the script to make it work in your local environment. At the very least you will need to edit the following:

```
LOCAL_USER=''
[…]
REMOTE_SSH_HOSTNAME=''
[…]
DOMAIN=''
[…]
REVERSE_PROXY=''
```

Once edited, copy both scripts to `/usr/local/bin` checking that `clp-certificate-transfer` is executable and add the following to the root users crontab, via `sudo crontab -e`:

```
5	6	*	*	*	/usr/local/bin/clp-certificate-transfer 2>&1 | /usr/bin/logger -t CLP-CERT-TRANSFER
```