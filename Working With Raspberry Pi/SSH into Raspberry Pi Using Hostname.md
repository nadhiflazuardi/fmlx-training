1. Edit /etc/systemd/resolved.conf
	Under Resolve section header, add:
```bash
	MulticastDNS=yes
	LLMNR=no
	DNSStubListener=yes
```
2. Restart systemd-resolved
```bash
	sudo systemctl restart systemd-resolved
```
3. Try pinging or ssh using <hostname>.local