
# Postgresql HA cluster in docker using Patroni and Consul.

This project accompanies a blog located here: https://digitalis.io/blog/postgresql/using-consul-for-postgresql-ha-with-no-load-balancers-needed/

This is a minimalistic demo application showing HA Postgresql cluster managed by Patroni and Consul with no extra load balancers.
The entire setup runs in `docker-compose` and requires `make` utility for helper commands.

For Postgresql and application containers the vanilla `ubuntu:20.04` image is used and for Consul servers the official `consul:1.10.3` image.

Patroni nodes run 2 services under supervisor: patroni and consul in agent mode.

Consul nodes run consul service in server mode.

Application nodes run 3 services under supervisor: consul in agent mode, dnsmasq and application itself.

Application connects to a current Postgresql master node and reports to the log its IP address.


```
                 ┌───dns:8600──┐                    ┌─────────────┐                            
                 │             │                    │             │                            
                 │             │                    │             │                            
       ┌──────────────────┐    │          ┌──────────────────┐    │                            
       │                  │    │          │                  │    │                            
       │  application01   ├─┐  │  ┌───────│  application02   ├─┐  │                            
       │                  │ │  │  │       │                  │ │  │                            
       └───┬──────────────┘ │◀─┘  │       └───┬──────────────┘ │◀─┘                            
       │   │    consul-agent│     │           │    consul-agent│                               
       │   └────────────────┘     │           └────────────────┘                               
       │            │             │                    │                                       
       │            │             │       ┌────────────┘                                       
       │            │             │       │                                                    
       │            │             │       ▼                                                    
       │            │             │.─────────────.                                             
       │            │           ,─'               '─.                                          
 postgres:5432      └─────────▶(      consul01       )                                         
       │                        `─┬.             _.─'──.                                       
       │                          │ `───────────'       '─.                                    
       │                          │  (      consul02       )◀───────────rpc:8300──────────────┐
       │                          │   `──.             _.─'──.                                │
       │                          │       `───────────'       '─.                             │
       │                          │ ┌─────▶(      consul03       )◀─┐                         │
       │                          │ │       `──.             _.─'   │                         │
       │                          │ │           `───────────'       │                         │
       │                          │ │                               │                         │
       ▼                          │ │                               │                         │
       ╔═══════════════════╗      │ │    ╔═══════════════════╗      │   ╔═══════════════════╗ │
       ║ PG                ║      │ │    ║ PG                ║      │   ║ PG                ║ │
       ║                   ║      │ │    ║                   ║      │   ║                   ║ │
    ┌──║     patroni01     ║◀─────┘ │ ┌──║     patroni02     ║      │┌──║     patroni03     ║ │
    │  ║                   ╠─┐      │ │  ║                   ╠─┐    ││  ║                   ╠─┐
    │  ║         [leader]  ║ │      │ │  ║         [follower]║ │    ││  ║         [follower]║ │
    │  ╚════╦══════════════╝ │──────┘ │  ╚════╦══════════════╝ │────┘│  ╚════╦══════════════╝ │
    │       │    consul-agent│        │       │    consul-agent│     │       │    consul-agent│
    │       └────────────────┘        │       └────────────────┘     │       └────────────────┘
    │                ▲                │                ▲             │                ▲        
    │   http:8500    │                │                │             │                │        
    └────────────────┘                └────────────────┘             └────────────────┘        
```

## Usage

### `make build`

```sh
Pull and build container images. This make take some time when running for first time.
consul01 uses an image, skipping
consul02 uses an image, skipping
consul03 uses an image, skipping
Building patroni01
Sending build context to Docker daemon  12.29kB
Step 1/14 : FROM ubuntu:20.04
20.04: Pulling from library/ubuntu
7b1a6ab2e44d: Pull complete 
Digest: sha256:626ffe58f6e7566e00254b638eb7e0f3b11d4da9675088f4781a50ae288f3322
Status: Downloaded newer image for ubuntu:20.04
 ---> ba6acccedd29
Step 2/14 : ARG DEBIAN_FRONTEND=noninteractive
 ---> Running in 0200e7404741
Removing intermediate container 0200e7404741
 ---> caa769cf7322
Step 3/14 : RUN apt update
 ---> Running in 7233f5befb28

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Get:1 http://security.ubuntu.com/ubuntu focal-security InRelease [114 kB]
...
Removing intermediate container 4310e7ecc28f
 ---> ee0f2dec7130
Step 5/14 : RUN wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
 ---> Running in 230b7dea74c7
Warning: apt-key output should not be parsed (stdout is not a terminal)
OK
Removing intermediate container 230b7dea74c7
 ---> 990a41aafb51
Step 6/14 : RUN echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >  /etc/apt/sources.list.d/pgdg.list
 ---> Running in 3248bd7cad00
Removing intermediate container 3248bd7cad00
 ---> 55526e9d7da3
Step 7/14 : RUN apt update && apt -y install postgresql-13 postgresql-client-13 patroni python3-consul
 ---> Running in 12fed402b068
...
Success. You can now start the database server using:

    pg_ctlcluster 13 main start
...
Processing triggers for dbus (1.12.16-2ubuntu2.1) ...
Removing intermediate container 6fdf6e8ceebd
 ---> 23bf8193a16b
Step 5/10 : RUN pip3 install psycopg2
 ---> Running in 88b951792136
Collecting psycopg2
  Downloading psycopg2-2.9.1.tar.gz (379 kB)
Building wheels for collected packages: psycopg2
  Building wheel for psycopg2 (setup.py): started
...
Successfully built psycopg2
Installing collected packages: psycopg2
Successfully installed psycopg2-2.9.1
Removing intermediate container 88b951792136
 ---> 72358ff035c6
Step 6/10 : RUN wget --quiet https://releases.hashicorp.com/consul/1.10.3/consul_1.10.3_linux_amd64.zip
 ---> Running in c89270de26f4
Removing intermediate container c89270de26f4
 ---> 71d535a8fcf4
Step 7/10 : RUN unzip consul_1.10.3_linux_amd64.zip && mv consul /usr/local/bin/ && rm consul_1.10.3_linux_amd64.zip
 ---> Running in a7e3f1ecdf96
Archive:  consul_1.10.3_linux_amd64.zip
  inflating: consul                  
Removing intermediate container a7e3f1ecdf96
 ---> f8661a59d1fc
Step 8/10 : RUN mkdir -p /etc/consul/certs && mkdir -p /consul/data
 ---> Running in c0d4b8434c6e
Removing intermediate container c0d4b8434c6e
 ---> 8be64b493b83
Step 9/10 : COPY ./entrypoint.sh /usr/bin/entrypoint.sh
 ---> e9dbb8bbc570
Step 10/10 : CMD ["/usr/bin/entrypoint.sh"]
 ---> Running in f0dc27b15cd0
Removing intermediate container f0dc27b15cd0
 ---> d4d3a9c9b3b1
Successfully built d4d3a9c9b3b1
Successfully tagged postgresql-consul-demo_application01:latest
Building application02
Sending build context to Docker daemon  10.24kB
Step 1/10 : FROM ubuntu:20.04
 ---> ba6acccedd29
Step 2/10 : ARG DEBIAN_FRONTEND=noninteractive
 ---> Using cache
 ---> caa769cf7322
Step 3/10 : RUN apt update
 ---> Using cache
 ---> 647290579e00
Step 4/10 : RUN apt -y install vim python3-pip libpq-dev supervisor wget unzip gettext-base dnsmasq iputils-ping dnsutils
 ---> Using cache
 ---> 23bf8193a16b
Step 5/10 : RUN pip3 install psycopg2
 ---> Using cache
 ---> 72358ff035c6
Step 6/10 : RUN wget --quiet https://releases.hashicorp.com/consul/1.10.3/consul_1.10.3_linux_amd64.zip
 ---> Using cache
 ---> 71d535a8fcf4
Step 7/10 : RUN unzip consul_1.10.3_linux_amd64.zip && mv consul /usr/local/bin/ && rm consul_1.10.3_linux_amd64.zip
 ---> Using cache
 ---> f8661a59d1fc
Step 8/10 : RUN mkdir -p /etc/consul/certs && mkdir -p /consul/data
 ---> Using cache
 ---> 8be64b493b83
Step 9/10 : COPY ./entrypoint.sh /usr/bin/entrypoint.sh
 ---> Using cache
 ---> e9dbb8bbc570
Step 10/10 : CMD ["/usr/bin/entrypoint.sh"]
 ---> Using cache
 ---> d4d3a9c9b3b1
Successfully built d4d3a9c9b3b1
Successfully tagged postgresql-consul-demo_application02:latest
```

```sh
$ docker image ls
REPOSITORY                                  TAG            IMAGE ID       CREATED         SIZE
postgresql-consul-demo_application02        latest         d4d3a9c9b3b1   2 minutes ago   646MB
postgresql-consul-demo_application01        latest         d4d3a9c9b3b1   2 minutes ago   646MB
postgresql-consul-demo_patroni01            latest         aa547a52f3f4   3 minutes ago   900MB
postgresql-consul-demo_patroni02            latest         aa547a52f3f4   3 minutes ago   900MB
postgresql-consul-demo_patroni03            latest         aa547a52f3f4   3 minutes ago   900MB
ubuntu                                      20.04          ba6acccedd29   5 days ago      72.8MB
...
```

### `make start`

```sh
Start cluster
Creating network "postgresql-consul-demo_pglab" with driver "bridge"
Pulling consul01 (consul:1.10.3)...
1.10.3: Pulling from library/consul
4e9f2cdf4387: Already exists
...
b694e38e70f2: Pull complete
Digest: sha256:7b0effaf3faf9ce134be8648e3cf5f82263a52cd80f5395452a9ca69700f0357
Status: Downloaded newer image for consul:1.10.3
Creating patroni03     ... done
Creating application01 ... done
Creating patroni01     ... done
Creating patroni02     ... done
Creating consul01      ... done
Creating application02 ... done
Creating consul03      ... done
Creating consul02      ... done
```


### `make status`

```sh
============================== Docker status ==============================
    Name                   Command               State                                                         Ports
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------
application01   /usr/bin/entrypoint.sh           Up
application02   /usr/bin/entrypoint.sh           Up
consul01        docker-entrypoint.sh agent ...   Up      8300/tcp, 8301/tcp, 8301/udp, 8302/tcp, 8302/udp, 0.0.0.0:8500->8500/tcp, 0.0.0.0:8600->8600/tcp,
                                                         0.0.0.0:8600->8600/udp
consul02        docker-entrypoint.sh agent ...   Up      8300/tcp, 8301/tcp, 8301/udp, 8302/tcp, 8302/udp, 8500/tcp, 8600/tcp, 8600/udp
consul03        docker-entrypoint.sh agent ...   Up      8300/tcp, 8301/tcp, 8301/udp, 8302/tcp, 8302/udp, 8500/tcp, 8600/tcp, 8600/udp
patroni01       /usr/bin/entrypoint.sh           Up
patroni02       /usr/bin/entrypoint.sh           Up
patroni03       /usr/bin/entrypoint.sh           Up

============================== Consul status ==============================
Node           Address          Status  Type    Build   Protocol  DC   Segment
consul01       172.19.0.7:8301  alive   server  1.10.3  2         dc1  <all>
consul02       172.19.0.8:8301  alive   server  1.10.3  2         dc1  <all>
consul03       172.19.0.6:8301  alive   server  1.10.3  2         dc1  <all>
application01  172.19.0.3:8301  alive   client  1.10.3  2         dc1  <default>
application02  172.19.0.9:8301  alive   client  1.10.3  2         dc1  <default>
patroni01      172.19.0.4:8301  alive   client  1.10.3  2         dc1  <default>
patroni02      172.19.0.5:8301  alive   client  1.10.3  2         dc1  <default>
patroni03      172.19.0.2:8301  alive   client  1.10.3  2         dc1  <default>

============================== Patroni status =============================
+ Cluster: pglab (7021393365364887600) -----+----+-----------+
| Member    | Host      | Role    | State   | TL | Lag in MB |
+-----------+-----------+---------+---------+----+-----------+
| patroni01 | patroni01 | Replica | running |  1 |         0 |
| patroni02 | patroni02 | Replica | running |  1 |         0 |
| patroni03 | patroni03 | Leader  | running |  1 |           |
+-----------+-----------+---------+---------+----+-----------+
```

> Note: if patroni nodes are experiencing error: "patroni private key file must
> be owned by the database user or root", ensure the permissions are correct for
> the PostgreSQL server certificate and key file. The files should be owned by
> the database user (default: `postgres`), and its permissions should be `0400`.
>
> To fix, run these commands on the container host:
> ```sh
> $ chown -R 102:102 patroni/certs
> $ chmod -R 0600 patroni/certs
> ```

> Note: if application nodes are experiencing error: "WARNING: password file
> "/root/.pgpass" has group or world access; permissions should be u=rw (0600)
> or less ... ERROR fe_sendauth: no password supplied".
>
> Fix it by running these commands on the container host:
> ```sh
> $ chmod 0600 application/pgpass
> ```

### `make stop`

```
Stop cluster
Stopping application01 ... done
Stopping application02 ... done
Stopping patroni01     ... done
Stopping patroni03     ... done
Stopping patroni02     ... done
Stopping consul01      ... done
Stopping consul03      ... done
Stopping consul02      ... done
```

### `make destroy`

```
Force destroy cluster
Stopping application01 ... done
Stopping application02 ... done
Stopping patroni01     ... done
Stopping patroni03     ... done
Stopping patroni02     ... done
Stopping consul03      ... done
Stopping consul02      ... done
Stopping consul01      ... done
Removing application01 ... done
Removing application02 ... done
Removing patroni01     ... done
Removing patroni03     ... done
Removing patroni02     ... done
Removing consul03      ... done
Removing consul02      ... done
Removing consul01      ... done
Removing network postgresql-consul-demo_pglab
```

### `make recreate`

```
Destroy and recreate cluster
Stopping application02 ... done
Stopping application01 ... done
Stopping patroni01     ... done
Stopping patroni03     ... done
Stopping consul03      ... done
Stopping patroni02     ... done
Stopping consul02      ... done
Stopping consul01      ... done
Going to remove application02, application01, patroni01, patroni03, consul03, patroni02, consul02, consul01
Removing application02 ... done
Removing application01 ... done
Removing patroni01     ... done
Removing patroni03     ... done
Removing consul03      ... done
Removing patroni02     ... done
Removing consul02      ... done
Removing consul01      ... done
Creating consul02      ... done
Creating consul01      ... done
Creating patroni02     ... done
Creating application01 ... done
Creating consul03      ... done
Creating application02 ... done
Creating patroni03     ... done
Creating patroni01     ... done
```

### `make switchover`

```
Force patroni switchover to another node
Current cluster topology
+ Cluster: pglab (7021401786240585771) -----+----+-----------+
| Member    | Host      | Role    | State   | TL | Lag in MB |
+-----------+-----------+---------+---------+----+-----------+
| patroni01 | patroni01 | Replica | running |  1 |         0 |
| patroni02 | patroni02 | Replica | running |  1 |         0 |
| patroni03 | patroni03 | Leader  | running |  1 |           |
+-----------+-----------+---------+---------+----+-----------+
2021-10-21 06:32:54.72729 Successfully switched over to "patroni02"
+ Cluster: pglab (7021401786240585771) -----+----+-----------+
| Member    | Host      | Role    | State   | TL | Lag in MB |
+-----------+-----------+---------+---------+----+-----------+
| patroni01 | patroni01 | Replica | running |  1 |         0 |
| patroni02 | patroni02 | Leader  | running |  1 |           |
| patroni03 | patroni03 | Replica | stopped |    |   unknown |
+-----------+-----------+---------+---------+----+-----------+
```

### `make applogs`

```
Attaching to application02, application01
application01    | 2021-04-19 16:42:32,564 CRIT Supervisor is running as root.  Privileges were not dropped because no user is specified in the config file.  If you intend to run as root, you can set user=root in the config file to avoid this message.
application01    | 2021-04-19 16:42:32,564 INFO Included extra file "/etc/supervisor/conf.d/application.conf" during parsing
application01    | 2021-04-19 16:42:32,564 INFO Included extra file "/etc/supervisor/conf.d/consul.conf" during parsing
application01    | 2021-04-19 16:42:32,572 INFO RPC interface 'supervisor' initialized
application01    | 2021-04-19 16:42:32,573 CRIT Server 'unix_http_server' running without any HTTP authentication checking
application01    | 2021-04-19 16:42:32,573 INFO supervisord started with pid 8
application01    | 2021-04-19 16:42:33,578 INFO spawned: 'application' with pid 10
application01    | 2021-04-19 16:42:33,583 INFO spawned: 'consul' with pid 11
application01    | 2021-04-19 16:42:33,587 INFO spawned: 'dnsmasq' with pid 12
application01    | 2021-04-19 16:42:34,114 INFO Starting.
application01    | 2021-04-19 16:42:34,131 INFO Connecting.
application01    | 2021-04-19 16:42:34,333 ERROR could not translate host name "master.pglab.service.consul" to address: Name or service not known
application01    |
application01    | 2021-04-19 16:42:35,336 INFO success: application entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
application01    | 2021-04-19 16:42:35,336 INFO success: consul entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
application01    | 2021-04-19 16:42:35,336 INFO success: dnsmasq entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
application01    | 2021-04-19 16:42:36,337 INFO Connecting.
application01    | 2021-04-19 16:42:38,768 ERROR could not translate host name "master.pglab.service.consul" to address: Name or service not known
application01    |
application01    | 2021-04-19 16:42:40,771 INFO Connecting.
application01    | 2021-04-19 16:42:40,775 ERROR could not translate host name "master.pglab.service.consul" to address: Name or service not known
application01    |
application01    | 2021-04-19 16:42:42,778 INFO Connecting.
application01    | 2021-04-19 16:42:42,801 INFO I'm connected to 172.24.0.5
application01    | 2021-04-19 16:42:43,804 INFO I'm connected to 172.24.0.5
application01    | 2021-04-19 16:42:44,805 INFO I'm connected to 172.24.0.5
...
```
