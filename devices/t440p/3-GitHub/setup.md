#GHOST GIT SETUP
Some prerequisites include that you cannot fully anonymize it. You would need to create an alias github account and even so, everything is tied to it. The idea of this here is to assure the seperation of the machine and the account. This is done with either tor or vpn currently.
  
## Create SSH Key
```bash
ssh-keygen -t ed25519 -f ~/.ssh/name_the_id -C 
```
With the -c flag, you type your password after it not in quotation marks to create a password for it.
  
## Add public key to github
```bash
cat ~/.ssh/name_of_id.pub
```
  
## SSH config for GitHub
```bash
sudo nvim ~/.ssh/config
```
Inside you will add the following:
```bash
Host github-ghost
  HostName: github.com
  User git
  IdentityFile ~/.ssh/name_of_id
  IdentitiesOnly yes
  ServerAliveInterval 60
```
  
## Pull the Repo
You should now be able to test your ssh but make sure VPN or tor is enabled.:
```bash
ssh -T git@github-ghost
```
Pull the repo and then you can set username and email:
```bash
git config user.name "Enter the name you want"
git config user.email "Enter whatever you want here too"
```
  

## Some Extra Tips
In order to check if your VPN or tor is working, you can run:
```bash
curl ifconfig.me
```
  
If you want to make the process of activating a virtual environment that you have, you can use aliases:
```bash
# Normally when you want to activate an env
source  ~/.venvs/myenv/bin/activate
  
# To make it the alias you can run
alias myenv='source ~/.venvs/myenv/bin/activate'
  
# Now, whenever you want to activate your env, you just type
myenv
```

