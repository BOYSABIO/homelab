# PROTON VPN SETUP
1. Download ovpn file from https://account.protonvpn.com/account-password
2. Also get Auth username and password

```Bash
# Create vpn directory
mkdir -p ~/.vpn
chmod 700 ~/.vpn
  
# Create directory for .ovpn files
mkdir -p ~/.vpn/configs
mv ~/Downloads/us-free-4.proton.udp.ovpn ~/.vpn/configs/us-free-4.protonvpn.udp.ovpn
  
# Create file for auth user and password
nvim ~/.vpn/credentials.txt
AUTH_USER_NAME
AUTH_PASSWORD
  
# Add/create the bash file
sudo nvim /usr/local/bin/vpn
```
  

### HOW TO USE
To turn the vpn on:
```
vpn us
```
  
To check the status:
```
vpn status
```
  
To turn the vpn off:
```
vpn off
```

