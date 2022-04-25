# ansible-role-sip-proxy
ansible role to install a sip proxy for Miarec Recorders

Kamailio requires a database to maintain call state and routing destionations, the ansible playbook assumes a seperate database that has already been created

## Description

## Requirements

postgres instance
postgresql user with permission to create users and database


## Role Varailble

### Required Varaibles

- `public_ip_address` publive ipv4 address of host
- `private_ip_address` private ipv4 address of host

These varaibles are unique per host, as such they should be supplied as hostvars in the ansible inventory
```ini
[all]
sipproxy0 public_ip_address=1.2.3.4 private_ip_address=10.0.0.1
sipproxy1 public_ip_address=5.6.7.8 private_ip_address=10.0.0.2
```

- `dbhost` URL or IP address of postgres instance
- `dbrootuser` database username, must have privledge to create databases and users
- `dbrootpass` password for dbroot user

These varaibles can be assigned to a group vars
```ini
[sip-proxy:vars]
dbhost = database.example.com
dbrootuser = rootuser
dbrootpass = secert
```
### Optional Variables

- `kamailio_version` version of kamailio to install, default = 5.5, https://github.com/kamailio/kamailio/branches

Database
- `dbport` tcp port where postgresQL instance is listening, default = 5432
- `kam_dbname` name of database that will be created
- `kam_dbuser` username that kamailio modules will with to access PostgreSQL database
- `kam_dbpass` password for `kam_dbuser`

Connectivty
- `sip_tcp_port` tcp port that will be used for SIP signaling, default = 5080
- `sip_udp_port` udp port that will be used for SIP signaling, default = 5080
- `rtp_start` starting UDP port for RTP range, default = 20000
- `rtp_stop` ending UDP port for RTP range, default = 30000
- `tcp_max_connections` Maximum number of tcp connections (if the number is exceeded no new tcp connections will be accepted) this includes TLS connection, default = 2048
- `tcp_connection_lifetime` Lifetime in seconds for TCP sessions. TCP sessions which are inactive for longer than tcp_connection_lifetime will be closed by Kamailio, default = 3605
- `enable_NAT_keepalive` It might be required for SIP OPTIONS messages to go through NAT depending on archetecture, if so this will need to be enabled to accuratly rewrite OPTION messages, default = false

Loadbalancing
- `disp_set` dispatcher set - a partition name followed by colon and an id of the set or a list of sets from where to pick up destination address
- `disp_alg` disaptcher alg - the algorithm(s) used to select the destination address (variables are accepted)

Debug
- `enable_debug` enables the debugger module
- `debug_level` LOG Levels: 3=DBG, 2=INFO, 1=NOTICE, 0=WARN, -1=ERR, default = 2
- `log_stderror` set to 'yes' to print log messages to terminal, otherwise, debug will be available in syslog, default ='no'
- `enable_jsonrpc` enables [Kamailio RPC Interface](https://www.kamailio.org/w/2020/11/kamailio-jsonrpc-client-with-http-rest-interface/), default = false

TLS
- `enable_tls` Enables TLS module, default = false
- `tls_max_connections` Maxium number of tls connection, must not excees tecp_max_connections, must not exceed tcp_max_connections, default = 2048

Antiflood
- `enable_anitflood` [Enables Anitflood](https://www.kamailio.org/docs/modules/devel/modules/pike.html) Automatically ban IP addresses with excessive messaging, default = true
- `pike_sample_time` Time period in seconds used for sampling (or the sampling accuracy). The smaller the better, but slower, default = 2
- `pike_reqs_density` How many requests should be allowed per sampling_time_unit before blocking all the incoming request from that, default = 30
- `pike_remove_latency` Specifies for how long the IP address will be kept in memory after the last request from that IP address. It's a sort of timeout value, in seconds, default = 120
- `ipban_expire` Time in seconds ip will be stored in ipban table, default = 300


## Example

Example Playbook
```yaml
- name: Deploy Kamailio and rtpproxy
  hosts:
    - sip-proxy
  become: true
  roles:
    - role: 'ansible-role-sip-proxy'
  tags: 'sip-proxy'
```

Example Inventory
```ini
[all]
sipproxy0 ansible_host=10.0.0.1 public_ip_address=1.2.3.4 private_ip_address=10.0.0.1
sipproxy1 ansible_host=10.0.0.2 public_ip_address=5.6.7.8 private_ip_address=10.0.0.2

[sip-proxy]
sipproxy0
sipproxy1

[sip-proxy:vars]
dbhost = database.example.com
dbrootuser = rootuser
dbrootpass = secert
enable_debug = true
debug_level = 3
```