# ansible-role-sip-proxy
ansible role to install a sip proxy for Miarec Recorders

Kamailio requires a database to maintain call state and routing destionations, the ansible playbook assumes a seperate database that has already been created

## Description

## Requirements

## Role Varailble

### Required Varaibles

`public_ip_address` publive ipv4 address of host
`private_ip_address` private ipv4 address of host
These varaibles are unique per host, as such they should be supplied as hostvars in the ansible inventory
```
[all]
sipproxy0 public_ip_address=1.2.3.4 private_ip_address=10.0.0.1
sipproxy1 public_ip_address=5.6.7.8 private_ip_address=10.0.0.2
```

`dbhost` URL or IP address of postgres instance
`dbrootuser` database username, must have privledge to create databases and users
`dbrootpass` password for dbroot user
These varaibles can be assigned to a group vars
```
[sip-proxy:vars]
dbhost = database.example.com
dbrootuser = rootuser
dbrootpass = secert
```
### Optional Variables

`kamailio_version` version of kamailio to install, default = 5.5, https://github.com/kamailio/kamailio/branches
`dbport` tcp port where postgresQL instance is listening, default = 5432
kam_dbname: kamailio
kam_dbuser: kamailio
kam_dbpass: kamailiorw

# SIP Variables
port_tcp: 5080
port_udp: 5080

rtp_start: 20000
rtp_stop: 30000

tcp_max_connections: 2048   # Maximum number of tcp connections (if the number is exceeded no new tcp connections will be accepted)
tcp_connection_lifetime: 3605   # Lifetime in seconds for TCP sessions. TCP sessions which are inactive for longer than tcp_connection_lifetime will be closed by Kamailio



# Dispatcher Variables
disp_set: 1  # dispatcher set - a partition name followed by colon and an id of the set or a list of sets from where to pick up destination address
disp_alg: 4  # disaptcher alg - the algorithm(s) used to select the destination address (variables are accepted)



# Configuration variables
enable_debug: false
debug_level: 2   # LOG Levels: 3=DBG, 2=INFO, 1=NOTICE, 0=WARN, -1=ERR
log_stderror: "no"   # set to 'yes' to print log messages to terminal or use '-E' cli option, otherwise, debug will be available in syslog

enable_jsonrpc: false   # enables Kamailio RPC Interface

enable_tls: false   # Enables TLS module
tls_max_connections: 2048   # Maxium number of tls connection, must not excees tecp_max_connections, requires 'enable_tls'=true

# Antiflood Configuration
# Antiflood, will automatically ban IP addresses that exceed configured limits
# Documention about pike module https://www.kamailio.org/docs/modules/devel/modules/pike.html
enable_anitflood: true
pike_sample_time: 2   # Time period in seconds used for sampling (or the sampling accuracy). The smaller the better, but slower
pike_reqs_density: 30   # How many requests should be allowed per sampling_time_unit before blocking all the incoming request from that
pike_remove_latency: 120   # Specifies for how long the IP address will be kept in memory after the last request from that IP address. It's a sort of timeout value, in seconds
ipban_expire: 300   # Time in seconds ip will be stored in ipban table


enable_NAT_keepalive: false # It might be required to NAT SIP OPTIONS messages depending on archetecture

## Example