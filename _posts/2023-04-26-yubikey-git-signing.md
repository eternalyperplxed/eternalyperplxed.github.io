---
layout: post
title: Setting Up a Yubikey for Git Signing
date: 2023-04-26 16:45
---
One of the many hot topics of late is supply chain security. In the interest of having security be a part of the equation as early as possible, or "shifting security to the left", ensuring the integrity and identity of who is commiting changes to your internal code bases is critical. Whether you have internally developed applications and microservices or just infrastructure as code repositories, ensuring only authorized users are making changes is a key component to securing your internal supply chain. One great way of doing this is by forcing contributors to cryptographically sign their commits via GPG and hardware tokens.

Understandably, GPG is considered a bit unweildy, but hopefully this guide will make the process a bit easier to understand and implement.

Prerequisites:
- A Git based code repository (Github, Gitlab, etc)
- A new hardware token that supports GPG keys such as a Yubikey 5c (recommended)

1. Install GPG for your operating system
	1. MacOS - `brew install gpg`
	2. Linux - `apt install gnupg` or `yum install gnupg` 
	3. Windows - Download and install [Gpg4win](https://www.gpg4win.de/index.html)
	4. Verify installation by running `gpg --version` from a terminal
2. Install utilities for your hardware token. Since I'm using a Yubikey, I'll be using the `ykman` utilities
	1. MacOs - `brew install ykman`
	2. Linux - `apt isntall yubikey-manager` or `yum install yubikey-manager`
	3. Windows - Install the [Yubikey Manager](https://www.yubico.com/support/download/yubikey-manager/)
	4. Verify installation with `ykman --version` (NOTE: Windows users will need to navigate to the directory to run `ykman`)
3. Generate PIN's for Yubikey.
   Yubikey's have 2 PIN's, a user PIN, and an admin PIN which is also called a PIN Ublock Code or PUK. By default, the user PIN is 123456 and the admin PIN/PUK is 123456578.
	1. With the Yubikey inserted into the system, run `gpg --edit-card` from a terminal. You should see something similar to this:
	   ![gpg edit-card status](/assets/images/gpg_card_status.png)
	2. Your prompt should now read `gpg/card>` 
	3. Enter admin mode with the `admin` command
     ![gpg admin allowed](/assets/images/gpg_admin_allowed.png)
	4. Enter `passwd` to being setting custom PINs
     ![gpg passwd](/assets/images/gpg_pin_change.png)
	5. First change the admin PIN/PUK by select option `3`
	6. A window will appear prompting you to enter the existing admin PIN and then asking you to enter the new PIN twice.
	7. Select option `1` to change the user PIN. 
	8. A window will appear promopting you to enter the existing user PIN and then asking you to enter the new PIN twice.
	9. Quit the PIN change menu by entering `q`
4. Configure Key Size on Yubikey
   I'll be generating keys based on the RSA standard since that still has the broadest compatibility.
	1. First, I want to enforce a 4096 bit key for all types of encryption keys. Enter the `key-attr` from the `gpg/card>` prompt.
     ![gpg key size](/assets/images/gpg_key_size.png)
	2. Select option `1` for RSA
	3. Enter `4096` at the prompt
	4. You will be prompted for you admin PIN/PUK. Repeat these steps for each type of key (encryption and authentication)
5. Generate GPG Key on Yubikey
	1. From a terminal make sure you are at a `gpg/card>` prompt by entering `gpg --edit-card`
	2. Enter admin mode with the `admin` command
	3. Generate keys by entering the `generate` command. When asked about making an off-card backup I typically say no, but this is up to you and your threat model.
	4. You will be prompted for your user PIN, followed by a key expiration period. Choose an expiration time that you're comfortable with, but remember, the private key is stored on the hardware token making key exposure exceptionally difficult. 
	5. Provide your name and email so the key is associated to you when used.
	6. Confirm your entries, and then watch the light on the Yubikey blink like Las Vegas for a while as it generates the keys.
	7. Once complete, quit out of the `gpg/card>` prompt with the `quit` command.
6. Require touch on Yubikey
   Requiring a physical touch on the hardware token ensure that if a malicious actor gains remote control of your system while the token is inserted, they still will not be able to utilize it.
	1. Enter the following commands from a terminal to require a touch for each type of key usage. (NOTE: Remove your token and reinsert it if you just generated fresh keys)
		1. `ykman openpgp keys set-touch sig fixed`
		2. `ykman openpgp keys set-touch aut fixed`
		3. `ykman openpgp keys set-touch enc fixed`
	2. You will be prompted for your admin PIN/PUK after each of these commands along with a confirmation.
7. Export public GPG key and add to user profile
	1. Export your GPG key with the following command `gpg --export --armor <email address>` using the email address your setup the key with
     ![gpg export](/assets/images/gpg_export.png)
	2. Copy output of the command including the header and footer
	3. I'm using Github for my repositories, so I'll go to my account settings, and then SSH and GPG keys.
	4. Paste the GPG key block including the header and footer into the key and give it a unique name.
	5. If you want, enable Vigilant mode, but be aware that this will flag all over your previous commits as unverified.
8. Enforce signed commits on repository
   In Github, requiring signed commits is a per repository setting (at least with free accounts)
	1. Navigate to your repository and go to settings
	2. Under Code and Automation, select Branches
	3. Click the button for "Add branch protection rule"
	4. Enter a pattern for which branch or branches you want this to apply to. For all, enter `*`, or the names of branches such as `main`, `dev`, etc.
	5. Select the checkbox for "Require signed commits"
     ![Github branch rule](/assets/images/github_branch_rule.png)
	6. Click the "Create" button at the bottom of the page.

Ta Da! If everything has gone according to plan, you now have a shiny new GPG key stored on a hardware token that requires a physical interaction to use in order to sign commits to a software code repository. You can now ensure a high level of security at one of the earliest stages of your internal software supply chain.

If you have any suggestions to improve this or have any questions, feel free to reach out!