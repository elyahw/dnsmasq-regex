### dnsmasq-regex (fork of: https://github.com/lixingcong/dnsmasq-regex)

1. Install normal dnsmasq, make sure it works (follow local setup instructions).
2. Download dnsmasq v2.90 (https://thekelleys.org.uk/dnsmasq/) and extract it into: `./dnsmasq-regex/dnsmasq`

2.1 Edit `./dnsmasq-regex/Makefile`, change:

`DNSMASQ_COPTS="-DHAVE_REGEX -DHAVE_REGEX_IPSET"`
To:
```
#DNSMASQ_COPTS="-DHAVE_REGEX -DHAVE_REGEX_IPSET"
DNSMASQ_COPTS="-DHAVE_REGEX"
```

3. Now we need to apply the regex patches (from repo https://github.com/lixingcong/dnsmasq-regex) on dnsmasq:
4. Open `./dnsmasq-regex/Makefile`, comment out the two lines `cd dnsmasq && $(MAKE) COPTS=$(DNSMASQ_COPTS)` (we don't want to make the project yet, just apply patches).
5. Run `make` inside `./dnsmasq-regex/`. This will apply the patches.
6. If you compile it now and run it, you will get the error:

`Job for dnsmasq.service failed because the control process exited with error code.
See "systemctl status dnsmasq.service" and "journalctl -xeu dnsmasq.service" for details.`

`journalctl -xeu dnsmasq.service` gives:
`Failed to start dnsmasq - A lightweight DHCP and caching DNS server.`

Also: See error if you check the logs: `journalctl -f` shows:
`dnsmasq: failed to bind DHCP server socket: Permission denied`.

To fix it, you need to enable `dbus`.

7. Enable dbus: Edit `./dnsmasq-regex/dnsmasq/src/config.h` uncomment the line `/* #define HAVE_DBUS */`
8. Open `./dnsmasq-regex/Makefile`, uncomment the two lines we comented earlier, then comment the line `@patch -p 1 -d dnsmasq < $^ && touch $@` to disable applying patches (this is becaues the patches won't be applied if the `config.h` file is modified.
9. Run `make` inside `./dnsmasq-regex/`. This will build dnsmasq.
10. To install it: We don't want to build it again, so edit `./dnsmasq-regex/dnsmasq/Makefile` and change:
`install : all install-common`
To:
```
#install : all install-common
install : install-common
```
Then run `sudo make install`

11. You will need to modify the service path `sudo vim /usr/lib/systemd/system/dnsmasq.service` from `/usr/bin/dnsmasq` to `/usr/local/sbin/dnsmasq` then run `systemctl daemon-reload`

12. Run `systemctl restart dnsmasq` Then `/usr/local/sbin/dnsmasq --version` You should see it saying `regex`


# Original README.md:
## Dnsmasq with regex support

Lastest version: v2.90

patches:
- [001-regex-server.patch](/patches/001-regex-server.patch)
- [002-regex-ipset.patch](/patches/002-regex-ipset.patch)

Inspired by these repos:
- [dnsmasq-regexp_2.76](https://github.com/spacedingo/dnsmasq-regexp_2.76)
- [dnsmasq-regex](https://github.com/cuckoohello/dnsmasq-regex)

Original regex patch for dnsmasq 2.63
- [using regular expressions in server list](http://lists.thekelleys.org.uk/pipermail/dnsmasq-discuss/2013q2/007124.html)
- [dnsmasq-2.63-regex.patch](http://lists.thekelleys.org.uk/pipermail/dnsmasq-discuss/attachments/20130428/b3fc0de0/attachment.obj)

Offical dnsmasq:
- [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/)

## Compile

For Debian/Ubuntu:

```
# Install the dependencies
sudo apt install -y libpcre3-dev libnftables-dev pkg-config

# Clone the repo
git clone https://github.com/lixingcong/dnsmasq-regex
cd dnsmasq-regex

# update the sub-module 'dnsmasq' to latest version
# only update when a newer version is released
bash ./update_submodule.sh

# build it
make

# Run the binary, check if the compile option contains "regex(+ipset,nftset)"
./dnsmasq/src/dnsmasq --version
```

*Tips:* If you do not need the patch of ipset/nftables, just edit the file "Makefile" and build from source again.

Change this line

```
DNSMASQ_COPTS="-DHAVE_REGEX -DHAVE_REGEX_IPSET"
```

to

```
DNSMASQ_COPTS="-DHAVE_REGEX"
```

## Config file example

You could write regex line starts with ':' and ends with ':'

```
server=114.114.114.114
server=/google.com/8.8.8.8
server=/:myvpn[0-9]*\.company\.com:/1.1.1.1
server=/:a[0-9]\.yyy\.com:/#
address=/:a[0-9]\.xxx\.com:/127.0.0.1
ipset=/:.*youtube.*:/test
nftset=/:.*\.google.co.*:/ip#dnsmasq-table#google-ipset
```

The config above will:

- set default upstream server to ```114.114.114.114```
- match normal domain ```google.com``` then forward DNS queries to ```8.8.8.8```
- match domain ```myvpn[0-9]*\.company\.com``` then forward DNS queries to ```1.1.1.1```
- match domain ```a[0-9]\.yyy\.com``` then forward DNS queries ```114.114.114.114``` normally(default upstream server)
- match domain ```a[0-9]\.xxx\.com``` then return DNS record of localhost(to block ads?)
- add ```.*youtube.*``` query answers to ipset ```test```
- add ```.*\.google.co.*``` query answers to nftables set, equivalent to ```nft add element ip dnsmasq-table google-ipset { 172.217.161.74 }```

Here is a example config file: [dnsmasq\_regex\_example.conf](/dnsmasq_regex_example.conf)

Tips:

- A simple script to generate domains configurations: [my-gfwlist](https://github.com/lixingcong/my-gfwlist)

- The regex line ```[a-z]*gle\.com``` will match both ```google.com``` and ```google.com.hk```. Use anchor ```^``` and ```$``` to produce a more precise match.

### Notes for version >= v2.86

Simon, the author of Dnsmasq, has [rewritten](https://thekelleys.org.uk/gitweb/?p=dnsmasq.git;a=commit;h=12a9aa7c628e2d7dcd34949603848a3fb53fce9c) the function to shorten the lookup time for queries. I have to rewrite the patch too. So the domain match function was changed.

If you upgrade from older version(2.85 or older), considering modify your config file. Maybe just simply move lines up and down.😉

The regex lines will generate a linkedlist to match(from top to bottom). If the domain matched both regex servers, DNS query will be forwarded the one which appears first.

Consider the config file below, the domain ```wx.qq.com``` will be forwarded to upstream ```1.1.1.1```, not ```8.8.8.8```

```
server=/:\.qq\.com:/1.1.1.1
server=/:\.qq\.com:/8.8.8.8
```

If the domain matched normal and regex servers, DNS query will be forwarded to the normal one.

Consider the config file below, the domain ```wx.qq.com``` will be forwarded to upstream ```1.1.1.1```, neither ```8.8.8.8``` nor ```1.2.4.8```

```
server=/:w\w?\.qq\.com:/1.2.4.8
server=/qq.com/1.1.1.1
server=/:\.qq\.com:/8.8.8.8
```

## OpenWrt/LEDE package

Please check this page: [dnsmasq-regex-openwrt](https://github.com/lixingcong/dnsmasq-regex-openwrt)



