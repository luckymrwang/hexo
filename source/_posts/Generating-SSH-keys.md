title: "Generating SSH keys"
date: 2015-05-16 12:03:03
categories: example #分类
tags: [SSH]
description: 
---

##Step 1: Check for SSH keys
``` bash
$ ls -al ~/.ssh
# Lists the files in your .ssh directory, if they exist
```
Check the directory listing to see if you already have a public SSH key. By default, the filenames of the public keys are one of the following:
`>` id_dsa.pub
`>` id_ecdsa.pub
`>` id_ed25519.pub
`>` id_rsa.pub
<!-- more -->
##Step 2: Generate a new SSH key
- With Git Bash still open, copy and paste the text below. Make sure you substitute in your GitHub email address.
``` bash
$ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
# Creates a new ssh key, using the provided email as a label
Generating public/private rsa key pair.
```
- We strongly suggest keeping the default settings as they are, so when you're prompted to "Enter a file in which to save the key", just press Enter to continue.
``` bash
Enter file in which to save the key (/Users/you/.ssh/id_rsa): [Press enter]
```
- You'll be asked to enter a passphrase.
``` bash
Enter passphrase (empty for no passphrase): [Type a passphrase]
Enter same passphrase again: [Type passphrase again]
```
- After you enter a passphrase, you'll be given the fingerprint, or id, of your SSH key. It will look something like this:
``` bash
Your identification has been saved in /Users/you/.ssh/id_rsa.
Your public key has been saved in /Users/you/.ssh/id_rsa.pub.
The key fingerprint is:
01:0f:f4:3b:ca:85:d6:17:a1:7d:f0:68:9d:f0:a2:db your_email@example.com
```

##Step 3: Add your SSH key to your account
Copy the SSH key to your clipboard. If your key is named id_dsa.pub, id_ecdsa.pub or id_ed25519.pub, then change the filename below from id_rsa.pub to the one that matches your key:
``` bash
$ clip < ~/.ssh/id_rsa.pub
# Copies the contents of the id_rsa.pub file to your clipboard
```

Add the copied key to GitHub:
- In the top right corner of any page, click `setting`.
![setting](/images/userbar-account-settings.png)

- In the user settings sidebar, click SSH keys.
![SSH keys](/images/settings-sidebar-ssh-keys.png)

- Click Add SSH key.
![Add SSH key](/images/ssh-add-ssh-key.png)

- In the Title field, add a descriptive label for the new key. For example, if you're using a personal Mac, you might call this key "Personal MacBook Air".
- Paste your key into the "Key" field.
![Key](/images/ssh-key-paste.png)

- Click Add key.
![Add key](/images/ssh-add-key.png)

- Confirm the action by entering your GitHub password.

##Step 4: Test the connection
To make sure everything is working, you'll now try to SSH into . When you do this, you will be asked to authenticate this action using your password, which is the SSH key passphrase you created earlier.

- Open Git Bash and enter:
``` bash
$ ssh -T git@github.com
# Attempts to ssh to GitHub
```

- You may see this warning:
``` bash
The authenticity of host 'github.com (207.97.227.239)' can't be established.
RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
Are you sure you want to continue connecting (yes/no)?
```
- Verify the fingerprint in the message you see matches the following message, then type yes:
``` bash
Hi username! You've successfully authenticated, but GitHub does not
provide shell access.
```

- If the username in the message is yours, you've successfully set up your SSH key!