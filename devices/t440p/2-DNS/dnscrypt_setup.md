# SETTING UP DNSCRYPT ON DEBIAN
On debian, it's not as simple as an install. You need to download the latest release from github using wget, extract the files, set ownership, and change the configs.
  
### Download Latest Release (wget)
```
cd /opt
sudo wget https://github.com/DNSCrypt/dnscrypt-proxy/releases/download/2.1.5/dnscrypt-proxy-linux_x86_64-2.1.5.tar.gz
```
The /opt folder is for optional application software packages. Here you install additional programs that are not part of the standard system installation. On the other hand `wget` is a utility for downloading files from the internet. Supports various protocols like HTTP, HTTPS, and FTP.
  
### Extract & Move
```
sudo tar -xvzf dnscrypt-proxy-linux_x86_64-*.tar.gz
sudo mv linux-x86_64 dnscrypt-proxy
sudo chown -R root:root /opt/dnscrypt-proxy
cd dnscrypt-proxy
```
`tar` is a utility for handling tape archive files. Mainly for extracting and creating archive files. `-xvzf` are all flags:
- -x : Extract files from an archive
- -v : Verbose mode, meaning tar will output the names of files as they are being extracted.
- -z : Specifies the archive is compressed with gzip meaning files usually having `.tar.gz` or `tgz`.
- -f : Indicates that a filename follows so it's specifying the location of the file.
  
`chown` is used to change the ownership of files or directories
  
`-R` is recursive so it will apply the changes to all contents in the folder.
  
`root:root` specifies the new owner and group:
- The first `root` indicates the owner of the files
- The second `root` indicates the group that will own the files
  
### Set Up Configuration
```
sudo cp dnscrypt-proxy.toml.example dnscrypt-proxy.toml
sudo nvim dnscrypt-proxy.toml
  
# Edit the following:
server_names = ['cloudflare', 'quad9-dnscrypt-ip4-filter-pri']
listen_addresses = ['127.0.0.1:53']
require_dnssec = true
require_nolog = true
require_nofilter = true
```
Just taking the example one and creating a copy to edit for your own configurations.
  
What this does is it binds dnscrypt on localhost port 53 so dns queries will be sent from there. You are selecting DNS resolvers that will be used and these specifically support encrypted DNS improving privacy. When we set require_nofilter to true, we ensure DNS data and no logs and in general protect against spoofing and tracking.
  
### Manual Testing
```
sudo /opt/dnscrypt-proxy -config /opt/dnscrypt-proxy/dnscrypt-proxy.toml
```
This will runthe dnscrypt using the specified config as root in order to verify thatit can start correctly, connect to resolvers, and serve DNS queries. Logs showing resolvers and proxy means success.
  
### Setting System DNS to use dnscrypt
```
sudo rm -f /etc/resolv.conf
echo "nameserver 127.0.0.1 | sudo tee /etc/resolv.conf
```
This deletes the existing DNS config and replaces it with a single nameserver pointing to localhost and this makes the system query dnscrypt for all DNS lookups.
  
### Creating a systemd service for dnscrypt
```
sudo nvim /etc/systemd/system/dnscrypt-proxy.service
  
# Insert the following:
[Unit]
Description=dnscrypt-proxy
After=network.target
  
[Service]
ExecStart=/opt/dnscrypt-proxy/dnscrypt-proxy -config /opt/dnscrypt-proxy/dnscrypt-proxy.toml
  
[Install]
WantedBy=multi-user.target
```
This defines a service to manage dnscrypt and therefore allows it to start automatically at boot and restart on failure.
  
### Enabling and starting the service
```
sudo systemctl daemon-reexec
sudo systemctl enable --now dnscrypt-proxy
sudo systemctl status dnscrypt-proxy
```
First reload the systemd manager config, enables dnscrypt to start at boot, starts it, and checks status. 
  
### Testing DNS
```
dig duckduckgo.com
```
Here you send a DNS query using the system's current DNS setup and confirm it works.

### EXTRA COMPATIBILITY WITH ANONSURF
You cannot use both anonsurf and dnscrypt at the same time as both services fight for the /etc/resolv.conf as to who controls the DNS. Therefore the best way is to create a custom script when running anonsurf in order to efficiently switch between using dnscrypt daily and using anonsurf for more secure tasks. Create two custom scripts to switch between the two. Once completed, you can run:
```
tor-on #to disable dnscrypt and enable anonsurf
tor-off #to disable anonsurf and reenable dnscrypt
```
