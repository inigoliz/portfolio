---
date: '2026-05-05T00:08:42+02:00'
draft: true
title: 'Different Github accounts in the same pc'
---

This is going to be a short post. I found a good solution to a problem that I've had for a long time, and I wanted to share it.

## The solution that I don't like:

If you've had this issue and googled around, surely you have come accross the following proposal to have separate accounts on the same pc:

```bash
# ~/.ssh/config

# Personal GitHub
Host personal.github.com
    HostName ssh.github.com
    User git
    Port 443
    IdentityFile ~/.ssh/id_ed25519_personal
    IdentitiesOnly yes

# Work GitHub
Host work.github.com
    HostName ssh.github.com
    User git
    Port 443
    IdentityFile ~/.ssh/id_ed25519_work
    IdentitiesOnly yes
```

I don't quite like this solution. Cloning requires that you overwrite the remote address:

```bash
# For personal projects
git clone git@personal.github.com:yourusername/repo.git

# For work projects
git clone git@work.github.com:yourcompany/repo.git
```

I personally always forget what's the name of the remote I set, so when I want to clone a new repo first I have to open `config`, copy-paste the remote name, paste it in the clone url in the right place. Way too complicated.

## The solution I like

I found that I can simply define an alias that overwrites the ssh key envvar on the fly:

```bash
alias ghp='SSH_AUTH_SOCK= GIT_SSH_COMMAND="ssh -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519_personal"'
```

Then, when I want to use my personal github ssh key, I prepend the usual git commands that need access to the remote by `ghp` (which stands for *github personal*):

```
ghp git clone git@github.com:inigoliz/portfolio.git

git add -a
git commit -m "..."
ghp git push
```

Ah! And don't forget to set your username in the `.gitconfig` of your repo:

```bash
# repo/.gitconfig
git config --global user.name "Miguel de Cervantes"
git config --global user.email miguel@cervantes.es
```

That's it!

