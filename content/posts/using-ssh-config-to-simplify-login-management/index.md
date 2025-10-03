+++
date = '2025-10-02T09:27:24-05:00'
draft = false
title = 'Mastering SSH Config: Stop Juggling Keys, Ports, and Hostnames'
+++
If you're like me, you use SSH constantly. On any given day, I use SSH to
access GitHub, log in to various servers at work, and connect to my self-hosted
equipment at home. In my case, I juggle multiple SSH keys for different
identities, numerous domains and static IPs, and sometimes special port numbers
for SSH access. Managing all of this by hand would be a nightmare, but
thankfully, SSH config offers a powerful solution.

## The Problem

Before mastering my SSH config, my command history was littered with unwieldy
SSH patterns. I might need a work key to access a build server:

```bash
ssh -i ~/.ssh/id_ed25519.work workuser@dev.autopilot.NotTesla.com
```

Or access some server with an obscure SSH port:

```bash
ssh -i ~/.ssh/id_ed25519.work workuser@1.2.3.4 -p 12345
```

Or log in to my NAS device on my local network using my personal SSH key:

```bash
ssh -i ~/.ssh/id_ed25519.personal homeuser@192.168.1.101
```

Or use separate SSH keys for my professional vs. personal GitHub accounts:

```bash
GIT_SSH_COMMAND="ssh -i ~/.ssh/id_ed25519.work" git clone git@github.com:john_bigcorp/secret-sauce.git
GIT_SSH_COMMAND="ssh -i ~/.ssh/id_ed25519.personal" git clone git@github.com:phreaknik/blog.git
```

*Note: Git has some config options that help reduce this complexity, but in my
experience they don't cover all the bases.*

These are just a few examples, and already I'm juggling:
- Which SSH key for which user at which domain
- Which username on which domain
- Long, tedious key arguments on every command
- Port numbers for some domains

## Why So Many SSH Identities?

I'll admit... I'm probably using more SSH keys than the average person. But
**even if you only have one SSH key, you may still find yourself juggling
domains, ports, and usernames**. If you're a software engineer, let me make the
case for why you might want at least two SSH identities.

The short answer: **separation of concerns**.

When I leave a job and return my laptop, I want to destroy any SSH keys on
those devices. If I used my personal keys for work, I'd have to generate new
personal keys on all my personal devices and unregister the old keys from my
NAS and other personal services. This is tedious and error-prone. If you're a
consultant with multiple clients, this problem could multiply quickly.

By using separate keys for work vs. personal services, I can lose my work key
without worrying about access to my personal equipment, or vice versa.

*Note: Technically it would be better to have a unique key per device, but your
IT admin might refuse when you ask them to register 6+ keys.*

## ~/.ssh/config to the Rescue!

With a few entries in your SSH config file (usually at `~/.ssh/config`), you
can eliminate all this juggling.

### SSH Config for Git Access

Here's how I set up my GitHub identities:

```bash
# Personal GitHub Identity (github.com/phreaknik)
Host phreaknik.github.com
	HostName github.com
	User git
	IdentityFile ~/.ssh/id_ed25519.phreaknik

# Work GitHub Identity (github.com/john_bigcorp)
Host john_bigcorp.github.com
	HostName github.com
	User git
	IdentityFile ~/.ssh/id_ed25519.work
```

Now I can easily use either identity by prefixing the GitHub URL with my custom
host:

```bash
# Clone a personal project
git clone git@phreaknik.github.com:phreaknik/microcli.git

# Clone a company project that john_bigcorp has access to
git clone git@john_bigcorp.github.com:NotTesla/secret-sauce.git
```

### SSH Config for Generic Server Access

```bash
# Nickname 'box1' for server at 1.2.3.4
Host box1
	HostName 1.2.3.4
	User workuser
	IdentityFile ~/.ssh/id_ed25519.work

# Specify custom port for server at dev.fiatmine.com
Host fiatmine
	HostName dev.fiatmine.com
	User workuser
	Port 1234
	IdentityFile ~/.ssh/id_ed25519.work

# Personal user on my NAS box
Host nas
	HostName 192.168.1.101
	User phreaknik
	IdentityFile ~/.ssh/id_ed25519.phreaknik

# My wife's user on my NAS (just use my personal key for this)
Host nas
	HostName 192.168.1.101
	User mawife
	IdentityFile ~/.ssh/id_ed25519.phreaknik
```

Now I can SSH into any of these servers much more easily:

```bash
# Login to the work server at 1.2.3.4 with key ~/.ssh/id_ed25519.work
ssh workuser@box1

# Login to the fiatmine dev server
ssh workuser@fiatmine

# Login to my NAS with my personal user
ssh phreaknik@nas

# Do some admin work in my wife's account on our NAS
ssh mawife@nas
```

## Conclusion

This post is as much for my own reference as anything, but I hope it helps you
too. Mastering SSH config has saved me countless headaches over the years; no
more memorizing IP addresses, hunting for the right key, or copying commands
from my shell history. Just simple, memorable aliases that make SSH actually
pleasant to use.
