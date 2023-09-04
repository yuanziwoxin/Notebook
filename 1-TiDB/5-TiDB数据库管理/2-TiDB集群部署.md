# TiDB集群拓扑结构

## 集群节点规划

```
# pd
bigdata01 192.168.93.128:2379
bigdata02 192.168.93.129:2379
bigdata03 192.168.93.130:2379

# tikv
bigdata01 192.168.93.128:20160
bigdata02 192.168.93.129:20160
bigdata03 192.168.93.130:20160

# tidb
bigdata01 192.168.93.128:4000

# tiflash
bigdata02 192.168.93.129:9000
bigdata03 192.168.93.130:9000

# ticdc
bigdata01 192.168.93.128:8300

# prometheus
bigdata01 192.168.93.128:9090

# grafana
bigdata01 192.168.93.128:3000

# Dashboard
http://192.168.93.128:2379/dashboard
```





# 在线安装TiUP

```shell
[tidb@BigData01 root]$ curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 7322k  100 7322k    0     0  7291k      0  0:00:01  0:00:01 --:--:-- 7293k
WARN: adding root certificate via internet: https://tiup-mirrors.pingcap.com/root.json
You can revoke this by remove /home/tidb/.tiup/bin/7b8e153f2e2d0928.root.json
Successfully set mirror to https://tiup-mirrors.pingcap.com
Detected shell: bash
Shell profile:  /home/tidb/.bash_profile
Installed path: /home/tidb/.tiup/bin/tiup
===============================================
Have a try:     tiup playground
===============================================
[tidb@BigData01 root]$ cat /home/tidb/.bash_profile
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
	. ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/.local/bin:$HOME/bin

export PATH
[tidb@BigData01 ~]$ vi /home/tidb/.bash_profile
[tidb@BigData01 ~]$ source /home/tidb/.bash_profile
[tidb@BigData01 ~]$ cat /home/tidb/.bash_profile
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
	. ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/.local/bin:$HOME/bin

export PATH

export PATH=/home/tidb/.tiup/bin:$PATH
```



# 初始化集群拓扑文件

```shell
[tidb@BigData01 ~]$ tiup cluster template > topology.yaml
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster template
```

修改集群拓扑文件中相关的IP配置

```shell
[tidb@BigData01 ~]$  vi topology.yaml
[tidb@BigData01 ~]$  cat topology.yaml
# # Global variables are applied to all deployments and used as the default value of
# # the deployments if a specific deployment value is missing.
global:
  # # The user who runs the tidb cluster.
  user: "tidb"
  # # group is used to specify the group name the user belong to if it's not the same as user.
  # group: "tidb"
  # # SSH port of servers in the managed cluster.
  ssh_port: 22
  # # Storage directory for cluster deployment files, startup scripts, and configuration files.
  deploy_dir: "/tidb-deploy"
  # # TiDB Cluster data storage directory
  data_dir: "/tidb-data"
  # # Supported values: "amd64", "arm64" (default: "amd64")
  arch: "amd64"
  # # Resource Control is used to limit the resource of an instance.
  # # See: https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html
  # # Supports using instance-level `resource_control` to override global `resource_control`.
  # resource_control:
  #   # See: https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#MemoryLimit=bytes
  #   memory_limit: "2G"
  #   # See: https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#CPUQuota=
  #   # The percentage specifies how much CPU time the unit shall get at maximum, relative to the total CPU time available on one CPU. Use values > 100% for allotting CPU time on more than one CPU.
  #   # Example: CPUQuota=200% ensures that the executed processes will never get more than two CPU time.
  #   cpu_quota: "200%"
  #   # See: https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#IOReadBandwidthMax=device%20bytes
  #   io_read_bandwidth_max: "/dev/disk/by-path/pci-0000:00:1f.2-scsi-0:0:0:0 100M"
  #   io_write_bandwidth_max: "/dev/disk/by-path/pci-0000:00:1f.2-scsi-0:0:0:0 100M"

# # Monitored variables are applied to all the machines.
monitored:
  # # The communication port for reporting system information of each node in the TiDB cluster.
  node_exporter_port: 9100
  # # Blackbox_exporter communication port, used for TiDB cluster port monitoring.
  blackbox_exporter_port: 9115
  # # Storage directory for deployment files, startup scripts, and configuration files of monitoring components.
  # deploy_dir: "/tidb-deploy/monitored-9100"
  # # Data storage directory of monitoring components.
  # data_dir: "/tidb-data/monitored-9100"
  # # Log storage directory of the monitoring component.
  # log_dir: "/tidb-deploy/monitored-9100/log"

# # Server configs are used to specify the runtime configuration of TiDB components.
# # All configuration items can be found in TiDB docs:
# # - TiDB: https://pingcap.com/docs/stable/reference/configuration/tidb-server/configuration-file/
# # - TiKV: https://pingcap.com/docs/stable/reference/configuration/tikv-server/configuration-file/
# # - PD: https://pingcap.com/docs/stable/reference/configuration/pd-server/configuration-file/
# # - TiFlash: https://docs.pingcap.com/tidb/stable/tiflash-configuration
# #
# # All configuration items use points to represent the hierarchy, e.g:
# #   readpool.storage.use-unified-pool
# #           ^       ^
# # - example: https://github.com/pingcap/tiup/blob/master/examples/topology.example.yaml.
# # You can overwrite this configuration via the instance-level `config` field.
# server_configs:
  # tidb:
  # tikv:
  # pd:
  # tiflash:
  # tiflash-learner:

# # Server configs are used to specify the configuration of PD Servers.
pd_servers:
  # # The ip address of the PD Server.
  - host: 192.168.93.128
    # # SSH port of the server.
    # ssh_port: 22
    # # PD Server name
    # name: "pd-1"
    # # communication port for TiDB Servers to connect.
    # client_port: 2379
    # # Communication port among PD Server nodes.
    # peer_port: 2380
    # # PD Server deployment file, startup script, configuration file storage directory.
    # deploy_dir: "/tidb-deploy/pd-2379"
    # # PD Server data storage directory.
    # data_dir: "/tidb-data/pd-2379"
    # # PD Server log file storage directory.
    # log_dir: "/tidb-deploy/pd-2379/log"
    # # numa node bindings.
    # numa_node: "0,1"
    # # The following configs are used to overwrite the `server_configs.pd` values.
    # config:
    #   schedule.max-merge-region-size: 20
    #   schedule.max-merge-region-keys: 200000
  - host: 192.168.93.129
    # ssh_port: 22
    # name: "pd-1"
    # client_port: 2379
    # peer_port: 2380
    # deploy_dir: "/tidb-deploy/pd-2379"
    # data_dir: "/tidb-data/pd-2379"
    # log_dir: "/tidb-deploy/pd-2379/log"
    # numa_node: "0,1"
    # config:
    #   schedule.max-merge-region-size: 20
    #   schedule.max-merge-region-keys: 200000
  - host: 192.168.93.130
    # ssh_port: 22
    # name: "pd-1"
    # client_port: 2379
    # peer_port: 2380
    # deploy_dir: "/tidb-deploy/pd-2379"
    # data_dir: "/tidb-data/pd-2379"
    # log_dir: "/tidb-deploy/pd-2379/log"
    # numa_node: "0,1"
    # config:
    #   schedule.max-merge-region-size: 20
    #   schedule.max-merge-region-keys: 200000

# # Server configs are used to specify the configuration of TiDB Servers.
tidb_servers:
  # # The ip address of the TiDB Server.
  - host: 192.168.93.128
    # # SSH port of the server.
    # ssh_port: 22
    # # The port for clients to access the TiDB cluster.
    # port: 4000
    # # TiDB Server status API port.
    # status_port: 10080
    # # TiDB Server deployment file, startup script, configuration file storage directory.
    # deploy_dir: "/tidb-deploy/tidb-4000"
    # # TiDB Server log file storage directory.
    # log_dir: "/tidb-deploy/tidb-4000/log"
  # # The ip address of the TiDB Server.

# # Server configs are used to specify the configuration of TiKV Servers.
tikv_servers:
  # # The ip address of the TiKV Server.
  - host: 192.168.93.128
    # # SSH port of the server.
    # ssh_port: 22
    # # TiKV Server communication port.
    # port: 20160
    # # TiKV Server status API port.
    # status_port: 20180
    # # TiKV Server deployment file, startup script, configuration file storage directory.
    # deploy_dir: "/tidb-deploy/tikv-20160"
    # # TiKV Server data storage directory.
    # data_dir: "/tidb-data/tikv-20160"
    # # TiKV Server log file storage directory.
    # log_dir: "/tidb-deploy/tikv-20160/log"
    # # The following configs are used to overwrite the `server_configs.tikv` values.
    # config:
    #   log.level: warn
  # # The ip address of the TiKV Server.
  - host: 192.168.93.129
    # ssh_port: 22
    # port: 20160
    # status_port: 20180
    # deploy_dir: "/tidb-deploy/tikv-20160"
    # data_dir: "/tidb-data/tikv-20160"
    # log_dir: "/tidb-deploy/tikv-20160/log"
    # config:
    #   log.level: warn
  - host: 192.168.93.130
    # ssh_port: 22
    # port: 20160
    # status_port: 20180
    # deploy_dir: "/tidb-deploy/tikv-20160"
    # data_dir: "/tidb-data/tikv-20160"
    # log_dir: "/tidb-deploy/tikv-20160/log"
    # config:
    #   log.level: warn

# # Server configs are used to specify the configuration of TiFlash Servers.
tiflash_servers:
  # # The ip address of the TiFlash Server.
  - host: 192.168.93.129
    # # SSH port of the server.
    # ssh_port: 22
    # # TiFlash TCP Service port.
    # tcp_port: 9000
    # # TiFlash raft service and coprocessor service listening address.
    # flash_service_port: 3930
    # # TiFlash Proxy service port.
    # flash_proxy_port: 20170
    # # TiFlash Proxy metrics port.
    # flash_proxy_status_port: 20292
    # # TiFlash metrics port.
    # metrics_port: 8234
    # # TiFlash Server deployment file, startup script, configuration file storage directory.
    # deploy_dir: /tidb-deploy/tiflash-9000
    ## With cluster version >= v4.0.9 and you want to deploy a multi-disk TiFlash node, it is recommended to
    ## check config.storage.* for details. The data_dir will be ignored if you defined those configurations.
    ## Setting data_dir to a ','-joined string is still supported but deprecated.
    ## Check https://docs.pingcap.com/tidb/stable/tiflash-configuration#multi-disk-deployment for more details.
    # # TiFlash Server data storage directory.
    # data_dir: /tidb-data/tiflash-9000
    # # TiFlash Server log file storage directory.
    # log_dir: /tidb-deploy/tiflash-9000/log
  # # The ip address of the TiKV Server.
  - host: 192.168.93.130
    # ssh_port: 22
    # tcp_port: 9000
    # flash_service_port: 3930
    # flash_proxy_port: 20170
    # flash_proxy_status_port: 20292
    # metrics_port: 8234
    # deploy_dir: /tidb-deploy/tiflash-9000
    # data_dir: /tidb-data/tiflash-9000
    # log_dir: /tidb-deploy/tiflash-9000/log

# # Server configs are used to specify the configuration of Prometheus Server.  
monitoring_servers:
  # # The ip address of the Monitoring Server.
  - host: 192.168.93.128
    # # SSH port of the server.
    # ssh_port: 22
    # # Prometheus Service communication port.
    # port: 9090
    # # ng-monitoring servive communication port
    # ng_port: 12020
    # # Prometheus deployment file, startup script, configuration file storage directory.
    # deploy_dir: "/tidb-deploy/prometheus-8249"
    # # Prometheus data storage directory.
    # data_dir: "/tidb-data/prometheus-8249"
    # # Prometheus log file storage directory.
    # log_dir: "/tidb-deploy/prometheus-8249/log"

# # Server configs are used to specify the configuration of Grafana Servers.  
grafana_servers:
  # # The ip address of the Grafana Server.
  - host: 192.168.93.128
    # # Grafana web port (browser access)
    # port: 3000
    # # Grafana deployment file, startup script, configuration file storage directory.
    # deploy_dir: /tidb-deploy/grafana-3000

# # Server configs are used to specify the configuration of Alertmanager Servers.  
alertmanager_servers:
  # # The ip address of the Alertmanager Server.
  - host: 192.168.93.128
    # # SSH port of the server.
    # ssh_port: 22
    # # Alertmanager web service port.
    # web_port: 9093
    # # Alertmanager communication port.
    # cluster_port: 9094
    # # Alertmanager deployment file, startup script, configuration file storage directory.
    # deploy_dir: "/tidb-deploy/alertmanager-9093"
    # # Alertmanager data storage directory.
    # data_dir: "/tidb-data/alertmanager-9093"
    # # Alertmanager log file storage directory.
    # log_dir: "/tidb-deploy/alertmanager-9093"
cdc_servers:
  - host: 192.168.93.128
```

# 执行部署命令

## 检查集群存在的潜在风险

```shell
[tidb@BigData01 ~]$ tiup cluster check ./topology.yaml --user root -p
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster check ./topology.yaml --user root -p
Input SSH password: 



+ Detect CPU Arch Name
  - Detecting node 192.168.93.128 Arch info ... Done
  - Detecting node 192.168.93.129 Arch info ... Done
  - Detecting node 192.168.93.130 Arch info ... Done



+ Detect CPU OS Name
  - Detecting node 192.168.93.128 OS info ... Done
  - Detecting node 192.168.93.129 OS info ... Done
  - Detecting node 192.168.93.130 OS info ... Done
+ Download necessary tools
  - Downloading check tools for linux/amd64 ... Done
+ Collect basic system information
+ Collect basic system information
+ Collect basic system information
+ Collect basic system information
  - Getting system info of 192.168.93.129:22 ... Done
  - Getting system info of 192.168.93.130:22 ... Done
  - Getting system info of 192.168.93.128:22 ... Done
+ Check time zone
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
+ Cleanup check files
  - Cleanup check files on 192.168.93.129:22 ... Done
  - Cleanup check files on 192.168.93.130:22 ... Done
  - Cleanup check files on 192.168.93.128:22 ... Done
Node            Check         Result  Message
----            -----         ------  -------
192.168.93.128  service       Fail    service irqbalance is not running
192.168.93.128  cpu-governor  Warn    Unable to determine current CPU frequency governor policy
192.168.93.128  swap          Warn    swap is enabled, please disable it for best performance
192.168.93.128  disk          Warn    mount point / does not have 'noatime' option set
192.168.93.128  limits        Fail    hard limit of 'nofile' for user 'tidb' is not set or too low
192.168.93.128  limits        Fail    soft limit of 'stack' for user 'tidb' is not set or too low
192.168.93.128  limits        Fail    soft limit of 'nofile' for user 'tidb' is not set or too low
192.168.93.128  sysctl        Fail    fs.file-max = 788432, should be greater than 1000000
192.168.93.128  sysctl        Fail    net.core.somaxconn = 128, should be greater than 32768
192.168.93.128  sysctl        Fail    net.ipv4.tcp_syncookies = 1, should be 0
192.168.93.128  sysctl        Fail    vm.swappiness = 70, should be 0
192.168.93.128  selinux       Fail    SELinux is not disabled
192.168.93.128  thp           Fail    THP is enabled, please disable it for best performance
192.168.93.128  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.128  cpu-cores     Pass    number of CPU cores / threads: 1
192.168.93.128  memory        Pass    memory size is 8192MB
192.168.93.128  network       Pass    network speed of ens33 is 1000MB
192.168.93.128  command       Fail    numactl not usable, bash: numactl: command not found
192.168.93.129  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.129  network       Pass    network speed of ens33 is 1000MB
192.168.93.129  thp           Fail    THP is enabled, please disable it for best performance
192.168.93.129  command       Fail    numactl not usable, bash: numactl: command not found
192.168.93.129  cpu-cores     Pass    number of CPU cores / threads: 1
192.168.93.129  sysctl        Fail    fs.file-max = 178519, should be greater than 1000000
192.168.93.129  sysctl        Fail    net.core.somaxconn = 128, should be greater than 32768
192.168.93.129  sysctl        Fail    net.ipv4.tcp_syncookies = 1, should be 0
192.168.93.129  sysctl        Fail    vm.swappiness = 30, should be 0
192.168.93.129  timezone      Pass    time zone is the same as the first PD machine: Asia/Shanghai
192.168.93.129  disk          Fail    multiple components tikv:/tidb-data/tikv-20160,tiflash:/tidb-data/tiflash-9000 are using the same partition 192.168.93.129:/ as data dir
192.168.93.129  limits        Fail    soft limit of 'nofile' for user 'tidb' is not set or too low
192.168.93.129  limits        Fail    hard limit of 'nofile' for user 'tidb' is not set or too low
192.168.93.129  limits        Fail    soft limit of 'stack' for user 'tidb' is not set or too low
192.168.93.129  cpu-governor  Warn    Unable to determine current CPU frequency governor policy
192.168.93.129  swap          Warn    swap is enabled, please disable it for best performance
192.168.93.129  memory        Pass    memory size is 2024MB
192.168.93.129  disk          Warn    mount point / does not have 'noatime' option set
192.168.93.129  selinux       Fail    SELinux is not disabled
192.168.93.129  service       Fail    service irqbalance is not running
192.168.93.130  timezone      Pass    time zone is the same as the first PD machine: Asia/Shanghai
192.168.93.130  memory        Pass    memory size is 2024MB
192.168.93.130  network       Pass    network speed of ens33 is 1000MB
192.168.93.130  service       Fail    service irqbalance is not running
192.168.93.130  cpu-cores     Pass    number of CPU cores / threads: 1
192.168.93.130  swap          Warn    swap is enabled, please disable it for best performance
192.168.93.130  selinux       Fail    SELinux is not disabled
192.168.93.130  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.130  disk          Warn    mount point / does not have 'noatime' option set
192.168.93.130  disk          Fail    multiple components tikv:/tidb-data/tikv-20160,tiflash:/tidb-data/tiflash-9000 are using the same partition 192.168.93.130:/ as data dir
192.168.93.130  limits        Fail    soft limit of 'nofile' for user 'tidb' is not set or too low
192.168.93.130  limits        Fail    hard limit of 'nofile' for user 'tidb' is not set or too low
192.168.93.130  limits        Fail    soft limit of 'stack' for user 'tidb' is not set or too low
192.168.93.130  command       Fail    numactl not usable, bash: numactl: command not found
192.168.93.130  cpu-governor  Warn    Unable to determine current CPU frequency governor policy
192.168.93.130  sysctl        Fail    fs.file-max = 178519, should be greater than 1000000
192.168.93.130  sysctl        Fail    net.core.somaxconn = 128, should be greater than 32768
192.168.93.130  sysctl        Fail    net.ipv4.tcp_syncookies = 1, should be 0
192.168.93.130  sysctl        Fail    vm.swappiness = 30, should be 0
192.168.93.130  thp           Fail    THP is enabled, please disable it for best performance
```

## 自动修复集群潜在的风险

```shell
[tidb@BigData01 ~]$ tiup cluster check ./topology.yaml --apply --user root -p
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster check ./topology.yaml --apply --user root -p
Input SSH password: 



+ Detect CPU Arch Name
  - Detecting node 192.168.93.128 Arch info ... Done
  - Detecting node 192.168.93.129 Arch info ... Done
  - Detecting node 192.168.93.130 Arch info ... Done



+ Detect CPU OS Name
  - Detecting node 192.168.93.128 OS info ... Done
  - Detecting node 192.168.93.129 OS info ... Done
  - Detecting node 192.168.93.130 OS info ... Done
+ Download necessary tools
  - Downloading check tools for linux/amd64 ... Done
+ Collect basic system information
+ Collect basic system information
  - Getting system info of 192.168.93.129:22 ... ⠇ CopyComponent: component=insight, version=, remote=192.168.93.129:/tmp/tiup os=linux, arch=amd64
+ Collect basic system information
  - Getting system info of 192.168.93.129:22 ... Done
  - Getting system info of 192.168.93.130:22 ... Done
  - Getting system info of 192.168.93.128:22 ... Done
+ Check time zone
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
+ Cleanup check files
  - Cleanup check files on 192.168.93.129:22 ... Done
  - Cleanup check files on 192.168.93.130:22 ... Done
  - Cleanup check files on 192.168.93.128:22 ... Done
Node            Check         Result  Message
----            -----         ------  -------
192.168.93.129  thp           Fail    will try to disable THP, please check again after reboot
192.168.93.129  service       Fail    will try to 'start irqbalance.service'
192.168.93.129  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.129  cpu-cores     Pass    number of CPU cores / threads: 1
192.168.93.129  swap          Warn    will try to disable swap, please also check /etc/fstab manually
192.168.93.129  memory        Pass    memory size is 2024MB
192.168.93.129  command       Fail    numactl not usable, bash: numactl: command not found, auto fixing not supported
192.168.93.129  cpu-governor  Warn    Unable to determine current CPU frequency governor policy, auto fixing not supported
192.168.93.129  disk          Warn    mount point / does not have 'noatime' option set, auto fixing not supported
192.168.93.129  disk          Fail    multiple components tikv:/tidb-data/tikv-20160,tiflash:/tidb-data/tiflash-9000 are using the same partition 192.168.93.129:/ as data dir, auto fixing not supported
192.168.93.129  selinux       Fail    will try to disable SELinux, reboot might be needed
192.168.93.129  timezone      Pass    time zone is the same as the first PD machine: Asia/Shanghai
192.168.93.129  sysctl        Fail    will try to set 'fs.file-max = 1000000'
192.168.93.129  sysctl        Fail    will try to set 'net.core.somaxconn = 32768'
192.168.93.129  sysctl        Fail    will try to set 'net.ipv4.tcp_syncookies = 0'
192.168.93.129  sysctl        Fail    will try to set 'vm.swappiness = 0'
192.168.93.129  network       Pass    network speed of ens33 is 1000MB
192.168.93.129  limits        Fail    will try to set 'tidb    soft    nofile    1000000'
192.168.93.129  limits        Fail    will try to set 'tidb    hard    nofile    1000000'
192.168.93.129  limits        Fail    will try to set 'tidb    soft    stack    10240'
192.168.93.130  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.130  disk          Warn    mount point / does not have 'noatime' option set, auto fixing not supported
192.168.93.130  sysctl        Fail    will try to set 'fs.file-max = 1000000'
192.168.93.130  sysctl        Fail    will try to set 'net.core.somaxconn = 32768'
192.168.93.130  sysctl        Fail    will try to set 'net.ipv4.tcp_syncookies = 0'
192.168.93.130  sysctl        Fail    will try to set 'vm.swappiness = 0'
192.168.93.130  swap          Warn    will try to disable swap, please also check /etc/fstab manually
192.168.93.130  memory        Pass    memory size is 2024MB
192.168.93.130  limits        Fail    will try to set 'tidb    soft    stack    10240'
192.168.93.130  limits        Fail    will try to set 'tidb    soft    nofile    1000000'
192.168.93.130  limits        Fail    will try to set 'tidb    hard    nofile    1000000'
192.168.93.130  selinux       Fail    will try to disable SELinux, reboot might be needed
192.168.93.130  thp           Fail    will try to disable THP, please check again after reboot
192.168.93.130  service       Fail    will try to 'start irqbalance.service'
192.168.93.130  timezone      Pass    time zone is the same as the first PD machine: Asia/Shanghai
192.168.93.130  cpu-cores     Pass    number of CPU cores / threads: 1
192.168.93.130  disk          Fail    multiple components tikv:/tidb-data/tikv-20160,tiflash:/tidb-data/tiflash-9000 are using the same partition 192.168.93.130:/ as data dir, auto fixing not supported
192.168.93.130  command       Fail    numactl not usable, bash: numactl: command not found, auto fixing not supported
192.168.93.130  cpu-governor  Warn    Unable to determine current CPU frequency governor policy, auto fixing not supported
192.168.93.130  network       Pass    network speed of ens33 is 1000MB
192.168.93.128  memory        Pass    memory size is 8192MB
192.168.93.128  disk          Warn    mount point / does not have 'noatime' option set, auto fixing not supported
192.168.93.128  limits        Fail    will try to set 'tidb    soft    nofile    1000000'
192.168.93.128  limits        Fail    will try to set 'tidb    hard    nofile    1000000'
192.168.93.128  limits        Fail    will try to set 'tidb    soft    stack    10240'
192.168.93.128  thp           Fail    will try to disable THP, please check again after reboot
192.168.93.128  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.128  cpu-cores     Pass    number of CPU cores / threads: 1
192.168.93.128  swap          Warn    will try to disable swap, please also check /etc/fstab manually
192.168.93.128  selinux       Fail    will try to disable SELinux, reboot might be needed
192.168.93.128  service       Fail    will try to 'start irqbalance.service'
192.168.93.128  command       Fail    numactl not usable, bash: numactl: command not found, auto fixing not supported
192.168.93.128  cpu-governor  Warn    Unable to determine current CPU frequency governor policy, auto fixing not supported
192.168.93.128  network       Pass    network speed of ens33 is 1000MB
192.168.93.128  sysctl        Fail    will try to set 'fs.file-max = 1000000'
192.168.93.128  sysctl        Fail    will try to set 'net.core.somaxconn = 32768'
192.168.93.128  sysctl        Fail    will try to set 'net.ipv4.tcp_syncookies = 0'
192.168.93.128  sysctl        Fail    will try to set 'vm.swappiness = 0'



+ Try to apply changes to fix failed checks
+ Try to apply changes to fix failed checks
+ Try to apply changes to fix failed checks
+ Try to apply changes to fix failed checks
+ Try to apply changes to fix failed checks
+ Try to apply changes to fix failed checks
+ Try to apply changes to fix failed checks
  - Applying changes on 192.168.93.129 ... Done
  - Applying changes on 192.168.93.130 ... Done
  - Applying changes on 192.168.93.128 ... Done
```

修复后，再次检查集群中潜在的风险

```shell
[tidb@BigData01 ~]$ tiup cluster check ./topology.yaml --user root -p
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster check ./topology.yaml --user root -p
Input SSH password: 



+ Detect CPU Arch Name
  - Detecting node 192.168.93.128 Arch info ... Done
  - Detecting node 192.168.93.129 Arch info ... Done
  - Detecting node 192.168.93.130 Arch info ... Done



+ Detect CPU OS Name
  - Detecting node 192.168.93.128 OS info ... Done
  - Detecting node 192.168.93.129 OS info ... Done
  - Detecting node 192.168.93.130 OS info ... Done
+ Download necessary tools
  - Downloading check tools for linux/amd64 ... Done
+ Collect basic system information
+ Collect basic system information
+ Collect basic system information
+ Collect basic system information
  - Getting system info of 192.168.93.129:22 ... Done
  - Getting system info of 192.168.93.130:22 ... Done
  - Getting system info of 192.168.93.128:22 ... Done
+ Check time zone
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
+ Cleanup check files
  - Cleanup check files on 192.168.93.129:22 ... Done
  - Cleanup check files on 192.168.93.130:22 ... Done
  - Cleanup check files on 192.168.93.128:22 ... Done
Node            Check         Result  Message
----            -----         ------  -------
192.168.93.128  cpu-cores     Pass    number of CPU cores / threads: 1
192.168.93.128  service       Fail    service irqbalance is not running
192.168.93.128  command       Fail    numactl not usable, bash: numactl: command not found
192.168.93.128  network       Pass    network speed of ens33 is 1000MB
192.168.93.128  disk          Warn    mount point / does not have 'noatime' option set
192.168.93.128  selinux       Pass    SELinux is disabled
192.168.93.128  thp           Pass    THP is disabled
192.168.93.128  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.128  cpu-governor  Warn    Unable to determine current CPU frequency governor policy
192.168.93.128  memory        Pass    memory size is 8192MB
192.168.93.129  disk          Warn    mount point / does not have 'noatime' option set
192.168.93.129  disk          Fail    multiple components tikv:/tidb-data/tikv-20160,tiflash:/tidb-data/tiflash-9000 are using the same partition 192.168.93.129:/ as data dir
192.168.93.129  selinux       Pass    SELinux is disabled
192.168.93.129  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.129  cpu-cores     Pass    number of CPU cores / threads: 1
192.168.93.129  cpu-governor  Warn    Unable to determine current CPU frequency governor policy
192.168.93.129  memory        Pass    memory size is 2024MB
192.168.93.129  network       Pass    network speed of ens33 is 1000MB
192.168.93.129  thp           Pass    THP is disabled
192.168.93.129  service       Fail    service irqbalance is not running
192.168.93.129  command       Fail    numactl not usable, bash: numactl: command not found
192.168.93.129  timezone      Pass    time zone is the same as the first PD machine: Asia/Shanghai
192.168.93.130  disk          Warn    mount point / does not have 'noatime' option set
192.168.93.130  disk          Fail    multiple components tikv:/tidb-data/tikv-20160,tiflash:/tidb-data/tiflash-9000 are using the same partition 192.168.93.130:/ as data dir
192.168.93.130  selinux       Pass    SELinux is disabled
192.168.93.130  thp           Pass    THP is disabled
192.168.93.130  service       Fail    service irqbalance is not running
192.168.93.130  timezone      Pass    time zone is the same as the first PD machine: Asia/Shanghai
192.168.93.130  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.130  cpu-cores     Pass    number of CPU cores / threads: 1
192.168.93.130  cpu-governor  Warn    Unable to determine current CPU frequency governor policy
192.168.93.130  memory        Pass    memory size is 2024MB
192.168.93.130  network       Pass    network speed of ens33 is 1000MB
192.168.93.130  command       Fail    numactl not usable, bash: numactl: command not found
```



## 安装numactl

安装numactl, 修复numactl not usable, bash: numactl: command not found 的问题

```shell
[tidb@BigData01 ~]$ su root
Password: 
[root@BigData01 tidb]# yum install numactl
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.ustc.edu.cn
 * centos-sclo-rh: mirrors.tuna.tsinghua.edu.cn
 * centos-sclo-sclo: mirrors.tuna.tsinghua.edu.cn
 * extras: mirrors.tuna.tsinghua.edu.cn
 * updates: mirrors.aliyun.com
base                                                                                                                                                                                                                                                                | 3.6 kB  00:00:00     
centos-sclo-rh                                                                                                                                                                                                                                                      | 3.0 kB  00:00:00     
centos-sclo-sclo                                                                                                                                                                                                                                                    | 3.0 kB  00:00:00     
extras                                                                                                                                                                                                                                                              | 2.9 kB  00:00:00     
updates                                                                                                                                                                                                                                                             | 2.9 kB  00:00:00     
Resolving Dependencies
--> Running transaction check
---> Package numactl.x86_64 0:2.0.12-5.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

===========================================================================================================================================================================================================================================================================================
 Package                                                             Arch                                                               Version                                                                     Repository                                                        Size
===========================================================================================================================================================================================================================================================================================
Installing:
 numactl                                                             x86_64                                                             2.0.12-5.el7                                                                base                                                              66 k

Transaction Summary
===========================================================================================================================================================================================================================================================================================
Install  1 Package

Total download size: 66 k
Installed size: 141 k
Is this ok [y/d/N]: y
Downloading packages:
numactl-2.0.12-5.el7.x86_64.rpm                                                                                                                                                                                                                                     |  66 kB  00:00:00     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : numactl-2.0.12-5.el7.x86_64                                                                                                                                                                                                                                             1/1 
  Verifying  : numactl-2.0.12-5.el7.x86_64                                                                                                                                                                                                                                             1/1 

Installed:
  numactl.x86_64 0:2.0.12-5.el7                                                                                                                                                                                                                                                            

Complete!
```

## 部署TiDB集群

```shell
[tidb@BigData01 ~]$ tiup cluster deploy tidb-test v6.1.6 ./topology.yaml --user root -p
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster deploy tidb-test v6.1.6 ./topology.yaml --user root -p
Input SSH password: 



+ Detect CPU Arch Name
  - Detecting node 192.168.93.128 Arch info ... Done
  - Detecting node 192.168.93.129 Arch info ... Done
  - Detecting node 192.168.93.130 Arch info ... Done



+ Detect CPU OS Name
  - Detecting node 192.168.93.128 OS info ... Done
  - Detecting node 192.168.93.129 OS info ... Done
  - Detecting node 192.168.93.130 OS info ... Done
Please confirm your topology:
Cluster type:    tidb
Cluster name:    tidb-test
Cluster version: v6.1.6
Role          Host            Ports                            OS/Arch       Directories
----          ----            -----                            -------       -----------
pd            192.168.93.128  2379/2380                        linux/x86_64  /tidb-deploy/pd-2379,/tidb-data/pd-2379
pd            192.168.93.129  2379/2380                        linux/x86_64  /tidb-deploy/pd-2379,/tidb-data/pd-2379
pd            192.168.93.130  2379/2380                        linux/x86_64  /tidb-deploy/pd-2379,/tidb-data/pd-2379
tikv          192.168.93.128  20160/20180                      linux/x86_64  /tidb-deploy/tikv-20160,/tidb-data/tikv-20160
tikv          192.168.93.129  20160/20180                      linux/x86_64  /tidb-deploy/tikv-20160,/tidb-data/tikv-20160
tikv          192.168.93.130  20160/20180                      linux/x86_64  /tidb-deploy/tikv-20160,/tidb-data/tikv-20160
tidb          192.168.93.128  4000/10080                       linux/x86_64  /tidb-deploy/tidb-4000
tiflash       192.168.93.129  9000/8123/3930/20170/20292/8234  linux/x86_64  /tidb-deploy/tiflash-9000,/tidb-data/tiflash-9000
tiflash       192.168.93.130  9000/8123/3930/20170/20292/8234  linux/x86_64  /tidb-deploy/tiflash-9000,/tidb-data/tiflash-9000
cdc           192.168.93.128  8300                             linux/x86_64  /tidb-deploy/cdc-8300,/tidb-data/cdc-8300
prometheus    192.168.93.128  9090/12020                       linux/x86_64  /tidb-deploy/prometheus-9090,/tidb-data/prometheus-9090
grafana       192.168.93.128  3000                             linux/x86_64  /tidb-deploy/grafana-3000
alertmanager  192.168.93.128  9093/9094                        linux/x86_64  /tidb-deploy/alertmanager-9093,/tidb-data/alertmanager-9093
Attention:
    1. If the topology is not what you expected, check your yaml file.
    2. Please confirm there is no port/directory conflicts in same host.
Do you want to continue? [y/N]: (default=N) y
+ Generate SSH keys ... Done
+ Download TiDB components
  - Download pd:v6.1.6 (linux/amd64) ... Done
  - Download tikv:v6.1.6 (linux/amd64) ... Done
  - Download tidb:v6.1.6 (linux/amd64) ... Done
  - Download tiflash:v6.1.6 (linux/amd64) ... Done
  - Download cdc:v6.1.6 (linux/amd64) ... Done
  - Download prometheus:v6.1.6 (linux/amd64) ... Done
  - Download grafana:v6.1.6 (linux/amd64) ... Done
  - Download alertmanager: (linux/amd64) ... Done
  - Download node_exporter: (linux/amd64) ... Done
  - Download blackbox_exporter: (linux/amd64) ... Done
+ Initialize target host environments
  - Prepare 192.168.93.128:22 ... Done
  - Prepare 192.168.93.129:22 ... Done
  - Prepare 192.168.93.130:22 ... Done
+ Deploy TiDB instance
  - Copy pd -> 192.168.93.128 ... Done
  - Copy pd -> 192.168.93.129 ... Done
  - Copy pd -> 192.168.93.130 ... Done
  - Copy tikv -> 192.168.93.128 ... Done
  - Copy tikv -> 192.168.93.129 ... Done
  - Copy tikv -> 192.168.93.130 ... Done
  - Copy tidb -> 192.168.93.128 ... Done
  - Copy tiflash -> 192.168.93.129 ... Done
  - Copy tiflash -> 192.168.93.130 ... Done
  - Copy cdc -> 192.168.93.128 ... Done
  - Copy prometheus -> 192.168.93.128 ... Done
  - Copy grafana -> 192.168.93.128 ... Done
  - Copy alertmanager -> 192.168.93.128 ... Done
  - Deploy node_exporter -> 192.168.93.128 ... Done
  - Deploy node_exporter -> 192.168.93.129 ... Done
  - Deploy node_exporter -> 192.168.93.130 ... Done
  - Deploy blackbox_exporter -> 192.168.93.128 ... Done
  - Deploy blackbox_exporter -> 192.168.93.129 ... Done
  - Deploy blackbox_exporter -> 192.168.93.130 ... Done
+ Copy certificate to remote host
+ Init instance configs
  - Generate config pd -> 192.168.93.128:2379 ... Done
  - Generate config pd -> 192.168.93.129:2379 ... Done
  - Generate config pd -> 192.168.93.130:2379 ... Done
  - Generate config tikv -> 192.168.93.128:20160 ... Done
  - Generate config tikv -> 192.168.93.129:20160 ... Done
  - Generate config tikv -> 192.168.93.130:20160 ... Done
  - Generate config tidb -> 192.168.93.128:4000 ... Done
  - Generate config tiflash -> 192.168.93.129:9000 ... Done
  - Generate config tiflash -> 192.168.93.130:9000 ... Done
  - Generate config cdc -> 192.168.93.128:8300 ... Done
  - Generate config prometheus -> 192.168.93.128:9090 ... Done
  - Generate config grafana -> 192.168.93.128:3000 ... Done
  - Generate config alertmanager -> 192.168.93.128:9093 ... Done
+ Init monitor configs
  - Generate config node_exporter -> 192.168.93.128 ... Done
  - Generate config node_exporter -> 192.168.93.129 ... Done
  - Generate config node_exporter -> 192.168.93.130 ... Done
  - Generate config blackbox_exporter -> 192.168.93.128 ... Done
  - Generate config blackbox_exporter -> 192.168.93.129 ... Done
  - Generate config blackbox_exporter -> 192.168.93.130 ... Done
Enabling component pd
	Enabling instance 192.168.93.130:2379
	Enabling instance 192.168.93.128:2379
	Enabling instance 192.168.93.129:2379
	Enable instance 192.168.93.129:2379 success
	Enable instance 192.168.93.130:2379 success
	Enable instance 192.168.93.128:2379 success
Enabling component tikv
	Enabling instance 192.168.93.130:20160
	Enabling instance 192.168.93.128:20160
	Enabling instance 192.168.93.129:20160
	Enable instance 192.168.93.130:20160 success
	Enable instance 192.168.93.129:20160 success
	Enable instance 192.168.93.128:20160 success
Enabling component tidb
	Enabling instance 192.168.93.128:4000
	Enable instance 192.168.93.128:4000 success
Enabling component tiflash
	Enabling instance 192.168.93.130:9000
	Enabling instance 192.168.93.129:9000
	Enable instance 192.168.93.130:9000 success
	Enable instance 192.168.93.129:9000 success
Enabling component cdc
	Enabling instance 192.168.93.128:8300
	Enable instance 192.168.93.128:8300 success
Enabling component prometheus
	Enabling instance 192.168.93.128:9090
	Enable instance 192.168.93.128:9090 success
Enabling component grafana
	Enabling instance 192.168.93.128:3000
	Enable instance 192.168.93.128:3000 success
Enabling component alertmanager
	Enabling instance 192.168.93.128:9093
	Enable instance 192.168.93.128:9093 success
Enabling component node_exporter
	Enabling instance 192.168.93.129
	Enabling instance 192.168.93.130
	Enabling instance 192.168.93.128
	Enable 192.168.93.130 success
	Enable 192.168.93.129 success
	Enable 192.168.93.128 success
Enabling component blackbox_exporter
	Enabling instance 192.168.93.129
	Enabling instance 192.168.93.130
	Enabling instance 192.168.93.128
	Enable 192.168.93.129 success
	Enable 192.168.93.130 success
	Enable 192.168.93.128 success
Cluster `tidb-test` deployed successfully, you can start it with command: `tiup cluster start tidb-test --init`
```

# 查看TiUP管理的集群情况

```shell
[tidb@BigData01 ~]$ tiup cluster list
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster list
Name       User  Version  Path                                                 PrivateKey
----       ----  -------  ----                                                 ----------
tidb-test  tidb  v6.1.6   /home/tidb/.tiup/storage/cluster/clusters/tidb-test  /home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa
```

# 检查部署的TiDB集群情况

```shell
[tidb@BigData01 ~]$ tiup cluster display tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v6.1.6
Deploy user:        tidb
SSH type:           builtin
Grafana URL:        http://192.168.93.128:3000
ID                    Role          Host            Ports                            OS/Arch       Status  Data Dir                      Deploy Dir
--                    ----          ----            -----                            -------       ------  --------                      ----------
192.168.93.128:9093   alertmanager  192.168.93.128  9093/9094                        linux/x86_64  Down    /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
192.168.93.128:8300   cdc           192.168.93.128  8300                             linux/x86_64  Down    /tidb-data/cdc-8300           /tidb-deploy/cdc-8300
192.168.93.128:3000   grafana       192.168.93.128  3000                             linux/x86_64  Down    -                             /tidb-deploy/grafana-3000
192.168.93.128:2379   pd            192.168.93.128  2379/2380                        linux/x86_64  Down    /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.129:2379   pd            192.168.93.129  2379/2380                        linux/x86_64  Down    /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.130:2379   pd            192.168.93.130  2379/2380                        linux/x86_64  Down    /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.128:9090   prometheus    192.168.93.128  9090/12020                       linux/x86_64  Down    /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
192.168.93.128:4000   tidb          192.168.93.128  4000/10080                       linux/x86_64  Down    -                             /tidb-deploy/tidb-4000
192.168.93.129:9000   tiflash       192.168.93.129  9000/8123/3930/20170/20292/8234  linux/x86_64  N/A     /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.130:9000   tiflash       192.168.93.130  9000/8123/3930/20170/20292/8234  linux/x86_64  N/A     /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.128:20160  tikv          192.168.93.128  20160/20180                      linux/x86_64  N/A     /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.129:20160  tikv          192.168.93.129  20160/20180                      linux/x86_64  N/A     /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.130:20160  tikv          192.168.93.130  20160/20180                      linux/x86_64  N/A     /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 13
```

# 启动TiDB集群

```shell
[tidb@BigData01 ~]$ tiup cluster start tidb-test --init
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster start tidb-test --init
Starting cluster tidb-test...
+ [ Serial ] - SSHKeySet: privateKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa, publicKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa.pub
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [ Serial ] - StartCluster
Starting component pd
	Starting instance 192.168.93.130:2379
	Starting instance 192.168.93.128:2379
	Starting instance 192.168.93.129:2379
	Start instance 192.168.93.129:2379 success
	Start instance 192.168.93.128:2379 success
	Start instance 192.168.93.130:2379 success
Starting component tikv
	Starting instance 192.168.93.130:20160
	Starting instance 192.168.93.128:20160
	Starting instance 192.168.93.129:20160
	Start instance 192.168.93.128:20160 success
	Start instance 192.168.93.130:20160 success
	Start instance 192.168.93.129:20160 success
Starting component tidb
	Starting instance 192.168.93.128:4000
	Start instance 192.168.93.128:4000 success
Starting component tiflash
	Starting instance 192.168.93.129:9000
	Starting instance 192.168.93.130:9000
	Start instance 192.168.93.129:9000 success
	Start instance 192.168.93.130:9000 success
Starting component cdc
	Starting instance 192.168.93.128:8300
	Start instance 192.168.93.128:8300 success
Starting component prometheus
	Starting instance 192.168.93.128:9090
	Start instance 192.168.93.128:9090 success
Starting component grafana
	Starting instance 192.168.93.128:3000
	Start instance 192.168.93.128:3000 success
Starting component alertmanager
	Starting instance 192.168.93.128:9093
	Start instance 192.168.93.128:9093 success
Starting component node_exporter
	Starting instance 192.168.93.130
	Starting instance 192.168.93.128
	Starting instance 192.168.93.129
	Start 192.168.93.130 success
	Start 192.168.93.129 success
	Start 192.168.93.128 success
Starting component blackbox_exporter
	Starting instance 192.168.93.130
	Starting instance 192.168.93.128
	Starting instance 192.168.93.129
	Start 192.168.93.130 success
	Start 192.168.93.129 success
	Start 192.168.93.128 success
+ [ Serial ] - UpdateTopology: cluster=tidb-test
Started cluster `tidb-test` successfully
The root password of TiDB database has been changed.
The new password is: 'i1RAkV8r*_@^9e67w4'.
Copy and record it to somewhere safe, it is only displayed once, and will not be stored.
The generated password can NOT be get and shown again.
```

# 检查TiDB集群状态

```shell
[tidb@BigData01 ~]$ tiup cluster display tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v6.1.6
Deploy user:        tidb
SSH type:           builtin
Dashboard URL:      http://192.168.93.128:2379/dashboard
Grafana URL:        http://192.168.93.128:3000
ID                    Role          Host            Ports                            OS/Arch       Status  Data Dir                      Deploy Dir
--                    ----          ----            -----                            -------       ------  --------                      ----------
192.168.93.128:9093   alertmanager  192.168.93.128  9093/9094                        linux/x86_64  Up      /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
192.168.93.128:8300   cdc           192.168.93.128  8300                             linux/x86_64  Up      /tidb-data/cdc-8300           /tidb-deploy/cdc-8300
192.168.93.128:3000   grafana       192.168.93.128  3000                             linux/x86_64  Up      -                             /tidb-deploy/grafana-3000
192.168.93.128:2379   pd            192.168.93.128  2379/2380                        linux/x86_64  Up|UI   /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.129:2379   pd            192.168.93.129  2379/2380                        linux/x86_64  Up      /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.130:2379   pd            192.168.93.130  2379/2380                        linux/x86_64  Up|L    /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.128:9090   prometheus    192.168.93.128  9090/12020                       linux/x86_64  Up      /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
192.168.93.128:4000   tidb          192.168.93.128  4000/10080                       linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
192.168.93.129:9000   tiflash       192.168.93.129  9000/8123/3930/20170/20292/8234  linux/x86_64  Up      /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.130:9000   tiflash       192.168.93.130  9000/8123/3930/20170/20292/8234  linux/x86_64  Up      /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.128:20160  tikv          192.168.93.128  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.129:20160  tikv          192.168.93.129  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.130:20160  tikv          192.168.93.130  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 13
```

# 连接数据库、新增用户、修改密码

```shell
[tidb@BigData01 ~]$ mysql -u root -h 192.168.93.128 -P 4000 -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 407
Server version: 5.7.25-TiDB-v6.1.6 TiDB Server (Apache License 2.0) Community Edition, MySQL 5.7 compatible

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> create user 'tidb'@'%' identified by 'xxx';
Query OK, 0 rows affected (0.10 sec)

MySQL [(none)]> set password for 'root'@'%'=password('xxx');
Query OK, 0 rows affected (0.10 sec)
```

# 停止TiDB集群

```shell
[tidb@BigData01 ~]$ tiup cluster stop tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster stop tidb-test
Will stop the cluster tidb-test with nodes: , roles: .
Do you want to continue? [y/N]:(default=N) y
+ [ Serial ] - SSHKeySet: privateKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa, publicKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa.pub
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [ Serial ] - StopCluster
Stopping component alertmanager
	Stopping instance 192.168.93.128
	Stop alertmanager 192.168.93.128:9093 success
Stopping component grafana
	Stopping instance 192.168.93.128
	Stop grafana 192.168.93.128:3000 success
Stopping component prometheus
	Stopping instance 192.168.93.128
	Stop prometheus 192.168.93.128:9090 success
Stopping component cdc
	Stopping instance 192.168.93.128
	Stop cdc 192.168.93.128:8300 success
Stopping component tiflash
	Stopping instance 192.168.93.130
	Stopping instance 192.168.93.129
	Stop tiflash 192.168.93.129:9000 success
	Stop tiflash 192.168.93.130:9000 success
Stopping component tidb
	Stopping instance 192.168.93.128
	Stop tidb 192.168.93.128:4000 success
Stopping component tikv
	Stopping instance 192.168.93.130
	Stopping instance 192.168.93.128
	Stopping instance 192.168.93.129
	Stop tikv 192.168.93.130:20160 success
	Stop tikv 192.168.93.129:20160 success
	Stop tikv 192.168.93.128:20160 success
Stopping component pd
	Stopping instance 192.168.93.130
	Stopping instance 192.168.93.128
	Stopping instance 192.168.93.129
	Stop pd 192.168.93.129:2379 success
	Stop pd 192.168.93.130:2379 success
	Stop pd 192.168.93.128:2379 success
Stopping component node_exporter
	Stopping instance 192.168.93.130
	Stopping instance 192.168.93.128
	Stopping instance 192.168.93.129
	Stop 192.168.93.129 success
	Stop 192.168.93.130 success
	Stop 192.168.93.128 success
Stopping component blackbox_exporter
	Stopping instance 192.168.93.130
	Stopping instance 192.168.93.128
	Stopping instance 192.168.93.129
	Stop 192.168.93.129 success
	Stop 192.168.93.130 success
	Stop 192.168.93.128 success
Stopped cluster `tidb-test` successfully
```

# TiDB集群管理

扩缩容之前，集群情况

```shell
[tidb@BigData01 ~]$ tiup cluster display tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v6.1.6
Deploy user:        tidb
SSH type:           builtin
Dashboard URL:      http://192.168.93.128:2379/dashboard
Grafana URL:        http://192.168.93.128:3000
ID                    Role          Host            Ports                            OS/Arch       Status  Data Dir                      Deploy Dir
--                    ----          ----            -----                            -------       ------  --------                      ----------
192.168.93.128:9093   alertmanager  192.168.93.128  9093/9094                        linux/x86_64  Up      /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
192.168.93.128:8300   cdc           192.168.93.128  8300                             linux/x86_64  Up      /tidb-data/cdc-8300           /tidb-deploy/cdc-8300
192.168.93.128:3000   grafana       192.168.93.128  3000                             linux/x86_64  Up      -                             /tidb-deploy/grafana-3000
192.168.93.128:2379   pd            192.168.93.128  2379/2380                        linux/x86_64  Up|UI   /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.129:2379   pd            192.168.93.129  2379/2380                        linux/x86_64  Up|L    /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.130:2379   pd            192.168.93.130  2379/2380                        linux/x86_64  Up      /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.128:9090   prometheus    192.168.93.128  9090/12020                       linux/x86_64  Up      /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
192.168.93.128:4000   tidb          192.168.93.128  4000/10080                       linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
192.168.93.129:9000   tiflash       192.168.93.129  9000/8123/3930/20170/20292/8234  linux/x86_64  Up      /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.130:9000   tiflash       192.168.93.130  9000/8123/3930/20170/20292/8234  linux/x86_64  Up      /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.128:20160  tikv          192.168.93.128  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.129:20160  tikv          192.168.93.129  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.130:20160  tikv          192.168.93.130  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 13
```



## 缩容TiKV节点

缩容一个TiKV节点：192.168.93.128:20160

> TiDB、TiKV和PD节点的缩容方式类似

```shell
[tidb@BigData01 ~]$ tiup cluster scale-in tidb-test --node 192.168.93.128:20160
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster scale-in tidb-test --node 192.168.93.128:20160
This operation will delete the 192.168.93.128:20160 nodes in `tidb-test` and all their data.
Do you want to continue? [y/N]:(default=N) y
The component `[tikv]` will become tombstone, maybe exists in several minutes or hours, after that you can use the prune command to clean it
Do you want to continue? [y/N]:(default=N) y
Scale-in nodes...
+ [ Serial ] - SSHKeySet: privateKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa, publicKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa.pub
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [ Serial ] - ClusterOperate: operation=DestroyOperation, options={Roles:[] Nodes:[192.168.93.128:20160] Force:false SSHTimeout:5 OptTimeout:120 APITimeout:600 IgnoreConfigCheck:false NativeSSH:false SSHType: Concurrency:5 SSHProxyHost: SSHProxyPort:22 SSHProxyUser:tidb SSHProxyIdentity:/home/tidb/.ssh/id_rsa SSHProxyUsePassword:false SSHProxyTimeout:5 SSHCustomScripts:{BeforeRestartInstance:{Raw:} AfterRestartInstance:{Raw:}} CleanupData:false CleanupLog:false CleanupAuditLog:false RetainDataRoles:[] RetainDataNodes:[] DisplayMode:default Operation:StartOperation}
TiKV instance number 2 will be less than max-replicas setting after scale-in. TiFlash won't be able to receive data from leader before TiKV instance number reach 3
The component `tikv` will become tombstone, maybe exists in several minutes or hours, after that you can use the prune command to clean it
+ [ Serial ] - UpdateMeta: cluster=tidb-test, deleted=`''`
+ [ Serial ] - UpdateTopology: cluster=tidb-test
+ Refresh instance configs
  - Generate config pd -> 192.168.93.128:2379 ... Done
  - Generate config pd -> 192.168.93.129:2379 ... Done
  - Generate config pd -> 192.168.93.130:2379 ... Done
  - Generate config tikv -> 192.168.93.129:20160 ... Done
  - Generate config tikv -> 192.168.93.130:20160 ... Done
  - Generate config tidb -> 192.168.93.128:4000 ... Done
  - Generate config tiflash -> 192.168.93.129:9000 ... Done
  - Generate config tiflash -> 192.168.93.130:9000 ... Done
  - Generate config cdc -> 192.168.93.128:8300 ... Done
  - Generate config prometheus -> 192.168.93.128:9090 ... Done
  - Generate config grafana -> 192.168.93.128:3000 ... Done
  - Generate config alertmanager -> 192.168.93.128:9093 ... Done
+ Reload prometheus and grafana
  - Reload prometheus -> 192.168.93.128:9090 ... Done
  - Reload grafana -> 192.168.93.128:3000 ... Done
Scaled cluster `tidb-test` in successfully
You have mail in /var/spool/mail/root
[tidb@BigData01 ~]$ 
[tidb@BigData01 ~]$ 
[tidb@BigData01 ~]$ tiup cluster display tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v6.1.6
Deploy user:        tidb
SSH type:           builtin
Dashboard URL:      http://192.168.93.128:2379/dashboard
Grafana URL:        http://192.168.93.128:3000
ID                    Role          Host            Ports                            OS/Arch       Status           Data Dir                      Deploy Dir
--                    ----          ----            -----                            -------       ------           --------                      ----------
192.168.93.128:9093   alertmanager  192.168.93.128  9093/9094                        linux/x86_64  Up               /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
192.168.93.128:8300   cdc           192.168.93.128  8300                             linux/x86_64  Up               /tidb-data/cdc-8300           /tidb-deploy/cdc-8300
192.168.93.128:3000   grafana       192.168.93.128  3000                             linux/x86_64  Up               -                             /tidb-deploy/grafana-3000
192.168.93.128:2379   pd            192.168.93.128  2379/2380                        linux/x86_64  Up|UI            /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.129:2379   pd            192.168.93.129  2379/2380                        linux/x86_64  Up|L             /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.130:2379   pd            192.168.93.130  2379/2380                        linux/x86_64  Up               /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.128:9090   prometheus    192.168.93.128  9090/12020                       linux/x86_64  Up               /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
192.168.93.128:4000   tidb          192.168.93.128  4000/10080                       linux/x86_64  Up               -                             /tidb-deploy/tidb-4000
192.168.93.129:9000   tiflash       192.168.93.129  9000/8123/3930/20170/20292/8234  linux/x86_64  Up               /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.130:9000   tiflash       192.168.93.130  9000/8123/3930/20170/20292/8234  linux/x86_64  Up               /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.128:20160  tikv          192.168.93.128  20160/20180                      linux/x86_64  Pending Offline  /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.129:20160  tikv          192.168.93.129  20160/20180                      linux/x86_64  Up               /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.130:20160  tikv          192.168.93.130  20160/20180                      linux/x86_64  Up               /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 13
[tidb@BigData01 ~]$ 
```



发现缩容之后，该缩容的TiKV节点仍然一直处于Pending Offline的状态，并没有处于tombstone的状态；
原因如下：因为原来集群就只有3个TiKV节点，现在缩容1个，就只有2个TiKV节点了，那肯定是不行的，因为TiKV默认的副本就是3个，所以存活的TiKV节点是不能少于3个的；

通过命令可以发现，对应TiKV节点的store状态是offline，但是region_count和used_size没有减少；

```shell
[tidb@BigData01 ~]$ tiup ctl:v6.1.6 pd -u http://192.168.93.128:2379 store
The component `ctl` version v6.1.6 is not installed; downloading from repository.
download https://tiup-mirrors.pingcap.com/ctl-v6.1.6-linux-amd64.tar.gz 282.76 MiB / 282.76 MiB 100.00% 11.11 MiB/s                                                                                                                                                                        
Starting component `ctl`: /home/tidb/.tiup/components/ctl/v6.1.6/ctl pd -u http://192.168.93.128:2379 store
{
  "count": 5,
  "stores": [
    {
      "store": {
        "id": 1,
        "address": "192.168.93.128:20160",
        "version": "6.1.6",
        "peer_address": "192.168.93.128:20160",
        "status_address": "192.168.93.128:20180",
        "git_hash": "a80fcf47da7a3d548617a9df66be98d32ada56e1",
        "start_timestamp": 1685843103,
        "deploy_path": "/tidb-deploy/tikv-20160/bin",
        "last_heartbeat": 1685871414941590113,
        "state_name": "Offline"
      },
      "status": {
        "capacity": "36.97GiB",
        "available": "22.27GiB",
        "used_size": "288.9MiB",
        "leader_count": 0,
        "leader_weight": 1,
        "leader_score": 0,
        "leader_size": 0,
        "region_count": 7,
        "region_weight": 1,
        "region_score": 5405879793.309714,
        "region_size": 7,
        "slow_score": 1,
        "start_ts": "2023-06-04T09:45:03+08:00",
        "last_heartbeat_ts": "2023-06-04T17:36:54.941590113+08:00",
        "uptime": "7h51m51.941590113s"
      }
    },
    {
      "store": {
        "id": 4,
        "address": "192.168.93.130:20160",
        "version": "6.1.6",
        "peer_address": "192.168.93.130:20160",
        "status_address": "192.168.93.130:20180",
        "git_hash": "a80fcf47da7a3d548617a9df66be98d32ada56e1",
        "start_timestamp": 1685843107,
        "deploy_path": "/tidb-deploy/tikv-20160/bin",
        "last_heartbeat": 1685871409092251263,
        "state_name": "Up"
      },
      "status": {
        "capacity": "36.97GiB",
        "available": "27.69GiB",
        "used_size": "288.1MiB",
        "leader_count": 5,
        "leader_weight": 1,
        "leader_score": 5,
        "leader_size": 5,
        "region_count": 7,
        "region_weight": 1,
        "region_score": 4462169001.431313,
        "region_size": 7,
        "slow_score": 1,
        "start_ts": "2023-06-04T09:45:07+08:00",
        "last_heartbeat_ts": "2023-06-04T17:36:49.092251263+08:00",
        "uptime": "7h51m42.092251263s"
      }
    },
    {
      "store": {
        "id": 5,
        "address": "192.168.93.129:20160",
        "version": "6.1.6",
        "peer_address": "192.168.93.129:20160",
        "status_address": "192.168.93.129:20180",
        "git_hash": "a80fcf47da7a3d548617a9df66be98d32ada56e1",
        "start_timestamp": 1685843103,
        "deploy_path": "/tidb-deploy/tikv-20160/bin",
        "last_heartbeat": 1685871405707909125,
        "state_name": "Up"
      },
      "status": {
        "capacity": "36.97GiB",
        "available": "27.56GiB",
        "used_size": "288.1MiB",
        "leader_count": 2,
        "leader_weight": 1,
        "leader_score": 2,
        "leader_size": 2,
        "region_count": 7,
        "region_weight": 1,
        "region_score": 4487841292.650547,
        "region_size": 7,
        "slow_score": 1,
        "start_ts": "2023-06-04T09:45:03+08:00",
        "last_heartbeat_ts": "2023-06-04T17:36:45.707909125+08:00",
        "uptime": "7h51m42.707909125s"
      }
    },
    {
      "store": {
        "id": 136,
        "address": "192.168.93.129:3930",
        "labels": [
          {
            "key": "engine",
            "value": "tiflash"
          }
        ],
        "version": "v6.1.6",
        "peer_address": "192.168.93.129:20170",
        "status_address": "192.168.93.129:20292",
        "git_hash": "d5f20a089e490ea40a0dd05bbf21f8ba6ebafe5c",
        "start_timestamp": 1685843117,
        "deploy_path": "/tidb-deploy/tiflash-9000/bin/tiflash",
        "last_heartbeat": 1685871409249507042,
        "state_name": "Up"
      },
      "status": {
        "capacity": "36.97GiB",
        "available": "27.56GiB",
        "used_size": "21.15KiB",
        "leader_count": 0,
        "leader_weight": 1,
        "leader_score": 0,
        "leader_size": 0,
        "region_count": 2,
        "region_weight": 1,
        "region_score": 4487839951.577229,
        "region_size": 2,
        "slow_score": 1,
        "start_ts": "2023-06-04T09:45:17+08:00",
        "last_heartbeat_ts": "2023-06-04T17:36:49.249507042+08:00",
        "uptime": "7h51m32.249507042s"
      }
    },
    {
      "store": {
        "id": 137,
        "address": "192.168.93.130:3930",
        "labels": [
          {
            "key": "engine",
            "value": "tiflash"
          }
        ],
        "version": "v6.1.6",
        "peer_address": "192.168.93.130:20170",
        "status_address": "192.168.93.130:20292",
        "git_hash": "d5f20a089e490ea40a0dd05bbf21f8ba6ebafe5c",
        "start_timestamp": 1685843118,
        "deploy_path": "/tidb-deploy/tiflash-9000/bin/tiflash",
        "last_heartbeat": 1685871410056129961,
        "state_name": "Up"
      },
      "status": {
        "capacity": "36.97GiB",
        "available": "27.69GiB",
        "used_size": "21.76KiB",
        "leader_count": 0,
        "leader_weight": 1,
        "leader_score": 0,
        "leader_size": 0,
        "region_count": 2,
        "region_weight": 1,
        "region_score": 4462169471.780401,
        "region_size": 2,
        "slow_score": 1,
        "start_ts": "2023-06-04T09:45:18+08:00",
        "last_heartbeat_ts": "2023-06-04T17:36:50.056129961+08:00",
        "uptime": "7h51m32.056129961s"
      }
    }
  ]
}
```

要想缩容 192.168.93.128:20160 TiKV节点成功，必须先让集群的TiKV节点数在缩容完该节点之后，仍至少有3个存活节点；

## 扩容TiKV节点

扩容一个TiKV节点：192.168.93.131:20160；

先编辑一个scacle-out-tikv.yaml 扩容配置文件

```shell
[tidb@BigData01 ~]$ vi scale-out-for-tikv.yaml
[tidb@BigData01 ~]$ cat scale-out-for-tikv.yaml
tikv_servers:
  - host: 192.168.93.131
    # # SSH port of the server.
    ssh_port: 22
    # # TiKV Server communication port.
    port: 20160
    # # TiKV Server status API port.
    status_port: 20180
    # # TiKV Server deployment file, startup script, configuration file storage directory.
    deploy_dir: "/tidb-deploy/tikv-20160"
    # # TiKV Server data storage directory.
    data_dir: "/tidb-data/tikv-20160"
    # # TiKV Server log file storage directory.
    log_dir: "/tidb-deploy/tikv-20160/log"
    # # The following configs are used to overwrite the `server_configs.tikv` values.
    config:
       log.level: warn
```

扩容TiKV节点

```shell
[tidb@BigData01 ~]$ tiup cluster scale-out tidb-test scale-out-for-tikv.yaml -u root -p
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster scale-out tidb-test scale-out-for-tikv.yaml -u root -p
Input SSH password: 

+ Detect CPU Arch Name
  - Detecting node 192.168.93.131 Arch info ... Done

+ Detect CPU OS Name
  - Detecting node 192.168.93.131 OS info ... Done
Please confirm your topology:
Cluster type:    tidb
Cluster name:    tidb-test
Cluster version: v6.1.6
Role  Host            Ports        OS/Arch       Directories
----  ----            -----        -------       -----------
tikv  192.168.93.131  20160/20180  linux/x86_64  /tidb-deploy/tikv-20160,/tidb-data/tikv-20160
Attention:
    1. If the topology is not what you expected, check your yaml file.
    2. Please confirm there is no port/directory conflicts in same host.
Do you want to continue? [y/N]: (default=N) y
+ [ Serial ] - SSHKeySet: privateKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa, publicKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa.pub
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ Download TiDB components
  - Download tikv:v6.1.6 (linux/amd64) ... Done
  - Download node_exporter: (linux/amd64) ... Done
  - Download blackbox_exporter: (linux/amd64) ... Done
+ Initialize target host environments
  - Initialized host 192.168.93.131  ... Done
+ Deploy TiDB instance
  - Deploy instance tikv -> 192.168.93.131:20160 ... Done
  - Deploy node_exporter -> 192.168.93.131 ... Done
  - Deploy blackbox_exporter -> 192.168.93.131 ... Done
+ Copy certificate to remote host
+ Generate scale-out config
  - Generate scale-out config tikv -> 192.168.93.131:20160 ... Done
+ Init monitor config
  - Generate config node_exporter -> 192.168.93.131 ... Done
  - Generate config blackbox_exporter -> 192.168.93.131 ... Done
Enabling component tikv
	Enabling instance 192.168.93.131:20160
	Enable instance 192.168.93.131:20160 success
Enabling component node_exporter
	Enabling instance 192.168.93.131
	Enable 192.168.93.131 success
Enabling component blackbox_exporter
	Enabling instance 192.168.93.131
	Enable 192.168.93.131 success
+ [ Serial ] - Save meta
+ [ Serial ] - Start new instances
Starting component tikv
	Starting instance 192.168.93.131:20160
	Start instance 192.168.93.131:20160 success
Starting component node_exporter
	Starting instance 192.168.93.131
	Start 192.168.93.131 success
Starting component blackbox_exporter
	Starting instance 192.168.93.131
	Start 192.168.93.131 success
+ Refresh components conifgs
  - Generate config pd -> 192.168.93.128:2379 ... Done
  - Generate config pd -> 192.168.93.129:2379 ... Done
  - Generate config pd -> 192.168.93.130:2379 ... Done
  - Generate config tikv -> 192.168.93.128:20160 ... Done
  - Generate config tikv -> 192.168.93.129:20160 ... Done
  - Generate config tikv -> 192.168.93.130:20160 ... Done
  - Generate config tikv -> 192.168.93.131:20160 ... Done
  - Generate config tidb -> 192.168.93.128:4000 ... Done
  - Generate config tiflash -> 192.168.93.129:9000 ... Done
  - Generate config tiflash -> 192.168.93.130:9000 ... Done
  - Generate config cdc -> 192.168.93.128:8300 ... Done
  - Generate config prometheus -> 192.168.93.128:9090 ... Done
  - Generate config grafana -> 192.168.93.128:3000 ... Done
  - Generate config alertmanager -> 192.168.93.128:9093 ... Done
+ Reload prometheus and grafana
  - Reload prometheus -> 192.168.93.128:9090 ... Done
  - Reload grafana -> 192.168.93.128:3000 ... Done
+ [ Serial ] - UpdateTopology: cluster=tidb-test
Scaled cluster `tidb-test` out successfully
You have mail in /var/spool/mail/root
[tidb@BigData01 ~]$ 
[tidb@BigData01 ~]$ tiup cluster display tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v6.1.6
Deploy user:        tidb
SSH type:           builtin
Dashboard URL:      http://192.168.93.128:2379/dashboard
Grafana URL:        http://192.168.93.128:3000
ID                    Role          Host            Ports                            OS/Arch       Status     Data Dir                      Deploy Dir
--                    ----          ----            -----                            -------       ------     --------                      ----------
192.168.93.128:9093   alertmanager  192.168.93.128  9093/9094                        linux/x86_64  Up         /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
192.168.93.128:8300   cdc           192.168.93.128  8300                             linux/x86_64  Up         /tidb-data/cdc-8300           /tidb-deploy/cdc-8300
192.168.93.128:3000   grafana       192.168.93.128  3000                             linux/x86_64  Up         -                             /tidb-deploy/grafana-3000
192.168.93.128:2379   pd            192.168.93.128  2379/2380                        linux/x86_64  Up|UI      /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.129:2379   pd            192.168.93.129  2379/2380                        linux/x86_64  Up|L       /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.130:2379   pd            192.168.93.130  2379/2380                        linux/x86_64  Up         /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.128:9090   prometheus    192.168.93.128  9090/12020                       linux/x86_64  Up         /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
192.168.93.128:4000   tidb          192.168.93.128  4000/10080                       linux/x86_64  Up         -                             /tidb-deploy/tidb-4000
192.168.93.129:9000   tiflash       192.168.93.129  9000/8123/3930/20170/20292/8234  linux/x86_64  Up         /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.130:9000   tiflash       192.168.93.130  9000/8123/3930/20170/20292/8234  linux/x86_64  Up         /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.128:20160  tikv          192.168.93.128  20160/20180                      linux/x86_64  Tombstone  /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.129:20160  tikv          192.168.93.129  20160/20180                      linux/x86_64  Up         /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.130:20160  tikv          192.168.93.130  20160/20180                      linux/x86_64  Up         /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.131:20160  tikv          192.168.93.131  20160/20180                      linux/x86_64  Up         /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 14
There are some nodes can be pruned: 
	Nodes: [192.168.93.128:20160]
	You can destroy them with the command: `tiup cluster prune tidb-test`
```

> 扩容完该TiKV节点之后，发现原有在只有3个TiKV节点时缩容的那个TiKV节点不再一直处于Pending Offline，而是处于Tombstone（下线）状态了；因为多扩容一个TiKV节点之后，原有缩容一直处于Pending Offline的TiKV节点就可以把数据转移到新扩容的TiKV节点上，从而满足TiKV默认三副本的情况至少需要有3个TiKV节点存活的条件；

此时再卸载已经被缩容的TiKV节点（即清理缩容的TiKV节点信息），这样缩容操作就全部完成了；

此时，再次查看集群状态，就可以发现 192.168.93.128:20160 TiKV 节点就已经不存在了，说明缩容成功；

```shell
[tidb@BigData01 ~]$ tiup cluster prune tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster prune tidb-test
+ [ Serial ] - SSHKeySet: privateKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa, publicKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa.pub
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.131
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [ Serial ] - FindTomestoneNodes
Will destroy these nodes: [192.168.93.128:20160]
Do you confirm this action? [y/N]:(default=N) y
Start destroy Tombstone nodes: [192.168.93.128:20160] ...
+ [ Serial ] - ClusterOperate: operation=ScaleInOperation, options={Roles:[] Nodes:[] Force:true SSHTimeout:5 OptTimeout:120 APITimeout:600 IgnoreConfigCheck:true NativeSSH:false SSHType: Concurrency:5 SSHProxyHost: SSHProxyPort:22 SSHProxyUser:tidb SSHProxyIdentity:/home/tidb/.ssh/id_rsa SSHProxyUsePassword:false SSHProxyTimeout:5 SSHCustomScripts:{BeforeRestartInstance:{Raw:} AfterRestartInstance:{Raw:}} CleanupData:false CleanupLog:false CleanupAuditLog:false RetainDataRoles:[] RetainDataNodes:[] DisplayMode:default Operation:StartOperation}
Stopping component tikv
	Stopping instance 192.168.93.128
	Stop tikv 192.168.93.128:20160 success
Destroying component tikv
	Destroying instance 192.168.93.128
Destroy 192.168.93.128 success
- Destroy tikv paths: [/tidb-deploy/tikv-20160 /etc/systemd/system/tikv-20160.service /tidb-data/tikv-20160 /tidb-deploy/tikv-20160/log]
+ [ Serial ] - UpdateMeta: cluster=tidb-test, deleted=`'192.168.93.128:20160'`
+ [ Serial ] - UpdateTopology: cluster=tidb-test
+ Refresh instance configs
  - Generate config pd -> 192.168.93.128:2379 ... Done
  - Generate config pd -> 192.168.93.129:2379 ... Done
  - Generate config pd -> 192.168.93.130:2379 ... Done
  - Generate config tikv -> 192.168.93.129:20160 ... Done
  - Generate config tikv -> 192.168.93.130:20160 ... Done
  - Generate config tikv -> 192.168.93.131:20160 ... Done
  - Generate config tidb -> 192.168.93.128:4000 ... Done
  - Generate config tiflash -> 192.168.93.129:9000 ... Done
  - Generate config tiflash -> 192.168.93.130:9000 ... Done
  - Generate config cdc -> 192.168.93.128:8300 ... Done
  - Generate config prometheus -> 192.168.93.128:9090 ... Done
  - Generate config grafana -> 192.168.93.128:3000 ... Done
  - Generate config alertmanager -> 192.168.93.128:9093 ... Done
+ Reload prometheus and grafana
  - Reload prometheus -> 192.168.93.128:9090 ... Done
  - Reload grafana -> 192.168.93.128:3000 ... Done
Destroy success
[tidb@BigData01 ~]$ 
[tidb@BigData01 ~]$ 
[tidb@BigData01 ~]$ tiup cluster display tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v6.1.6
Deploy user:        tidb
SSH type:           builtin
Dashboard URL:      http://192.168.93.128:2379/dashboard
Grafana URL:        http://192.168.93.128:3000
ID                    Role          Host            Ports                            OS/Arch       Status  Data Dir                      Deploy Dir
--                    ----          ----            -----                            -------       ------  --------                      ----------
192.168.93.128:9093   alertmanager  192.168.93.128  9093/9094                        linux/x86_64  Up      /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
192.168.93.128:8300   cdc           192.168.93.128  8300                             linux/x86_64  Up      /tidb-data/cdc-8300           /tidb-deploy/cdc-8300
192.168.93.128:3000   grafana       192.168.93.128  3000                             linux/x86_64  Up      -                             /tidb-deploy/grafana-3000
192.168.93.128:2379   pd            192.168.93.128  2379/2380                        linux/x86_64  Up|UI   /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.129:2379   pd            192.168.93.129  2379/2380                        linux/x86_64  Up|L    /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.130:2379   pd            192.168.93.130  2379/2380                        linux/x86_64  Up      /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.128:9090   prometheus    192.168.93.128  9090/12020                       linux/x86_64  Up      /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
192.168.93.128:4000   tidb          192.168.93.128  4000/10080                       linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
192.168.93.129:9000   tiflash       192.168.93.129  9000/8123/3930/20170/20292/8234  linux/x86_64  Up      /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.130:9000   tiflash       192.168.93.130  9000/8123/3930/20170/20292/8234  linux/x86_64  Up      /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.129:20160  tikv          192.168.93.129  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.130:20160  tikv          192.168.93.130  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.131:20160  tikv          192.168.93.131  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 13
```

## 缩容TiFlash节点

缩容TiFlash节点：192.168.93.130:9000;

先将创建了TiFlash副本的表的TiFlash副本数都设置为0；

```sql
MySQL [(none)]> select * from information_schema.tiflash_replica;
+--------------+------------+----------+---------------+-----------------+-----------+----------+
| TABLE_SCHEMA | TABLE_NAME | TABLE_ID | REPLICA_COUNT | LOCATION_LABELS | AVAILABLE | PROGRESS |
+--------------+------------+----------+---------------+-----------------+-----------+----------+
| lsy          | cust       |       71 |             2 |                 |         1 |        1 |
| lsy          | acct       |       73 |             2 |                 |         1 |        1 |
+--------------+------------+----------+---------------+-----------------+-----------+----------+
2 rows in set (0.02 sec)

MySQL [(none)]> 
MySQL [(none)]> alter table lsy.acct set tiflash replica 0;
Query OK, 0 rows affected (0.12 sec)

MySQL [(none)]> alter table lsy.cust set tiflash replica 0;
Query OK, 0 rows affected (0.11 sec)
```

缩容TiFlash节点

```shell
[tidb@BigData01 ~]$ tiup cluster scale-in tidb-test --node 192.168.93.130:9000
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster scale-in tidb-test --node 192.168.93.130:9000
This operation will delete the 192.168.93.130:9000 nodes in `tidb-test` and all their data.
Do you want to continue? [y/N]:(default=N) y
The component `[tiflash]` will become tombstone, maybe exists in several minutes or hours, after that you can use the prune command to clean it
Do you want to continue? [y/N]:(default=N) y
Scale-in nodes...
+ [ Serial ] - SSHKeySet: privateKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa, publicKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa.pub
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [ Serial ] - ClusterOperate: operation=DestroyOperation, options={Roles:[] Nodes:[192.168.93.130:9000] Force:false SSHTimeout:5 OptTimeout:120 APITimeout:600 IgnoreConfigCheck:false NativeSSH:false SSHType: Concurrency:5 SSHProxyHost: SSHProxyPort:22 SSHProxyUser:tidb SSHProxyIdentity:/home/tidb/.ssh/id_rsa SSHProxyUsePassword:false SSHProxyTimeout:5 SSHCustomScripts:{BeforeRestartInstance:{Raw:} AfterRestartInstance:{Raw:}} CleanupData:false CleanupLog:false CleanupAuditLog:false RetainDataRoles:[] RetainDataNodes:[] DisplayMode:default Operation:StartOperation}
The component `tiflash` will become tombstone, maybe exists in several minutes or hours, after that you can use the prune command to clean it
+ [ Serial ] - UpdateMeta: cluster=tidb-test, deleted=`''`
+ [ Serial ] - UpdateTopology: cluster=tidb-test
+ Refresh instance configs
  - Generate config pd -> 192.168.93.128:2379 ... Done
  - Generate config pd -> 192.168.93.129:2379 ... Done
  - Generate config pd -> 192.168.93.130:2379 ... Done
  - Generate config tikv -> 192.168.93.129:20160 ... Done
  - Generate config tikv -> 192.168.93.130:20160 ... Done
  - Generate config tikv -> 192.168.93.128:20160 ... Done
  - Generate config tidb -> 192.168.93.128:4000 ... Done
  - Generate config tiflash -> 192.168.93.129:9000 ... Done
  - Generate config cdc -> 192.168.93.128:8300 ... Done
  - Generate config prometheus -> 192.168.93.128:9090 ... Done
  - Generate config grafana -> 192.168.93.128:3000 ... Done
  - Generate config alertmanager -> 192.168.93.128:9093 ... Done
+ Reload prometheus and grafana
  - Reload prometheus -> 192.168.93.128:9090 ... Done
  - Reload grafana -> 192.168.93.128:3000 ... Done
Scaled cluster `tidb-test` in successfully
```

缩容后，TiFlash节点状态变为Tombstone（即下线状态）；

```shell
[tidb@BigData01 ~]$ tiup cluster display tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v6.1.6
Deploy user:        tidb
SSH type:           builtin
Dashboard URL:      http://192.168.93.128:2379/dashboard
Grafana URL:        http://192.168.93.128:3000
ID                    Role          Host            Ports                            OS/Arch       Status     Data Dir                      Deploy Dir
--                    ----          ----            -----                            -------       ------     --------                      ----------
192.168.93.128:9093   alertmanager  192.168.93.128  9093/9094                        linux/x86_64  Up         /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
192.168.93.128:8300   cdc           192.168.93.128  8300                             linux/x86_64  Up         /tidb-data/cdc-8300           /tidb-deploy/cdc-8300
192.168.93.128:3000   grafana       192.168.93.128  3000                             linux/x86_64  Up         -                             /tidb-deploy/grafana-3000
192.168.93.128:2379   pd            192.168.93.128  2379/2380                        linux/x86_64  Up|UI      /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.129:2379   pd            192.168.93.129  2379/2380                        linux/x86_64  Up|L       /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.130:2379   pd            192.168.93.130  2379/2380                        linux/x86_64  Up         /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.128:9090   prometheus    192.168.93.128  9090/12020                       linux/x86_64  Up         /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
192.168.93.128:4000   tidb          192.168.93.128  4000/10080                       linux/x86_64  Up         -                             /tidb-deploy/tidb-4000
192.168.93.129:9000   tiflash       192.168.93.129  9000/8123/3930/20170/20292/8234  linux/x86_64  Up         /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.130:9000   tiflash       192.168.93.130  9000/8123/3930/20170/20292/8234  linux/x86_64  Tombstone  /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.128:20160  tikv          192.168.93.128  20160/20180                      linux/x86_64  Up         /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.129:20160  tikv          192.168.93.129  20160/20180                      linux/x86_64  Up         /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.130:20160  tikv          192.168.93.130  20160/20180                      linux/x86_64  Up         /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 13
There are some nodes can be pruned: 
	Nodes: [192.168.93.130:3930]
	You can destroy them with the command: `tiup cluster prune tidb-test`
```

> 如果在缩容TiFlash节点之前，没有将设置了TiFlash副本的表的TiFlash副本数设置为0，则缩容完TiFlash节点之后，该节点会一直处于Pending Offline的状态；只有表的TiFlash副本数都设置为0之后，该缩容的TiFlash节点才会变成Tombstone状态；

最后，卸载TiFlash节点，即清理TiFlash节点信息

```shell
[tidb@BigData01 ~]$ tiup cluster prune tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster prune tidb-test
+ [ Serial ] - SSHKeySet: privateKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa, publicKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa.pub
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [ Serial ] - FindTomestoneNodes
Will destroy these nodes: [192.168.93.130:3930]
Do you confirm this action? [y/N]:(default=N) y
Start destroy Tombstone nodes: [192.168.93.130:3930] ...
+ [ Serial ] - ClusterOperate: operation=ScaleInOperation, options={Roles:[] Nodes:[] Force:true SSHTimeout:5 OptTimeout:120 APITimeout:600 IgnoreConfigCheck:true NativeSSH:false SSHType: Concurrency:5 SSHProxyHost: SSHProxyPort:22 SSHProxyUser:tidb SSHProxyIdentity:/home/tidb/.ssh/id_rsa SSHProxyUsePassword:false SSHProxyTimeout:5 SSHCustomScripts:{BeforeRestartInstance:{Raw:} AfterRestartInstance:{Raw:}} CleanupData:false CleanupLog:false CleanupAuditLog:false RetainDataRoles:[] RetainDataNodes:[] DisplayMode:default Operation:StartOperation}
Stopping component tiflash
	Stopping instance 192.168.93.130
	Stop tiflash 192.168.93.130:9000 success
Destroying component tiflash
	Destroying instance 192.168.93.130
Destroy 192.168.93.130 success
- Destroy tiflash paths: [/tidb-data/tiflash-9000 /tidb-deploy/tiflash-9000/log /tidb-deploy/tiflash-9000 /etc/systemd/system/tiflash-9000.service]
+ [ Serial ] - UpdateMeta: cluster=tidb-test, deleted=`'192.168.93.130:3930'`
+ [ Serial ] - UpdateTopology: cluster=tidb-test
+ Refresh instance configs
  - Generate config pd -> 192.168.93.128:2379 ... Done
  - Generate config pd -> 192.168.93.129:2379 ... Done
  - Generate config pd -> 192.168.93.130:2379 ... Done
  - Generate config tikv -> 192.168.93.129:20160 ... Done
  - Generate config tikv -> 192.168.93.130:20160 ... Done
  - Generate config tikv -> 192.168.93.128:20160 ... Done
  - Generate config tidb -> 192.168.93.128:4000 ... Done
  - Generate config tiflash -> 192.168.93.129:9000 ... Done
  - Generate config tiflash -> 192.168.93.130:9000 ... Error
  - Generate config cdc -> 192.168.93.128:8300 ... Done
  - Generate config prometheus -> 192.168.93.128:9090 ... Done
  - Generate config grafana -> 192.168.93.128:3000 ... Done
  - Generate config alertmanager -> 192.168.93.128:9093 ... Done
+ Reload prometheus and grafana
  - Reload prometheus -> 192.168.93.128:9090 ... Done
  - Reload grafana -> 192.168.93.128:3000 ... Done
Destroy success
[tidb@BigData01 ~]$ 
[tidb@BigData01 ~]$ 
[tidb@BigData01 ~]$ 
[tidb@BigData01 ~]$ 
[tidb@BigData01 ~]$ tiup cluster display tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v6.1.6
Deploy user:        tidb
SSH type:           builtin
Dashboard URL:      http://192.168.93.128:2379/dashboard
Grafana URL:        http://192.168.93.128:3000
ID                    Role          Host            Ports                            OS/Arch       Status  Data Dir                      Deploy Dir
--                    ----          ----            -----                            -------       ------  --------                      ----------
192.168.93.128:9093   alertmanager  192.168.93.128  9093/9094                        linux/x86_64  Up      /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
192.168.93.128:8300   cdc           192.168.93.128  8300                             linux/x86_64  Up      /tidb-data/cdc-8300           /tidb-deploy/cdc-8300
192.168.93.128:3000   grafana       192.168.93.128  3000                             linux/x86_64  Up      -                             /tidb-deploy/grafana-3000
192.168.93.128:2379   pd            192.168.93.128  2379/2380                        linux/x86_64  Up|UI   /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.129:2379   pd            192.168.93.129  2379/2380                        linux/x86_64  Up|L    /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.130:2379   pd            192.168.93.130  2379/2380                        linux/x86_64  Up      /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.128:9090   prometheus    192.168.93.128  9090/12020                       linux/x86_64  Up      /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
192.168.93.128:4000   tidb          192.168.93.128  4000/10080                       linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
192.168.93.129:9000   tiflash       192.168.93.129  9000/8123/3930/20170/20292/8234  linux/x86_64  Up      /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.128:20160  tikv          192.168.93.128  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.129:20160  tikv          192.168.93.129  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.130:20160  tikv          192.168.93.130  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 12
```

## 扩容TiFlash节点

扩容TiFlash节点：192.168.93.130:9000；

编辑扩容配置文件

```shell
[tidb@BigData01 ~]$ vi scale-out-for-tiflash.yaml 
[tidb@BigData01 ~]$ 
[tidb@BigData01 ~]$ cat scale-out-for-tiflash.yaml 
tiflash_servers:
  - host: 192.168.93.130
    ssh_port: 22
    tcp_port: 9000
    flash_service_port: 3930
    flash_proxy_port: 20170
    flash_proxy_status_port: 20292
    metrics_port: 8234
    deploy_dir: /tidb-deploy/tiflash-9000
    data_dir: /tidb-data/tiflash-9000
    log_dir: /tidb-deploy/tiflash-9000/log
```

扩容TiFlash节点

```shell
[tidb@BigData01 ~]$ tiup cluster scale-out tidb-test  scale-out-for-tiflash.yaml -u root -p
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster scale-out tidb-test scale-out-for-tiflash.yaml -u root -p
Input SSH password: 

+ Detect CPU Arch Name
  - Detecting node 192.168.93.130 Arch info ... Done

+ Detect CPU OS Name
  - Detecting node 192.168.93.130 OS info ... Done
Please confirm your topology:
Cluster type:    tidb
Cluster name:    tidb-test
Cluster version: v6.1.6
Role     Host            Ports                            OS/Arch       Directories
----     ----            -----                            -------       -----------
tiflash  192.168.93.130  9000/8123/3930/20170/20292/8234  linux/x86_64  /tidb-deploy/tiflash-9000,/tidb-data/tiflash-9000
Attention:
    1. If the topology is not what you expected, check your yaml file.
    2. Please confirm there is no port/directory conflicts in same host.
Do you want to continue? [y/N]: (default=N) y
+ [ Serial ] - SSHKeySet: privateKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa, publicKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa.pub
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ Download TiDB components
  - Download tiflash:v6.1.6 (linux/amd64) ... Done
+ Initialize target host environments
+ Deploy TiDB instance
  - Deploy instance tiflash -> 192.168.93.130:9000 ... Done
+ Copy certificate to remote host
+ Generate scale-out config
  - Generate scale-out config tiflash -> 192.168.93.130:9000 ... Done
+ Init monitor config
Enabling component tiflash
	Enabling instance 192.168.93.130:9000
	Enable instance 192.168.93.130:9000 success
Enabling component node_exporter
	Enabling instance 192.168.93.130
	Enable 192.168.93.130 success
Enabling component blackbox_exporter
	Enabling instance 192.168.93.130
	Enable 192.168.93.130 success
+ [ Serial ] - Save meta
+ [ Serial ] - Start new instances
Starting component tiflash
	Starting instance 192.168.93.130:9000
	Start instance 192.168.93.130:9000 success
Starting component node_exporter
	Starting instance 192.168.93.130
	Start 192.168.93.130 success
Starting component blackbox_exporter
	Starting instance 192.168.93.130
	Start 192.168.93.130 success
+ Refresh components conifgs
  - Generate config pd -> 192.168.93.128:2379 ... Done
  - Generate config pd -> 192.168.93.129:2379 ... Done
  - Generate config pd -> 192.168.93.130:2379 ... Done
  - Generate config tikv -> 192.168.93.129:20160 ... Done
  - Generate config tikv -> 192.168.93.130:20160 ... Done
  - Generate config tikv -> 192.168.93.128:20160 ... Done
  - Generate config tidb -> 192.168.93.128:4000 ... Done
  - Generate config tiflash -> 192.168.93.129:9000 ... Done
  - Generate config tiflash -> 192.168.93.130:9000 ... Done
  - Generate config cdc -> 192.168.93.128:8300 ... Done
  - Generate config prometheus -> 192.168.93.128:9090 ... Done
  - Generate config grafana -> 192.168.93.128:3000 ... Done
  - Generate config alertmanager -> 192.168.93.128:9093 ... Done
+ Reload prometheus and grafana
  - Reload prometheus -> 192.168.93.128:9090 ... Done
  - Reload grafana -> 192.168.93.128:3000 ... Done
+ [ Serial ] - UpdateTopology: cluster=tidb-test
Scaled cluster `tidb-test` out successfully
[tidb@BigData01 ~]$ 
[tidb@BigData01 ~]$ tiup cluster display tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v6.1.6
Deploy user:        tidb
SSH type:           builtin
Dashboard URL:      http://192.168.93.128:2379/dashboard
Grafana URL:        http://192.168.93.128:3000
ID                    Role          Host            Ports                            OS/Arch       Status  Data Dir                      Deploy Dir
--                    ----          ----            -----                            -------       ------  --------                      ----------
192.168.93.128:9093   alertmanager  192.168.93.128  9093/9094                        linux/x86_64  Up      /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
192.168.93.128:8300   cdc           192.168.93.128  8300                             linux/x86_64  Up      /tidb-data/cdc-8300           /tidb-deploy/cdc-8300
192.168.93.128:3000   grafana       192.168.93.128  3000                             linux/x86_64  Up      -                             /tidb-deploy/grafana-3000
192.168.93.128:2379   pd            192.168.93.128  2379/2380                        linux/x86_64  Up|UI   /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.129:2379   pd            192.168.93.129  2379/2380                        linux/x86_64  Up|L    /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.130:2379   pd            192.168.93.130  2379/2380                        linux/x86_64  Up      /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.128:9090   prometheus    192.168.93.128  9090/12020                       linux/x86_64  Up      /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
192.168.93.128:4000   tidb          192.168.93.128  4000/10080                       linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
192.168.93.129:9000   tiflash       192.168.93.129  9000/8123/3930/20170/20292/8234  linux/x86_64  Up      /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.130:9000   tiflash       192.168.93.130  9000/8123/3930/20170/20292/8234  linux/x86_64  Up      /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.128:20160  tikv          192.168.93.128  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.129:20160  tikv          192.168.93.129  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.130:20160  tikv          192.168.93.130  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 13
```

创建TiFlash副本试试

```sql
MySQL [(none)]> alter table lsy.acct set tiflash replica 2;
Query OK, 0 rows affected (0.12 sec)

MySQL [(none)]> alter table lsy.cust set tiflash replica 2;
Query OK, 0 rows affected (0.10 sec)

MySQL [(none)]> 
MySQL [(none)]> select * from information_schema.tiflash_replica;
+--------------+------------+----------+---------------+-----------------+-----------+----------+
| TABLE_SCHEMA | TABLE_NAME | TABLE_ID | REPLICA_COUNT | LOCATION_LABELS | AVAILABLE | PROGRESS |
+--------------+------------+----------+---------------+-----------------+-----------+----------+
| lsy          | acct       |       73 |             2 |                 |         1 |        1 |
| lsy          | cust       |       71 |             2 |                 |         1 |        1 |
+--------------+------------+----------+---------------+-----------------+-----------+----------+
2 rows in set (0.00 sec)
```

# TiDB集群版本升级

## 检查集群监控状态

```shell
[tidb@BigData01 ~]$ tiup cluster check tidb-test --cluster
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster check tidb-test --cluster
+ Download necessary tools
  - Downloading check tools for linux/amd64 ... Done
+ Collect basic system information
  - Getting system info of 192.168.93.129:22 ... ⠴ CopyComponent: component=insight, version=, remote=192.168.93.129:/tmp/tiup os=linux, arch=amd64
+ Collect basic system information
+ Collect basic system information
  - Getting system info of 192.168.93.129:22 ... Done
  - Getting system info of 192.168.93.130:22 ... Done
  - Getting system info of 192.168.93.128:22 ... Done
+ Check time zone
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
+ Check system requirements
+ Check system requirements
  - Checking node 192.168.93.129 ... Done
+ Check system requirements
+ Check system requirements
+ Check system requirements
+ Check system requirements
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.128 ... Done
  - Checking node 192.168.93.129 ... Done
  - Checking node 192.168.93.130 ... Done
  - Checking node 192.168.93.128 ... Done
+ Cleanup check files
  - Cleanup check files on 192.168.93.129:22 ... Done
  - Cleanup check files on 192.168.93.130:22 ... Done
  - Cleanup check files on 192.168.93.128:22 ... Done
Node            Check         Result  Message
----            -----         ------  -------
192.168.93.128  swap          Warn    swap is enabled, please disable it for best performance
192.168.93.128  disk          Warn    mount point / does not have 'noatime' option set
192.168.93.128  sysctl        Fail    vm.swappiness = 70, should be 0
192.168.93.128  selinux       Pass    SELinux is disabled
192.168.93.128  thp           Fail    THP is enabled, please disable it for best performance
192.168.93.128  permission    Pass    /tidb-deploy/prometheus-9090 is writable
192.168.93.128  permission    Pass    /tidb-deploy/alertmanager-9093 is writable
192.168.93.128  permission    Pass    /tidb-data/prometheus-9090 is writable
192.168.93.128  permission    Pass    /tidb-deploy/grafana-3000 is writable
192.168.93.128  permission    Pass    /tidb-data/tikv-20160 is writable
192.168.93.128  permission    Pass    /tidb-data/cdc-8300 is writable
192.168.93.128  permission    Pass    /tidb-deploy/pd-2379 is writable
192.168.93.128  permission    Pass    /tidb-data/pd-2379 is writable
192.168.93.128  permission    Pass    /tidb-deploy/tikv-20160 is writable
192.168.93.128  permission    Pass    /tidb-deploy/tidb-4000 is writable
192.168.93.128  permission    Pass    /tidb-deploy/cdc-8300 is writable
192.168.93.128  permission    Pass    /tidb-data/alertmanager-9093 is writable
192.168.93.128  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.128  cpu-cores     Pass    number of CPU cores / threads: 1
192.168.93.128  cpu-governor  Warn    Unable to determine current CPU frequency governor policy
192.168.93.128  memory        Pass    memory size is 6144MB
192.168.93.128  network       Pass    network speed of ens33 is 1000MB
192.168.93.128  service       Fail    service irqbalance is not running
192.168.93.128  command       Pass    numactl: policy: default
192.168.93.129  command       Pass    numactl: policy: default
192.168.93.129  selinux       Pass    SELinux is disabled
192.168.93.129  swap          Warn    swap is enabled, please disable it for best performance
192.168.93.129  disk          Warn    mount point / does not have 'noatime' option set
192.168.93.129  thp           Fail    THP is enabled, please disable it for best performance
192.168.93.129  cpu-cores     Pass    number of CPU cores / threads: 1
192.168.93.129  cpu-governor  Warn    Unable to determine current CPU frequency governor policy
192.168.93.129  permission    Pass    /tidb-deploy/pd-2379 is writable
192.168.93.129  permission    Pass    /tidb-data/tiflash-9000 is writable
192.168.93.129  permission    Pass    /tidb-data/pd-2379 is writable
192.168.93.129  permission    Pass    /tidb-deploy/tikv-20160 is writable
192.168.93.129  permission    Pass    /tidb-data/tikv-20160 is writable
192.168.93.129  permission    Pass    /tidb-deploy/tiflash-9000 is writable
192.168.93.129  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.129  memory        Pass    memory size is 3072MB
192.168.93.129  network       Pass    network speed of ens33 is 1000MB
192.168.93.129  disk          Fail    multiple components tikv:/tidb-data/tikv-20160,tiflash:/tidb-data/tiflash-9000 are using the same partition 192.168.93.129:/ as data dir
192.168.93.129  service       Fail    service irqbalance is not running
192.168.93.129  timezone      Pass    time zone is the same as the first PD machine: Asia/Shanghai
192.168.93.130  thp           Fail    THP is enabled, please disable it for best performance
192.168.93.130  disk          Fail    multiple components tikv:/tidb-data/tikv-20160,tiflash:/tidb-data/tiflash-9000 are using the same partition 192.168.93.130:/ as data dir
192.168.93.130  selinux       Pass    SELinux is disabled
192.168.93.130  command       Pass    numactl: policy: default
192.168.93.130  timezone      Pass    time zone is the same as the first PD machine: Asia/Shanghai
192.168.93.130  cpu-governor  Warn    Unable to determine current CPU frequency governor policy
192.168.93.130  network       Pass    network speed of ens33 is 1000MB
192.168.93.130  service       Fail    service irqbalance is not running
192.168.93.130  swap          Warn    swap is enabled, please disable it for best performance
192.168.93.130  memory        Pass    memory size is 3072MB
192.168.93.130  disk          Warn    mount point / does not have 'noatime' option set
192.168.93.130  permission    Pass    /tidb-deploy/pd-2379 is writable
192.168.93.130  permission    Pass    /tidb-deploy/tiflash-9000 is writable
192.168.93.130  permission    Pass    /tidb-data/pd-2379 is writable
192.168.93.130  permission    Pass    /tidb-data/tiflash-9000 is writable
192.168.93.130  permission    Pass    /tidb-deploy/tikv-20160 is writable
192.168.93.130  permission    Pass    /tidb-data/tikv-20160 is writable
192.168.93.130  os-version    Pass    OS is CentOS Linux 7 (Core) 7.7.1908
192.168.93.130  cpu-cores     Pass    number of CPU cores / threads: 1
Checking region status of the cluster tidb-test...
All regions are healthy.
```

## 查看TiUP支持的最新可用版本

```shell
[tidb@BigData01 ~]$ tiup list tidb
Available versions for tidb:
Version                                   Installed  Release                              Platforms
-------                                   ---------  -------                              ---------
build-debug-mode                                     2022-06-10T14:29:34+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
nightly -> v7.2.0-alpha-nightly-20230603             2023-06-03T22:52:22+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v3.0.0                                               2020-04-16T14:03:31+08:00            darwin/amd64,linux/amd64
v3.0                                                 2020-04-16T16:58:06+08:00            darwin/amd64,linux/amd64
v3.0.1                                               2020-04-27T19:38:36+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.0.2                                               2020-04-16T23:55:11+08:00            darwin/amd64,linux/amd64
v3.0.3                                               2020-04-17T00:16:31+08:00            darwin/amd64,linux/amd64
v3.0.4                                               2020-04-17T00:22:46+08:00            darwin/amd64,linux/amd64
v3.0.5                                               2020-04-17T00:29:45+08:00            darwin/amd64,linux/amd64
v3.0.6                                               2020-04-17T00:39:33+08:00            darwin/amd64,linux/amd64
v3.0.7                                               2020-04-17T00:46:32+08:00            darwin/amd64,linux/amd64
v3.0.8                                               2020-04-17T00:54:19+08:00            darwin/amd64,linux/amd64
v3.0.9                                               2020-04-17T01:00:58+08:00            darwin/amd64,linux/amd64
v3.0.10                                              2020-03-13T14:11:53.774527401+08:00  darwin/amd64,linux/amd64
v3.0.11                                              2020-04-17T01:09:20+08:00            darwin/amd64,linux/amd64
v3.0.12                                              2020-04-17T01:16:04+08:00            darwin/amd64,linux/amd64
v3.0.13                                              2020-04-26T17:25:01+08:00            darwin/amd64,linux/amd64
v3.0.14                                              2020-05-09T21:11:49+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.0.15                                              2020-06-05T16:53:32+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.0.16                                              2020-07-03T20:07:45+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.0.17                                              2020-08-03T15:27:29+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.0.18                                              2020-08-21T20:02:59+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.0.19                                              2020-09-25T18:24:11+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.0.20                                              2020-12-25T15:20:36+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.1.0-beta                                          2020-05-22T14:35:59+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.1.0-beta.1                                        2020-05-22T15:22:30+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.1.0-beta.2                                        2020-05-22T15:28:20+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.1.0-rc                                            2020-05-22T15:56:23+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.1.0                                               2020-05-22T15:34:33+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.1.1                                               2020-04-30T21:02:32+08:00            darwin/amd64,linux/amd64,linux/arm64
v3.1.2                                               2020-06-04T17:55:40+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.0-beta                                          2020-05-26T11:18:05+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.0-beta.1                                        2020-05-26T11:42:48+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.0-beta.2                                        2020-05-26T11:56:51+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.0-rc                                            2020-05-26T14:56:06+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.0-rc.1                                          2020-04-29T01:03:31+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.0-rc.2                                          2020-05-15T21:54:51+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.0                                               2020-05-28T20:10:11+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.1                                               2020-06-12T21:21:12+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.2                                               2020-07-01T19:59:39+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.3                                               2020-07-25T01:03:31+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.4                                               2020-07-31T16:44:42+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.5                                               2020-08-31T23:53:35+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.6                                               2020-09-15T22:13:29+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.7                                               2020-09-29T20:15:07+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.8                                               2020-10-30T19:30:22+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.9                                               2020-12-21T17:25:20+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.10                                              2021-01-15T13:15:09+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.11                                              2021-02-26T17:39:23+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.12-20210427                                     2021-05-08T11:09:37+08:00            linux/amd64
v4.0.12                                              2021-04-02T16:55:00+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.13                                              2021-05-27T22:18:32+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.14                                              2021-07-27T18:08:31+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.15                                              2021-09-23T18:38:19+08:00            darwin/amd64,linux/amd64,linux/arm64
v4.0.16                                              2021-12-17T18:09:01+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.0.0-20210329                                      2021-03-29T19:45:19+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.0.0-20210403                                      2021-04-03T09:13:36+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.0.0-20210408                                      2021-04-08T17:02:41+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.0.0-rc                                            2021-01-12T23:40:27+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.0.0                                               2021-04-07T17:30:00+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.0.1                                               2021-04-24T21:31:28+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.0.2                                               2021-06-09T22:49:33+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.0.3                                               2021-07-02T16:13:22+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.0.4                                               2021-09-14T17:53:19+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.0.5                                               2021-12-03T11:26:56+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.0.6                                               2021-12-30T22:41:05+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.1.0                                               2021-06-24T16:23:50+08:00            darwin/amd64,linux/amd64,linux/arm64
v5.1.1                                               2021-07-30T15:59:38+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.1.2                                               2021-09-27T13:02:47+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.1.3                                               2021-12-03T17:46:16+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.1.4                                               2022-02-22T12:40:06+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.1.5                                               2022-12-28T12:18:53+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.2.0                                               2021-08-27T18:40:58+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.2.1                                               2021-09-09T18:50:00+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.2.2                                               2021-10-29T13:51:56+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.2.3                                               2021-12-02T18:42:12+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.2.4                                               2022-04-26T15:36:17+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.3.0                                               2021-11-29T17:05:42+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.3.1                                               2022-03-03T19:44:31+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.3.2                                               2022-06-29T11:05:04+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.3.3                                               2022-09-14T19:16:40+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.3.4                                               2022-11-24T12:36:22+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.4.0                                               2022-02-11T20:11:44+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.4.1                                               2022-05-13T13:48:10+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.4.2                                               2022-07-08T10:12:37+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v5.4.3                                               2022-10-13T22:13:21+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.0.0                                               2022-04-06T11:34:40+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.1.0                                               2022-06-13T12:30:16+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.1.1                                               2022-09-01T12:09:05+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.1.2                                               2022-10-24T15:16:17+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.1.3                                               2022-12-05T11:50:23+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.1.4                                               2023-02-08T11:34:10+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.1.5                                               2023-02-28T11:23:57+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.1.6                                               2023-04-12T11:05:35+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.2.0                                               2022-08-23T09:14:36+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.3.0                                               2022-09-30T10:59:36+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.4.0                                               2022-11-17T11:26:23+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.5.0                                               2022-12-29T11:32:06+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.5.1                                               2023-03-10T13:36:50+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.5.2                                               2023-04-21T10:52:46+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v6.6.0                                               2023-02-20T16:43:16+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v7.0.0                                               2023-03-30T10:33:19+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v7.1.0                                               2023-05-31T14:49:49+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
v7.2.0-alpha-nightly-20230603                        2023-06-03T22:52:22+08:00            darwin/amd64,darwin/arm64,linux/amd64,linux/arm64
```

## 升级TiDB集群

将TiDB集群版本从 V6.1.6 升级为 V6.5.0

```shell
[tidb@BigData01 ~]$ tiup cluster upgrade tidb-test v6.5.0
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster upgrade tidb-test v6.5.0
Before the upgrade, it is recommended to read the upgrade guide at https://docs.pingcap.com/tidb/stable/upgrade-tidb-using-tiup and finish the preparation steps.
This operation will upgrade tidb v6.1.6 cluster tidb-test to v6.5.0.
Do you want to continue? [y/N]:(default=N) y
Upgrading cluster...
+ [ Serial ] - SSHKeySet: privateKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa, publicKey=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/ssh/id_rsa.pub
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.128
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.130
+ [Parallel] - UserSSH: user=tidb, host=192.168.93.129
+ [ Serial ] - Download: component=cdc, version=v6.5.0, os=linux, arch=amd64
+ [ Serial ] - Download: component=tiflash, version=v6.5.0, os=linux, arch=amd64
+ [ Serial ] - Download: component=pd, version=v6.5.0, os=linux, arch=amd64
+ [ Serial ] - Download: component=tikv, version=v6.5.0, os=linux, arch=amd64
+ [ Serial ] - Download: component=tidb, version=v6.5.0, os=linux, arch=amd64
+ [ Serial ] - Download: component=prometheus, version=v6.5.0, os=linux, arch=amd64
+ [ Serial ] - Download: component=grafana, version=v6.5.0, os=linux, arch=amd64
+ [ Serial ] - Download: component=alertmanager, version=, os=linux, arch=amd64
+ [ Serial ] - Mkdir: host=192.168.93.130, directories='/tidb-data/pd-2379'
+ [ Serial ] - Mkdir: host=192.168.93.129, directories='/tidb-data/tiflash-9000'
+ [ Serial ] - Mkdir: host=192.168.93.130, directories='/tidb-data/tiflash-9000'
+ [ Serial ] - Mkdir: host=192.168.93.128, directories='/tidb-data/pd-2379'
+ [ Serial ] - Mkdir: host=192.168.93.129, directories='/tidb-data/pd-2379'
+ [ Serial ] - BackupComponent: component=pd, currentVersion=v6.1.6, remote=192.168.93.130:/tidb-deploy/pd-2379
+ [ Serial ] - BackupComponent: component=tiflash, currentVersion=v6.1.6, remote=192.168.93.130:/tidb-deploy/tiflash-9000
+ [ Serial ] - BackupComponent: component=pd, currentVersion=v6.1.6, remote=192.168.93.129:/tidb-deploy/pd-2379
+ [ Serial ] - BackupComponent: component=tiflash, currentVersion=v6.1.6, remote=192.168.93.129:/tidb-deploy/tiflash-9000
+ [ Serial ] - BackupComponent: component=pd, currentVersion=v6.1.6, remote=192.168.93.128:/tidb-deploy/pd-2379
+ [ Serial ] - CopyComponent: component=pd, version=v6.5.0, remote=192.168.93.128:/tidb-deploy/pd-2379 os=linux, arch=amd64
+ [ Serial ] - CopyComponent: component=pd, version=v6.5.0, remote=192.168.93.130:/tidb-deploy/pd-2379 os=linux, arch=amd64
+ [ Serial ] - CopyComponent: component=pd, version=v6.5.0, remote=192.168.93.129:/tidb-deploy/pd-2379 os=linux, arch=amd64
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.128, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/pd-2379.service, deploy_dir=/tidb-deploy/pd-2379, data_dir=[/tidb-data/pd-2379], log_dir=/tidb-deploy/pd-2379/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - Mkdir: host=192.168.93.129, directories='/tidb-data/tikv-20160'
+ [ Serial ] - BackupComponent: component=tikv, currentVersion=v6.1.6, remote=192.168.93.129:/tidb-deploy/tikv-20160
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.130, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/pd-2379.service, deploy_dir=/tidb-deploy/pd-2379, data_dir=[/tidb-data/pd-2379], log_dir=/tidb-deploy/pd-2379/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - CopyComponent: component=tiflash, version=v6.5.0, remote=192.168.93.129:/tidb-deploy/tiflash-9000 os=linux, arch=amd64
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.129, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/pd-2379.service, deploy_dir=/tidb-deploy/pd-2379, data_dir=[/tidb-data/pd-2379], log_dir=/tidb-deploy/pd-2379/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - Mkdir: host=192.168.93.130, directories='/tidb-data/tikv-20160'
+ [ Serial ] - BackupComponent: component=tikv, currentVersion=v6.1.6, remote=192.168.93.130:/tidb-deploy/tikv-20160
+ [ Serial ] - CopyComponent: component=tiflash, version=v6.5.0, remote=192.168.93.130:/tidb-deploy/tiflash-9000 os=linux, arch=amd64
+ [ Serial ] - CopyComponent: component=tikv, version=v6.5.0, remote=192.168.93.130:/tidb-deploy/tikv-20160 os=linux, arch=amd64
+ [ Serial ] - CopyComponent: component=tikv, version=v6.5.0, remote=192.168.93.129:/tidb-deploy/tikv-20160 os=linux, arch=amd64
+ [ Serial ] - Mkdir: host=192.168.93.128, directories='/tidb-data/tikv-20160'
+ [ Serial ] - BackupComponent: component=tikv, currentVersion=v6.1.6, remote=192.168.93.128:/tidb-deploy/tikv-20160
+ [ Serial ] - CopyComponent: component=tikv, version=v6.5.0, remote=192.168.93.128:/tidb-deploy/tikv-20160 os=linux, arch=amd64
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.129, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/tiflash-9000.service, deploy_dir=/tidb-deploy/tiflash-9000, data_dir=[/tidb-data/tiflash-9000], log_dir=/tidb-deploy/tiflash-9000/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.130, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/tiflash-9000.service, deploy_dir=/tidb-deploy/tiflash-9000, data_dir=[/tidb-data/tiflash-9000], log_dir=/tidb-deploy/tiflash-9000/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.130, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/tikv-20160.service, deploy_dir=/tidb-deploy/tikv-20160, data_dir=[/tidb-data/tikv-20160], log_dir=/tidb-deploy/tikv-20160/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.129, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/tikv-20160.service, deploy_dir=/tidb-deploy/tikv-20160, data_dir=[/tidb-data/tikv-20160], log_dir=/tidb-deploy/tikv-20160/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - Mkdir: host=192.168.93.128, directories=''
+ [ Serial ] - BackupComponent: component=tidb, currentVersion=v6.1.6, remote=192.168.93.128:/tidb-deploy/tidb-4000
+ [ Serial ] - Mkdir: host=192.168.93.128, directories='/tidb-data/cdc-8300'
+ [ Serial ] - Mkdir: host=192.168.93.128, directories='/tidb-data/prometheus-9090'
+ [ Serial ] - Mkdir: host=192.168.93.128, directories=''
+ [ Serial ] - BackupComponent: component=grafana, currentVersion=v6.1.6, remote=192.168.93.128:/tidb-deploy/grafana-3000
+ [ Serial ] - BackupComponent: component=cdc, currentVersion=v6.1.6, remote=192.168.93.128:/tidb-deploy/cdc-8300
+ [ Serial ] - BackupComponent: component=prometheus, currentVersion=v6.1.6, remote=192.168.93.128:/tidb-deploy/prometheus-9090
+ [ Serial ] - CopyComponent: component=tidb, version=v6.5.0, remote=192.168.93.128:/tidb-deploy/tidb-4000 os=linux, arch=amd64
+ [ Serial ] - CopyComponent: component=prometheus, version=v6.5.0, remote=192.168.93.128:/tidb-deploy/prometheus-9090 os=linux, arch=amd64
+ [ Serial ] - CopyComponent: component=cdc, version=v6.5.0, remote=192.168.93.128:/tidb-deploy/cdc-8300 os=linux, arch=amd64
+ [ Serial ] - CopyComponent: component=grafana, version=v6.5.0, remote=192.168.93.128:/tidb-deploy/grafana-3000 os=linux, arch=amd64
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.128, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/tikv-20160.service, deploy_dir=/tidb-deploy/tikv-20160, data_dir=[/tidb-data/tikv-20160], log_dir=/tidb-deploy/tikv-20160/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.128, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/tidb-4000.service, deploy_dir=/tidb-deploy/tidb-4000, data_dir=[], log_dir=/tidb-deploy/tidb-4000/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - Mkdir: host=192.168.93.128, directories='/tidb-data/alertmanager-9093'
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.128, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/cdc-8300.service, deploy_dir=/tidb-deploy/cdc-8300, data_dir=[/tidb-data/cdc-8300], log_dir=/tidb-deploy/cdc-8300/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - BackupComponent: component=alertmanager, currentVersion=v6.1.6, remote=192.168.93.128:/tidb-deploy/alertmanager-9093
+ [ Serial ] - CopyComponent: component=alertmanager, version=, remote=192.168.93.128:/tidb-deploy/alertmanager-9093 os=linux, arch=amd64
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.128, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/grafana-3000.service, deploy_dir=/tidb-deploy/grafana-3000, data_dir=[], log_dir=/tidb-deploy/grafana-3000/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.128, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/prometheus-9090.service, deploy_dir=/tidb-deploy/prometheus-9090, data_dir=[/tidb-data/prometheus-9090], log_dir=/tidb-deploy/prometheus-9090/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - InitConfig: cluster=tidb-test, user=tidb, host=192.168.93.128, path=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache/alertmanager-9093.service, deploy_dir=/tidb-deploy/alertmanager-9093, data_dir=[/tidb-data/alertmanager-9093], log_dir=/tidb-deploy/alertmanager-9093/log, cache_dir=/home/tidb/.tiup/storage/cluster/clusters/tidb-test/config-cache
+ [ Serial ] - UpgradeCluster
Upgrading component tiflash
	Restarting instance 192.168.93.129:9000
	Restart instance 192.168.93.129:9000 success
	Restarting instance 192.168.93.130:9000
	Restart instance 192.168.93.130:9000 success
Upgrading component pd
	Restarting instance 192.168.93.128:2379
	Restart instance 192.168.93.128:2379 success
	Restarting instance 192.168.93.130:2379
	Restart instance 192.168.93.130:2379 success
	Restarting instance 192.168.93.129:2379
	Restart instance 192.168.93.129:2379 success
Upgrading component tikv
	Evicting 1 leaders from store 192.168.93.129:20160...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	  Still waitting for 1 store leaders to transfer...
	Restarting instance 192.168.93.129:20160
	Restart instance 192.168.93.129:20160 success
	Restarting instance 192.168.93.130:20160
	Restart instance 192.168.93.130:20160 success
	Evicting 7 leaders from store 192.168.93.128:20160...
	  Still waitting for 7 store leaders to transfer...
	  Still waitting for 7 store leaders to transfer...
	  Still waitting for 7 store leaders to transfer...
	Restarting instance 192.168.93.128:20160
	Restart instance 192.168.93.128:20160 success
Upgrading component tidb
	Restarting instance 192.168.93.128:4000
	Restart instance 192.168.93.128:4000 success
Upgrading component cdc
	Restarting instance 192.168.93.128:8300
	Restart instance 192.168.93.128:8300 success
Upgrading component prometheus
	Restarting instance 192.168.93.128:9090
	Restart instance 192.168.93.128:9090 success
Upgrading component grafana
	Restarting instance 192.168.93.128:3000
	Restart instance 192.168.93.128:3000 success
Upgrading component alertmanager
	Restarting instance 192.168.93.128:9093
	Restart instance 192.168.93.128:9093 success
Stopping component node_exporter
	Stopping instance 192.168.93.130
	Stopping instance 192.168.93.128
	Stopping instance 192.168.93.129
	Stop 192.168.93.129 success
	Stop 192.168.93.130 success
	Stop 192.168.93.128 success
Stopping component blackbox_exporter
	Stopping instance 192.168.93.130
	Stopping instance 192.168.93.128
	Stopping instance 192.168.93.129
	Stop 192.168.93.130 success
	Stop 192.168.93.129 success
	Stop 192.168.93.128 success
Starting component node_exporter
	Starting instance 192.168.93.130
	Starting instance 192.168.93.128
	Starting instance 192.168.93.129
	Start 192.168.93.129 success
	Start 192.168.93.130 success
	Start 192.168.93.128 success
Starting component blackbox_exporter
	Starting instance 192.168.93.130
	Starting instance 192.168.93.128
	Starting instance 192.168.93.129
	Start 192.168.93.129 success
	Start 192.168.93.130 success
	Start 192.168.93.128 success
Upgraded cluster `tidb-test` successfully
```

升级成功后，查看集群状态

```shell
[tidb@BigData01 ~]$ tiup cluster display tidb-test
tiup is checking updates for component cluster ...
Starting component `cluster`: /home/tidb/.tiup/components/cluster/v1.12.2/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v6.5.0
Deploy user:        tidb
SSH type:           builtin
Dashboard URL:      http://192.168.93.128:2379/dashboard
Grafana URL:        http://192.168.93.128:3000
ID                    Role          Host            Ports                            OS/Arch       Status  Data Dir                      Deploy Dir
--                    ----          ----            -----                            -------       ------  --------                      ----------
192.168.93.128:9093   alertmanager  192.168.93.128  9093/9094                        linux/x86_64  Up      /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
192.168.93.128:8300   cdc           192.168.93.128  8300                             linux/x86_64  Up      /tidb-data/cdc-8300           /tidb-deploy/cdc-8300
192.168.93.128:3000   grafana       192.168.93.128  3000                             linux/x86_64  Up      -                             /tidb-deploy/grafana-3000
192.168.93.128:2379   pd            192.168.93.128  2379/2380                        linux/x86_64  Up|UI   /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.129:2379   pd            192.168.93.129  2379/2380                        linux/x86_64  Up      /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.130:2379   pd            192.168.93.130  2379/2380                        linux/x86_64  Up|L    /tidb-data/pd-2379            /tidb-deploy/pd-2379
192.168.93.128:9090   prometheus    192.168.93.128  9090/12020                       linux/x86_64  Up      /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
192.168.93.128:4000   tidb          192.168.93.128  4000/10080                       linux/x86_64  Up      -                             /tidb-deploy/tidb-4000
192.168.93.129:9000   tiflash       192.168.93.129  9000/8123/3930/20170/20292/8234  linux/x86_64  Up      /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.130:9000   tiflash       192.168.93.130  9000/8123/3930/20170/20292/8234  linux/x86_64  Up      /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
192.168.93.128:20160  tikv          192.168.93.128  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.129:20160  tikv          192.168.93.129  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
192.168.93.130:20160  tikv          192.168.93.130  20160/20180                      linux/x86_64  Up      /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 13
```

> 可以发现TiDB集群版本已经从 V6.1.6 升级为 V6.5.0了；



