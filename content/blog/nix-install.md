+++
title = "Installing nix the hard way"

description = "My story of installing `nix` package manager on machine behind MITM-proxy"

date = 2024-04-17
template = "page_with_toc.html"
[taxonomies]
tags = ["system-administration", "nix"]
[extra]
show_only_description = true
+++
# Why bother with all that staff?
People talk a lot about Nix and how it helps to build reproducible environments.
Nix community is so focused on reproducibility, that using `nix-env` is considered a [bad](https://stop-using-nix-env.privatevoid.net/) practice.

However, reproducibility is not the only positive feature of `nix`.
Another important feature is [multi-user mode](https://nixos.org/manual/nix/stable/installation/multi-user.html).
This may not sound very useful for individuals, but I think, that for big companies it is a dealbreaker.

Just think about it!
For IT companies it is a common situation when several developers work on a single remote machine.
For stability, and other Important Reasons™ such machines rarely have updated software.

Either for compatibility with some production code, or because of security policies, this servers are not just outdated, but the updates are intentionally disabled.

They may even be disconnected from the internet, allowing access only to the company's internal network.

Even if updates were possible, they would be very risky: updating one package may require updating its dependencies and therefore other packages which depend on them.
> Partial updates are not supported.
>
> © ArchWiki

Of course, package manager will resolve such conflicts and just updates everything.

...probably breaking someone else's environment.
So, updates are often restricted.

# Installation
A few days ago I've explained this to the management and today I was finally given access to proxy, which will allow me to download things from the internet.

First of all, I had to download the installer, as stated in the official [guides](https://nixos.org/manual/nix/stable/installation/#multi-user):
```bash
bash <(curl -L https://nixos.org/nix/install) --daemon
```
Of course, it didn't work without configuring the proxy, so actually I had to do:
```bash
export HTTP_PROXY=http://<username>:<password>@proxy.addr:port
export HTTPS_PROXY=http://<username>:<password>@proxy.addr:port
bash <(curl -L https://nixos.org/nix/install) --daemon
```
And...
It still hasn't worked!
`curl` failed with a cryptic error saying that `<my username>:` (yes, with colon) is not a valid port number.
WTF!?

My password has special characters.
Maybe they were not property escaped?

I've tried, but without success.
After a bit of digging I've found that (it looks very obvious now) `HTTP_PROXY` variable is parsed as URL and therefore all special characters have to be [percent-encoded](https://en.wikipedia.org/wiki/Percent-encoding).

OK... I've redefined proxy variables and installation script was finally downloaded.
The script started working and have downloaded the tarball.
I've entered `sudo` password, it successfully created users and copied files from the package to `/nix` directory.
Everything was fine until it runned `nix-channel --update nixpkgs`.
Nothing happened for a while, when a connection timeout warning appeared.

That looked suspiciously like a proxy failure.
I've added `--no-channel-add` flag to the install script, so that it will skip this step, uninstalled everything [^1] as explained in the [docs](https://nixos.org/manual/nix/stable/installation/uninstall#linux) and started from the begining.

The installation seemed to succeed this time, so I've started a new shell and did:
```bash
nix-channel --add https://nixos.org/channels/nixpkgs-unstable nixpkgs
```
But it failed because of insufficient permissions.
Of course!
I forgot to add my user to the right group.

According to the [manual](https://nixos.org/manual/nix/stable/installation/multi-user#restricting-access), only users who have write permission to `/nix/var/nix/daemon-socket` are allowed to work in multi-user mode.
I've already had `users` group, so I did just:
```bash
sudo chgrp users /nix/var/nix/daemon-socket
sudo chmod ug=rwx,o= /nix/var/nix/daemon-socket
```

Now my user had enough permissions and I was able to run `nix-channel`:

```bash
nix-channel --add https://nixos.org/channels/nixpkgs-unstable nixpkgs
nix-channel --update nixpkgs
```

As previously happend in the istall script, `--update ` command hanged.
I've started searching, how people deal with proxies in Nix.
Among others, I've found [this](https://github.com/NixOS/nix/pull/2946/files) PR, which explicitly add support for proxies, but, as far as I can tell, only for `nix-daemon`.

I've added `HTTP_PROXY` and `HTTPS_PROXY` vars and tried again.
This time I've got a different error: `nix-channel` was complaining about bad SSL cert.

# Here comes the funny part...
Well, our machines are not only behind proxy, they are behind MITM-proxy.

`curl` and other installed software works fine, because they use system-wide CA bundles, but Nix comes with its own certificates.

After some googling I've found `NIX_SSL_CERT_FILE` variable and tried to set it.
It worked!
The channel was successully updated and with a feel of acomplishment I've issued `nix-shell -p vim` to try it out.

The command was silent for almost a minute, then I saw a familiar timeout error.
But I've configured the proxy vars!
They were definetly passed to the systemd override, the installer has clearly shown it (I still had the installer output in the unclosed term).

In a fit of suspiciousness I've opened override file.
At first glance, everything seemed fine, proxy variables were set, but the password field in the URLs looked a bit strange.
I've taken a closer look and found that nix installer has reencoded my percent-encoded characters and messed them up!
I've also remembered the certificate thing, so I've also add `NIX_SSL_CERT_FILE` variable to the override.

```bash
sudo systemctl daemon-reload
sudo systemctl restart nix-daemon.service
```

And finaly `nix-env -i vim`[^2] worked!
Initial enviroment installation was a bit slow, but it worked!
It took me a whole working day to do this _simple_ install...

# TL;DR:
If you need to install `nix`  in multi-user mode behind MITM-proxy, do the following:
```bash
export HTTP_PROXY=http://<username>:<password>@proxy.addr:port
export HTTPS_PROXY=http://<username>:<password>@proxy.addr:port
bash <(curl -L https://nixos.org/nix/install) --daemon
```
If your username or pasword has special characters, [percent-encode](https://en.wikipedia.org/wiki/Percent-encoding) them.

Open `/etc/systemd/system/nix-daemon.service.d/override.conf` and check that variables `HTTPS_PROXY`, `HTTPS_PROXY` are not messed up.
Add variable `NIX_SSL_CERT_FILE` to that file and run:
```bash
sudo systemctl daemon-reload
sudo systemctl restart nix-daemon.service
```

Then add `nix-socket` to the right group:
```bash
sudo chgrp nix-users /nix/var/nix/daemon-socket
sudo chmod ug=rwx,o= /nix/var/nix/daemon-socket
```

In a new shell set `HTTPS_PROXY`, `HTTPS_PROXY` and`NIX_SSL_CERT_FILE` variables[^3] and run:
```bash
nix-channel --add https://nixos.org/channels/nixpkgs-unstable nixpkgs
nix-channel --update nixpkgs
```

That's it!
`nix` is installed on your system.


[^1]: without that the script will complain about existing installation.
[^2]: again, I know that using `nix-env` is discouraged, but for my particular setup it is more suitable.
[^3]: you need to set this variables only for operations with `nix-channel`. `nix-env` and `nix-shell` works without them.

---
