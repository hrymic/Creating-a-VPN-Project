# Creating your own VPN for Windows

This project was inspired by Wolfgang's YouTube [video](https://www.youtube.com/watch?v=gxpX_mubz2A&t=1231s), for creating your own VPN. The video is very informational on the purpose for creating your own VPN. For this project we will be following the video such as using Linode for the VPS and OpenVPN as the VPN. Let's begin.

## 1. Create a Linode

Create a [Linode](https://cloud.linode.com/). The plan I used was the Nanode 1 GB. At the time of the project if you created an account using your gmail account you received a $100 credit.

### Linode configuration
Image: Ubuntu (21.10)
Region: Fremont, CA # The location of the Linode will affect Latency
Linode Plan: Nanode 1GB # Located under Shared CPU
Linode Label: purplehazevpn
Root Password: # Make one
SSH Keys: # Skip for now
Attach a VLAN: # Skip
Add-ons: Check the box for Private IP

Click "Create Linode"
Take note of your VPS IP address


## 2. Generating the SSH key

Open PowerShell and install the SSH command. Then generate the SSH key:
Add-WindowsCapability -Online -Name OpenSSH.Client*
ssh-keygen -t rsa -b 4096

You will be prompted to enter a passphrase.

## 3. Server setup

### Logon to the server
In PowerShell use ssh root@(ip_address)

You should now be connected to your server by SSH. The server is headless and everything on the server will be performed in the Terminal.

### Update the OS
Check that the server is has the current version OS and software
apt-get update && apt-get upgrade


### Create a user
Since we will be disabling the root account later, we need to create a user that can perform administrative tasks.
useradd -G sudo -m purplehaze -s /bin/bash

Then we need to make a password for the new user
passwd purplehaze

### Copy the SSH key from the local machine to the server
Open a second PowerShell and enter:
type $env:USERPROFILE\.ssh\id_rsa.pub | ssh wolfgang@ip_address "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"


We should now have the SSH key on the server.
Go back to the SSH session and keep this second PowerShell open.

### Restricting SSH to key authentication
In the SSH session we need to edit the SSH config file. Edit the file using nano:
nano /etc/ssh/sshd_config

Port 69 # This is what Wolfgang used
PasswordAuthentication no # To login only using the public SSH key
PermitRootLogin no # Disables the root user

Save the file.

Restart the SSH service:
systemctl restart sshd

Exit the SSH session then try to reconnect with the user we created.
ssh -i ~/.ssh/id_rsa purplehaze@ip_address -p 69

If you're prompted to enter the "passphrase for key" followed by the path to the key then we confirmed SSH is asking for the public key for authentication.

### Creating a server alias for easy login
The above SSH command to logon to the server is long. We can create an alias on the local machine to make connecting to the server easier. This is done by creating a file we're calling "config" in the .ssh folder in our home directory.

In PowerShell:
Set-Location ~/.ssh; New-Item -Name "config" -ItemType "file"; notepad.exe config

This should create a new file called "config" in the .ssh folder then open the config file in Notepad so we can edit the file.

In Notepad we enter the following information:
Host = name of vpn
	User = user account we created on the server
	Port = ssh port on the server
	IdentityFile = location of ssh key file (this was specified by the option -i in the SSH command)
	HostName = ip_address of the server

It should look like this:
Host purplehazevpn
        User purplehaze
        Port 69
        IdentityFile ~/.ssh/id_rsa
        HostName ip_address

Save the file and close.

Test the alias:
ssh purplehazevpn

Now that the server is setup, we need to setup the VPN.

## 4. OpenVPN setup
OpenVPN is a free open source connection protocol that is a trusted technology used to send encrypted data between two points over the internet creating a private network.

Since setting up the OpenVPN server takes time, Wolfgang suggested using the [OpenVPN road warrior](https://github.com/Nyr/openvpn-install) script created by github user Nyr.

### Use wget
We need to check that wget is installed on the server:
sudo apt install wget

Next, get the script and run it:
wget https://git.io/vpn -O openvpn-install.sh
sudo bash openvpn-install.sh

The script will ask a few questions and should look similar to this:
Welcome to this OpenVPN road warrior installer!

Which IPv4 address should be used?
IPv4 address 1: 1 # server ip_address

Which protocol should OpenVPN use?
   1) UDP (recommended)
   2) TCP
Protocol 1: # Hit Enter key to skip

What port should OpenVPN listen to?
Port 1194: 443 # For HTTPS

Select a DNS server for the clients:
   1) Current system resolvers
   2) Google
   3) 1.1.1.1
   4) OpenDNS
   5) Quad9
   6) AdGuard
DNS server 1: 3 # Wolfgang uses 1.1.1.1

Enter a name for the first client:
Name client: purplehaze # Can be anything you want

After the script is completed, the script places the configuration file (.ovpn) in the root directory. We need to move this file into our user's directory:
sudo mv /root/purplehaze.ovpn ~

Then we need to add our user as an owner of the purplehaze.ovpn file:
sudo chown purplehaze purplehaze.ovpn

Confirm purplehaze was given privileges to purplehaze.ovpn:
sudo ls -l  purplehaze.ovpn

Since the main purpose for creating our own VPN is to have a VPN service that does not keep logs, we need edit the config file to disable logs by changing the line verb 3 to verb 0:
sudo nano /etc/openvpn/server/server.conf

Save the changes, then restart the OpenVPN service to update the changes:
systemctl restart openvpn-server@server.service
You might see the message:
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to restart 'openvpn-server@server.service'.
Authenticating as: purplehaze
Password:
==== AUTHENTICATION COMPLETE ===

Last thing to do on the server is change the hostname:
sudo hostnamectl set-hostname purplehazevpn

If you exit the current session and reconnect, the host name should change from local host to the hostname you chose.

## 5. Download the config file
To use the VPN we need the config file on our local machine. Open PowerShell and enter:
sftp purplehazevpn # the servername
get purplehaze.ovpn # the client name you chose when setting up OpenVPN

To use the VPN we created, you will need [OpenVPN Connect](https://openvpn.net/client-connect-vpn-for-windows/) for Windows.

After installing OpenVPN Connect v3, OpenVPN Connect will as for the .ovpn file  you created. Now you have the ability to turn your VPN off and on.

## 6. Adding security
At the end of Wolfgang's video he talks about setting up unattended updates on the server and MFA when connecting to the server by SSH.

### Unattended updates
This will run apt update and apt upgrade regularly.

Install the unattended-upgrades package, use the default options when prompted for input:
sudo apt install unattended-upgrades apt-listchanges bsd-mailx

- We're installing the script for performing security updates automatically. apt-listchanges is a tool that will show what has been changed and bsd-mailx is a mail tool that will email the user about the updates.

Enable the security updates, select Yes at the prompt:
sudo dpkg-reconfigure -plow unattended-upgrades

- dpkg-reconfigure will reconfigure an installed package. -plow is "priority low".

Edit the config file in nano, and make the following changes:
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades

Unattended-Upgrade::Mail "mail@example.com";

Unattended-Upgrade::Remove-Unused-Kernel-Packages "true"; 
Unattended-Upgrade::Remove-Unused-Dependencies "true"; 
Unattended-Upgrade::Automatic-Reboot "true"; 
Unattended-Upgrade::Automatic-Reboot-Time "05:00"; # the time we want our system to reboot

- The line for the mail is where you want to receive the updates

Now we have setup the server to perform automatic updates. Next is setting up MFA.

### Multi factor authentication
Earlier we set the server so that only our local machine can connect by SSH using the public key for authentication, which is "something you know." Next we are setting up another method for authentication, "something you have."

For this project I installed Authy on my phone to generate a OTP for authentication.

To setup the account I followed Wolfgang's method for installing Google-Authenticator on the server:
sudo apt install libpam-google-authenticator

- The Google Authenticator PAM module is basically an authentication module for adding anpther authentication method separate from the program's authentication, in this case the SSH public key.

Run the script by typing "google-authenticator" in the Terminal.
- A QR code will appear. Use your choice of MFA app, I'm using Authy, to scan the code and add the account.

Now we need to change the authentication settings for the ssh file:
sudo nano /etc/pam.d/sshd

Comment out @incude common-auth, the line should look like this:
[# @include common-auth]

Then we need to add the following line at the end of the file:
auth required pam_google_authenticator.so

Save the file, then edit the SSH config file and make the following changes:
sudo nano /etc/ssh/sshd_config

ChallengeResponseAuthentication yes 
UsePAM yes

Add a new line under UsePAM:
AuthenticationMethods publickey,password publickey,keyboard-interactive

Save the file, then restart the SSH service:
sudo systemctl restart sshd

Before closing our original session, open another PowerShell and SSH into your server. Confirm that the MFA works.

