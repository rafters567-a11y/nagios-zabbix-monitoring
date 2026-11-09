# Server Monitoring with Nagios and Zabbix

Install and test Nagios Core and Zabbix on one Amazon Linux 2023 EC2 instance (1 GB RAM, 2 vCPUs), then study the commands used to monitor servers and services.

## Environment
- Amazon Linux 2023, Nagios Core 4.5.14 (built from source), Zabbix 7.4 (official repo), MariaDB 10.5
- Port 80 was already used by an nginx container, so Apache (`httpd`) listens on **8081**
- Nagios: `http://<ip>:8081/nagios` and Zabbix: `http://<ip>:8081/zabbix`
- Security group: port 8081 open to my IP only

## Nagios install (summary)
1. Install build tools: gcc, make, openssl-devel, gd-devel, httpd, php
2. Set `Listen 8081` in `/etc/httpd/conf/httpd.conf`
3. Build Nagios Core: `./configure --with-httpd-conf=/etc/httpd/conf.d`, `make all`, `make install-groups-users install install-daemoninit install-commandmode install-config install-webconf`
4. Create the web user: `htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin`
5. Build and install nagios-plugins
6. Validate with `nagios -v /usr/local/nagios/etc/nagios.cfg`, then `systemctl enable --now httpd nagios`

Result: 1 host UP, 8 of 8 services OK (Load, Users, HTTP, Ping, Root Partition, SSH, Swap, Total Processes).

## Zabbix install (summary)
1. Add the Zabbix 7.4 repo for Amazon Linux 2023
2. Install zabbix-server-mysql, zabbix-web-mysql, zabbix-apache-conf, zabbix-sql-scripts, zabbix-agent2, mariadb105-server
3. Create database `zabbix` and user, import `server.sql.gz`
4. Low-memory tuning: see `configs/` (MariaDB buffer pool 128M, CacheSize 16M)
5. Start `zabbix-server`, `zabbix-agent2`, `php-fpm`, restart `httpd`, finish the setup wizard

## Trigger demo
Trigger: `last(/Zabbix server/vm.memory.size[pavailable])<20` (severity Warning)
- Create a problem: `sudo systemctl start jenkins` (available memory falls to about 11%)
- Resolve it: `sudo systemctl stop jenkins` (available memory rises to about 38%)
- Zabbix polls about once a minute, so wait 1 to 2 minutes (`sleep 120`)

## Testing from the command line
    /usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /
    /usr/local/nagios/libexec/check_load -w 4,3,2 -c 8,6,4
    zabbix_get -s 127.0.0.1 -k system.cpu.load

Nagios exit codes: 0 = OK, 1 = WARNING, 2 = CRITICAL, 3 = UNKNOWN.

## Monitoring commands studied
| Area | Commands |
|------|----------|
| CPU and load | uptime, top, vmstat, mpstat |
| Memory | free -h, /proc/meminfo |
| Disk | df -h, du -sh, iostat -x |
| Processes | ps aux, pgrep, pstree |
| Network | ss -tulpn, ip -s link, ping, curl -I, nc -zv |
| Services | systemctl status / is-active / --failed |
| Logs | journalctl -u, dmesg |

## Screenshots
See the `screenshots/` folder.

## Notes
- Passwords are not stored in this repo. The Zabbix DB password is redacted in `configs/zabbix_server.conf`.
