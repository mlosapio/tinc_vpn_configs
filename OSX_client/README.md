## Installing tinc on OSX


In this example the following info is assumed:

- the OSX initiator tunnel IP is 100.64.0.102/32
- the responder I'm connecting to is at 100.64.0.2/32
- I want to push all my traffic to the responder VPN node
- I am not running this in vmware
- my netname is mcloud

### On the OSX Client

*Think of this as the VPN initiator*

1. Make sure homebrew is installed, if it is, skip this step
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

2. Install tinc! (and LZO and the other dependencies)
```
brew install tinc
brew cask install tuntap
```

3. Create the folder hierarchy
```
mkdir -p /usr/local/etc/tinc/mcloud/hosts
```

4. Create the config

- Pick a name for this client. You should put that name where I have $NAME$
- ConnectTo is the name of the hub. You should put the name of your hub in $HUB$


```
cat << EOF > /usr/local/etc/tinc/mcloud/tinc.conf
Name = $NAME$
ConnectTo = $HUB$
PrivateKeyFile = /usr/local/etc/tinc/mcloud/rsa_key.priv
Device = /dev/tap0
EOF
```

5. Run the tincd key generation command and accept the defaults

```
/usr/local/sbin/tincd -n mcloud -K
```

6. Add your OSX host file to the VPN responder

Go into /usr/local/etc/tinc/mcloud/hosts and you'll find a file
that's been created named $NAME$ from the earlier step.

Add the header to the file and pass it along to the hosts directory
of the responder

```
Subnet = 100.64.0.102/32

-----BEGIN RSA PUBLIC KEY-----
...
-----END RSA PUBLIC KEY-----
```

7. Create the tinc-up file

This will be executed every time tinc connects

```
cat << 'EOF' > /usr/local/etc/tinc/mcloud/tinc-up
#!/bin/sh

ip=$(route -n get default | grep gateway | awk -F':' '{print $2}')

ifconfig $INTERFACE 100.64.0.102 netmask 255.255.255.0

route add -net 100.64.0.2 100.64.0.102 255.255.255.0

#make all traffic to be sent over our vpn
route add -net 0.0.0.0 100.64.0.2 128.0.0.0
route add -net 128.0.0.0 100.64.0.2 128.0.0.0

# keep normal route to server - This is helpful for VMWARE
route add -net <RESPONDERIP> ${ip} 255.255.255.255
EOF

```

8. Create the tinc-down file

This will be run when tinc disconnects

```
cat << 'EOF' > /usr/local/etc/tinc/mcloud/tinc-down
#!/bin/sh
  
ifconfig $INTERFACE down

route delete -net 100.64.0.2 100.64.0.102 255.255.255.0

#make all traffic to be sent over our vpn
route delete -net 0.0.0.0 100.64.0.2 128.0.0.0
route delete -net 128.0.0.0 100.64.0.2 128.0.0.0

# keep normal route to server - This is helpful for VMWARE
route delete -net 34.236.15.133/32
EOF
```

9. Create reload script

```
cat << EOF > /usr/local/etc/tinc/mcloud/reload_tinc.sh 
#!/bin/sh

launchctl stop tinc.vpn
launchctl start tinc.vpn
EOF
```

10. Create the plist file

(you may have to run this as root)

```
cat << 'EOF' > /Library/LaunchDaemons/mcloud.tinc.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
    <key>Label</key>
    <string>tinc.vpn</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/sbin/tincd</string>
        <string>-n</string>
        <string>mcloud</string>
        <string>-D</string>
        <string>-d</string>
        <string>2</string>
    </array>
    <key>StandardOutPath</key>
    <string>/var/log/tinc.log</string>
    <key>StandardErrorPath</key>
    <string>/var/log/tinc.log</string>
</dict>
</plist>
EOF
```

11. Create plist to montor for network changes

```
cat << EOF > /Library/LaunchDaemons/vpn.watcher.plist 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>vpn.watcher</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/etc/tinc/mcloud/reload_tinc.sh</string>
    </array>
    <key>WatchPaths</key>
    <array>
        <string>/var/run/resolv.conf</string>
    </array>
</dict>
</plist>
EOF
```

12. Add in your responder file into the hosts directory

13. Housekeeping items

```
mkdir -p  /usr/local/Cellar/tinc/1.0.35/var/run/
chmod +x /usr/local/etc/tinc/mcloud/tinc-up
chmod +x /usr/local/etc/tinc/mcloud/tinc-down
chmod +x /usr/local/etc/tinc/mcloud/reload_tinc.sh 
sudo launchctl load -w /Library/LaunchDaemons/mcloud.tinc.plist
```

14. Make sure your DNS record is not your local resolver since you're now
tunneling all the traffic!
