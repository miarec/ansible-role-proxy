# ansible-role-sip-proxy
This Ansible role installs a SIP/RTP proxy to load balance multiple Miarec recorders. This is accomplishe using [Kamailio](https://github.com/kamailio/kamailio) and [RTPProxy](https://github.com/sippy/rtpproxy)

## Architecture and Key Functions
Kamailio and RTPProxy will act as a SIP and RTP Proxy between Voice Platforms and MiaRec Recorders,

```
                                                                            +-----------------+
                                                              +------------>| MiaRec Recorder |
                                                              |   SIP/RTP   +-----------------+
                                        +----------------+    |
                 +---------+  SIP/RTP   |                |----+
 { Internet }----| NAT GW  |----------->|   Kamailio /   |    |             +-----------------+
                 +---------+            |     RTPProxy   |    +------------>| MiaRec Recorder |
                                        |                |        SIP/RTP   +-----------------+
                                        +----------------+
                                                      |         +--------------+
                                                      +-------->|  PostgreSQL  |
                                                                +--------------+
```
### NAT
The SIP/RTP Proxy is installed in a private network alongside MiaRec Recorders and PostgreSQL instance. Public Address is NAT'd to SIP-Proxy private address, Kamailio will handle NAT translation of SIP headers and SDP via [`nathelper` module](https://kamailio.org/docs/modules/5.0.x/modules/nathelper.html) and [`rtpproxy` module](https://kamailio.org/docs/modules/5.1.x/modules/rtpproxy.html)

### LoadBalancing
Calls are loadbalanceed between Recorder instances using the [`dispatcher` module](https://kamailio.org/docs/modules/4.3.x/modules/dispatcher.html).

## Requirements
Kamailio requires a database to maintain call state and routing destionations, the ansible playbook assumes this will be a postgreSQL instance available
- PostgreSQL instance
- Postgresql user with permission to create users and databases


## Role Varailble

### Required Varaibles

Host Variables - These varaibles are unique per host, as such they should be supplied as hostvars in the ansible inventory

- `public_ip_address` publive ipv4 address of host
- `private_ip_address` private ipv4 address of host

```ini
[all]
sipproxy0 public_ip_address=1.2.3.4 private_ip_address=10.0.0.1
sipproxy1 public_ip_address=5.6.7.8 private_ip_address=10.0.0.2
```
Group Variables - These varaibles can be assigned to group vars

- `dbhost` URL or IP address of postgres instance
- `dbuser_root` root user with privledge to create user and databases
- `dbpass_root` password for `dbuser_root`

```ini
[sipproxy:vars]
dbhost = database.example.com
dbuser_root = rootuser
dbpass_root = secert
```

### Optional Variables
- `kamailio_version` version of kamailio to install, default = 5.5, https://github.com/kamailio/kamailio/branches

#### Database
- `db_root` name of root database, default = 'miarecdb'
- `dbuser_root` database username, must have privledge to create databases and users, default = 'miarec'
- `dbport` tcp port where postgresQL instance is listening, default = 5432
- `db_kam` name of database that will be created
- `dbuser_kam` username that kamailio modules will with to access PostgreSQL database
- `dbpass_kam` password for `dbuser_kam`

#### Connectivty
- `sip_tcp_port` tcp port that will be used for SIP signaling, default = 5080
- `sip_udp_port` udp port that will be used for SIP signaling, default = 5080
- `rtp_start` starting UDP port for RTP range, default = 20000
- `rtp_stop` ending UDP port for RTP range, default = 30000
- `tcp_max_connections` Maximum number of tcp connections (if the number is exceeded no new tcp connections will be accepted) this includes TLS connection, default = 2048
- `tcp_connection_lifetime` Lifetime in seconds for TCP sessions. TCP sessions which are inactive for longer than tcp_connection_lifetime will be closed by Kamailio, default = 3605
- `enable_NAT_keepalive` It might be required for SIP OPTIONS messages to go through NAT depending on archetecture, if so this will need to be enabled to accuratly rewrite OPTION messages, default = false

#### Loadbalancing
- `disp_set` dispatcher set - a partition name followed by colon and an id of the set or a list of sets from where to pick up destination address
- `disp_alg` disaptcher alg - the algorithm(s) used to select the destination address (variables are accepted)

Recorder host vars for loadbalancing

the following varaibles apply to individual recorder hosts and effect how the sip proxy will interact with them
- `siprec_port` poort recorder will be listening on, default = 5080
- `siprec_protocol` tcp or udp, default = tcp

[documenation here](https://kamailio.org/docs/modules/4.3.x/modules/dispatcher.html#idp51005116)
- `siprec_flags` Various flags that affect dispatcher's behaviour, default = 0
- `siprec_priority` sets the priority in destination list (based on it is done the initial ordering inside the set), default = 0
- `siprec_attrs` extra fields in form of name1=value1;...;nameN=valueN., default = ''

Example
```ini
[all]
rec0.example siprec_port=5060 siprec_protocol=udp siprec_attrs='weight=60' private_ip_address=10.0.0.10
rec1.example siprec_attrs="weight=40" private_ip_address=10.0.0.11
```

result:

- rec0 would recieve siprec on udp:5060 and recieve 60% of calls
- rec1 would recieve siprec on tcp:5080 and recieve 40% of calls

Debug
- `enable_debug` enables the debugger module, default= false
- `debug_level` LOG Levels: 3=DBG, 2=INFO, 1=NOTICE, 0=WARN, -1=ERR, default = 2
- `log_stderror` set to 'yes' to print log messages to terminal, otherwise, debug will be available in syslog, default ='no'
- `enable_jsonrpc` enables [Kamailio RPC Interface](https://www.kamailio.org/w/2020/11/kamailio-jsonrpc-client-with-http-rest-interface/), default = false

TLS (This feature is not tested)
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
- name: Deploy Kamailio and Rtpproxy.
  hosts:
    - sipproxy
  become: true
  pre_tasks:
    - set_fact:
        tmp_dest:
          name: "{{ hostvars[item].inventory_hostname }}"
          ip: "{{ hostvars[item].private_ip_address }}"
          port: "{{ hostvars[item].siprec_port | default('5080') }}"
          protocol: "{{ hostvars[item].siprec_protocol | default('tcp') }}"
          flags: "{{ hostvars[item].siprec_flags | default('0') }}"
          priority: "{{ hostvars[item].siprec_priority | default('0') }}"
          attrs: "{{ hostvars[item].siprec_attrs | default('') }}"
      with_items: "{{ groups.recorder }}"
      register: _tmp_dispatch_dest

    - set_fact:
        dispatcher_destinations: "{{ _tmp_dispatch_dest.results | selectattr('ansible_facts', 'defined') | map(attribute='ansible_facts.tmp_dest') | list }}"

  roles:
    - role: 'ansible-role-sip-proxy'
  tags: 'sipproxy'
```

Example Inventory
```ini
[all]
sipproxy0 ansible_host=10.0.0.1 public_ip_address=1.2.3.4 private_ip_address=10.0.0.1
sipproxy1 ansible_host=10.0.0.2 public_ip_address=5.6.7.8 private_ip_address=10.0.0.2

[sipproxy]
sipproxy0
sipproxy1

[sipproxy:vars]
dbhost = database.example.com
dbrootpass = secert
enable_debug = true
debug_level = 3
```