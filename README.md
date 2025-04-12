# YubiKey macOS setup

This is an updated (and limited) version of https://github.com/liyanchang/yubikey-setup. All credit to the original instructions.

## Issues I have faced

- (non-issue) GitHub does not allow no-touch-required https://github.com/orgs/community/discussions/10593
- (see workaround below) Homebrew private repository taps break with Yubikey https://github.com/Homebrew/brew/issues/11425
- (open) Ansible has very limited support for `ed25519-sk` ssh keys, even `no-touch-required`: https://github.com/ansible/ansible/issues/82131
  - Best workarounds that I currently have, both non-Yubikey:
    - I have tried using Strongbox on macOS as SSH agent. It does not solve the issue to a satisfactory degree.
    - I have _not_ tried 1Password SSH agent. It may solve the issue to a satisfactory degree, and it is cross-platform.

### Homebrew private tap workaround

In all likelyhood you only have one or few private taps, so when you try to run `brew update && brew upgrade`, you may see an error like this:

```
==> Updating Homebrew..
git@github.com: Permissions denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
Error: Fetching $path failed!
```

You can go edit the git configuration of the failed path repository, i.e.

```sh
cd $path
cd .git
vim config
```

and you can add the following configuration:

```
[credential "https://github.com"]
    helper= !gh auth git-credential
```

and also update the remote to use https protocol via

```sh
git remote set-url origin https://github.com/${org}/${repo}.git
```

and now your `brew update && brew upgrade` should work without a hitch.

Whether you want the `gh` cli present on your system (because it enables a bypass like this) is a separate question.

## Preface

_You bought a YubiKey - now what?_

The goal is to outline the steps to configure your YubiKey in a sane method and to use it to maximize your security.

This guide is for users who are comfortable with the command line and various technical jargon.

This is highly opinionated on how you should and should not use your YubiKey but is organized well enough that you should be able to modify if you have a need.

The instructions have been tested on macOS 15.3.2 (Sequioa) with a YubiKey 5C Nano. While there are sections that are OS independent, most of the tricky bits are macOS specific.

To perform these instructions, the YubiKey should be plugged into your computer's USB port.

## Install some software

```sh
# For macOS
brew install openssh libfido2 ykman
```

## Turn off OTP - AKA the random letters when you accidentally touch it

This will turn off One-Time-Password. Most users will not find OTP useful and will be confused by the random letters that will appear when they accidentally touch the YubiKey.

```sh
ykman config usb --disable OTP
```

## Use for Two Factor Authentication / U2F Setup

U2F is the recommended two factor method. It is phishing resistant unlike TOTP/Google Authenticator. It is much harder to compromise than SMS/Voice call methods.

The instructions below are specific to provider, but they are all similar enough.

### GitHub

1. Go to your GitHub [Password and authentication](https://github.com/settings/security) settings
2. Turn on `Two-factor Authentication` if it's not already enabled. You will need to set up either an SMS or TOTP (Google Authenticator) if it's not.
3. Under `Security keys, choose `Register new device`
4. Type in a name: `yubikey-5c-nano` or something else that will help you remember the key. I have two identical Yubikeys and found it helpful to differentiate by the usb-c port they were in: `far` and `close`.
5. Click `Add`
6. Follow the instructions on screen - you'll probably need to tap the YubiKey for it to register.

Yubico has more [detailed instructions](https://www.yubico.com/works-with-yubikey/catalog/github/#setup-instructions).

## How to generate SSH keys for Git/GitHub

There are non-discoverable credentials as described at [Passwordless SSH login with YubiKey and FIDO2](https://www.ajfriesen.com/yubikey-ssh-key/):

> This will create an SSH key on your local system in ~/.ssh but only works together with the YubiKey. So if I remove my YubiKey or lose the YubiKey altogether I can not use this SSH key anymore. That also means other people can not use this YubiKey on their machines. But that also means I have to copy this private key to all of the servers I want to use this on.

The behaviour described in the quote is desirable because I use Yubikey nanos which I intend to leave attached to my laptop at all times. The combination of a file and the key ensures that if someone yanks my physical key, it won't let them access my accounts.

A Yubikey-backed ssh key for GitHub authentication can be generated like this:

```sh
ssh-keygen -t ed25519-sk -C "$EMAIL" -f ~/.ssh/github-auth
```

Set it up with GitHub (I trust you know how) and now all your Git operations require taps to proceed. You may want to disable automatic git fetches in your editors to avoid error messages.

For signing your Git commits, you probably want to use a `no-touch-required` key, because otherwise it will encourage larger commits and make rebasing/merging a huge hassle:

```sh
ssh-keygen -t ed25519-sk -C "$EMAIL" -O no-touch-required -f ~/.ssh/github-sign
```
