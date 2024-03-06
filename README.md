### Patches copied from:
### https://github.com/lixingcong/dnsmasq-regex

1. Install normal dnsmasq, make sure it works.
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





