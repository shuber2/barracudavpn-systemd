# Systemd integration of Barracuda VPN Linux client

This projects provides a systemd integration of the [Barracuda VPN Client for Linux and OpenBSD](https://documentation.campus.barracuda.com/wiki/spaces/NACv50/pages/2162985/Installing+the+Barracuda+VPN+Client+for+Linux+and+OpenBSD).

## Installation

```sh
# Download the barracudavpn linux package and unpack. On Debian system you
# install it by
dpkg -i barracudavpn_5.3.6_amd64.deb
# Then you need to put the barracudavpn config to /etc/barracudavpn or the tui:
barracudavpn

# Increase fs.mqueue.msg_max=16
cp 99-barracuda-mqueue.conf /etc/sysctl.d/
sysctl --system

# Install wrapper script
cp -a barracudavpn-wrapper /usr/local/bin
# Change USER and PASSWORD in this file.
# Attention: Check whether you are happy with the read permissions on this
# file! Maybe you want to replace PASSWORD by a password-fetch-command.
nano /usr/local/bin/barracudavpn-wrapper

# Install and launch systemd unit
cp barracudavpn.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now barracudavpn.service

```


## Technical background

**Client bug on mqueue.**
The barracudavpn client needs `fs.mqueue.msg_max=16`, but only when started
from a systemd unit file. Obviously, different code paths are involved then. I found this by calling `strace -f -e mq_open,mq_notify,clone,unshare barracudavp` to debug the error message

 ```
/usr/local/bin/barracudavpn[xxx]: Could not open QUEUE UI2WORKER. Error: Invalid argument
 ```
 which then gave
 ```
 mq_open("queue1", O_WRONLY|O_CREAT|O_NONBLOCK, 0600, {mq_flags=0, mq_maxmsg=16, mq_msgsize=8192, mq_curmsgs=0}) = -1 EINVAL (Invalid argument)
 ```


**DNS fixup.**
The barracudavpn sets up DNS to route all DNS requests. In
`barracudavpn-wrapper`, we fix this up after `barracudavpn --start` is done.
You probably want to adapt this to your use case. This fix relies on
`systemd-resolved` as local DNS resolver.
