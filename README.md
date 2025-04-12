# YubiKey macOS setup

This is an updated (and limited) version of https://github.com/liyanchang/yubikey-setup. All credit to the original instructions.

## Issues I have faced

- (non-issue) GitHub does not allow no-touch-required https://github.com/orgs/community/discussions/10593
- (see workaround below) Homebrew private repository taps break with Yubikey https://github.com/Homebrew/brew/issues/11425
- (open) Ansible has very limited support for `ed25519-sk` ssh keys: https://github.com/ansible/ansible/issues/82131
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

## YubiKey for GPG keysigning

1. Install GPG2 if you haven't already

   ```sh
   brew install gnupg gnupg2
   ```

2. Configure your GPG conf at `~/.gnupg/gpg.conf`

   Suggested hardened [configuration](https://github.com/ioerror/duraconf/blob/04f992ccd27fda38f742944066fbde39aa2ceb73/configs/gnupg/gpg.conf). Here's the minimum that makes sense:

   <!--I don't actually know enough about gpg as of writing to judge this-->

   ```
   use-agent
   personal-cipher-preferences AES256 AES192 AES CAST5
   personal-digest-preferences SHA512 SHA384 SHA256 SHA224
   cert-digest-algo SHA512
   default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP    Uncompressed
   ```

3. Generate Keys

   ```sh
   gpg --card-edit
   ```

   ```
   gpg/card> admin
   Admin commands are allowed

   gpg/card> generate
   Make off-card backup of encryption key? (Y/n) n

   [PIN Entry pops up, enter 123456, which is the default pin]

   Please specify how long the key should be valid.
            0 = key does not expire
         <n>  = key expires in n days
         <n>w = key expires in n weeks
         <n>m = key expires in n months
         <n>y = key expires in n years
   Key is valid for? (0)
   Key does not expire at all
   Is this correct? (y/N) Y

   GnuPG needs to construct a user ID to identify your key.

   Real name: <YOUR_NAME_HERE>
   Email address: <YOUR_EMAIL_HERE>
   Comment: <YUBIKEY IDENTIFIER, FAR/CLOSE>
   You selected this USER-ID:
       "YOUR_NAME_HERE <YOUR_EMAIL_HERE>"

   Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O

   Please enter the Admin PIN (default: 12345678)
   ```

   The YubiKey will flash as it's creating the key. Mine took about 18 seconds. When complete, it will say something like

   ```
   public and secret key created and signed.
   ```

   You should change your PIN and Admin PIN. You can do that here with `passwd` at the `gpg/card>` prompt:

   ```
   passwd

   1 - change PIN
   2 - unblock PIN
   3 - change Admin PIN
   4 - set the Reset Code
   Q - quit

   Your selection? 1
   [Enter 123456]
   [Enter your new PIN]
   [Enter your new PIN again]

   PIN changed.

   1 - change PIN
   2 - unblock PIN
   3 - change Admin PIN
   4 - set the Reset Code
   Q - quit

   Your selection? 3
   [Enter 12345678]
   [Enter your new Admin PIN]
   [Enter your new Admin PIN again]

   PIN changed.

   1 - change PIN
   2 - unblock PIN
   3 - change Admin PIN
   4 - set the Reset Code
   Q - quit

   Your selection? Q
   ```

4. (Optional) Other GPG Setup

   While you're here:

   ```bash
   gpg/card> name
   Cardholder's surname: [Your last name]
   Cardholder's given name: [Your first name]
   [Enter your admin PIN]

   gpg/card> sex
   Sex ((M)ale, (F)emale or space): [Your gender]

   gpg/card> lang
   Language preferences: [Your two letter language code, example: en)
   ```

   You can see the configuration by typing `list` on the `gpg/card>` prompt.

   https://support.yubico.com/hc/en-us/articles/360013790259-Using-Your-YubiKey-with-OpenPGP

<!--
## YubiKey for SSH logins

You can generate an SSH key from your PGP key and use it for SSH logins.

1. Identify your authentication key.

   ```bash
   > gpg2 --card-status | grep Authentication
   Authentication key: AAAA BBBB CCCC DDDD EEEE  FFFF GGGG HHHH IIII JJJJ
   ```

2. Generate the SSH key

   Take the last 16 digits and pass them to `gpg --export-ssh-key`.

   ```bash
   > gpg --export-ssh-key GGGGHHHHIIIIJJJJ
   ssh-rsa AAAAG4AFq6wm1eCcRclsVOYcJf8y
   ...
   ...
   G46wm1eCcRclsVOYcJf8yPr1b+kzUpGQLw==
   ```

3. Copy the public key and add it `~/.ssh/authorized_keys` the machine you want to SSH into
4. Attempt to login to the machine via SSH
-->

<!--
## YubiKey for PIV

Yubico has a GUI tool called yubikey-piv-manager that can help set up your YubiKey for PIV. While I have a preference for command-line tools, the GUI sets everything up in one click and saves significant hassle.

1. Install YubiKey PIV Manager

   ```bash
   > brew cask install Caskroom/cask/yubikey-piv-manager
   ```

2. Navigate to `Setup for macOS` and click `yes`.

   Choose a 6-8 digit number. Don't use non-numeric characters. Yubikey will be fine, but macOS will not.

   The default settings are fine.

3. Remove and re-insert your YubiKey.

4. Pair with macOS

   When you insert your Yubikey, a prompt should appear asking if you would like to pair your smartcard. Click `Pair`. It will ask for your username and password as well as the pin you just created. It may also ask you for your keychain password - it's the same as your account password.

5. Login with your YubiKey and PIN

   The next time you login with your YubiKey inserted, macOS should prompt you for your PIN and not a password.
-->

<!-- Notes from when I was trying to set it up by hand

```
> yubico-piv-tool -s 9a -A ECCP256 -a generate
-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----
Successfully generated a new private key.

```
yubico-piv-tool -s 9a -S '/CN=nano4/OU=yubikey/O=ldchang.com/' -P 123456 -a verify -a request

Successfully verified PIN.
Please paste the public key...
-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----
-----BEGIN CERTIFICATE REQUEST-----
...
-----END CERTIFICATE REQUEST-----
Successfully generated a certificate request.
```

```
yubico-piv-tool -s 9a -S '/CN=nano4/OU=yubikey/O=ldchang.com/' -P 123456 -a verify -a selfsign

Successfully verified PIN.
Please paste the public key...
-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
Successfully generated a new self signed certificate.
```

```
> yubico-piv-tool -s 9a -a import-certificate
Please paste the certificate...
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
Successfully imported a new certificate.
```


## YubiKey for OSX login

Once you have PIV credentials on your YubiKey, macOS should prompt you if you want to use it for login.

TODO: Look into `yubiswitch` to see how it will lock the screen when the YubiKey is removed.
-->

<!--
## Set up your YubiKey at TOTP - a Google Authenticator replacement

You can have your YubiKey generate TOTP codes, just like Google Authenticator or Authy.

If you use it as a replacement for Google Authenticator, remember that you'll be unable to get the code if you don't have your YubiKey with you and a computer with `ykman` or `Yubico Authenticator` installed or an Android phone with `Yubico Authenticator` installed.

You can also use both a phone based app and a YubiKey, knowing that either device will generate the same codes and will be able to access your account.

1. Go to [Dropbox Security Settings](https://www.dropbox.com/account/#security)
2. Choose to enable two factor.
3. Select `Use a mobile app`
4. Click `enter your secret key manually` to display a 26 digit long base32 key. Note: The link text will differ by provider. The length of the base32 key may also differ.
5. (Optional) If you also want to use your phone, you can scan the barcode or type in the code to `Google Authenticator`.
6. Copy the key below - don't forget to remove the spaces

   ```bash
   % The `-t` will require a touch inorder for codes to be generated.
   % This prevent malware from generating codes without your knowledge.
   % YubiKey Neo's do not support this feature. Just remove the `-t` flag.
   > ykman oath add -t <SERVICE_NAME> <32 DIGIT BASE32 KEY NO SPACES>
   > ykman oath code <SERVICE_NAME>
   Touch your YubiKey...
   SERVICE_NAME 693720
   ```

7. Repeat for other providers.

   The steps will be similar - the difference will be how to get the manual key instead of the QR code. When the QR code is displayed, there will often be a link to get the code. Here are some examples:

   - Dropbox: `Enter your secret key manually`
   - Gmail: `Can't scan it?`
   - Github `Enter this text code`
-->