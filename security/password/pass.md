# Introduction

Password is very important, so its safety and complexity too.
Specially in this time where the power of gpu is increasting and the hashrate too
Bruteforce and weaks passwords is working, check the number of tentatives using the
new generation of GPU and hashcat, [3080 benchmark](https://gist.github.com/Chick3nman/bb22b28ec4ddec0cb5f59df97c994db4)
However if you passwords is generated with a long length, alpha numeric, digits and all, you will be safe.

There is some passwords managers online, from keepass, to bitwarden.
If found this one, pass, its really simple use GPG, and filesystems, so the dude bind it with git.
I think its awsome. 


## Installation

Is available everywhere just use your favorite package manager and install it
The default directoy to saving password will be: `$HOME/.password-store` 


```
sudo apt install pass
sudu pacman install pass
brew install pass
nix-env -iA nixos.pass
```

## Basic command


#### Init

```
pass init <GPG KEY ID>
```

#### Generate a password

```
pass generate user/website 64
```

#### Getting the password
```
pass user/website
```


#### Copying to the clipboard
```
pass -c user/website
```

## Git

Pass will forward your git command.
Create a commit for each password insert or generated

#### Initialization

```
# Init the repository
pass git init
# Add origin remote 
pass git remote add origin git@example.com:user/passwords
```

#### Changing

You just need to push when ever you want, like after generated a new password
Simple use: 
```
pass git push
```

#### Signing the commit
Since you use gpg, you probably have a Signing key.
You can easly tell pass to sign the commit with:

```
pass git config pass.signcommits true
```


