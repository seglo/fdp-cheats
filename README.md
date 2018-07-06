## Hosts

ZooKeeper Exhibitor: http://master.mesos/exhibitor/
ZooKeeper Host: `master.mesos:2181` (each DC/OS master has a ZK node)

## Marathon

Marathon runs as a service on DC/OS master nodes.

Get logs

`journalctl -u dcos-marathon -f | less`

Check for failed resource offers:

`journalctl -u dcos-marathon | grep resource`

## Mesos

Show `unreserved_resources` of all agents to check what's available.

`curl -H "Authorization: token=$(dcos config show core.dcos_acs_token)" https://leader.mesos/mesos/state.json  --insecure | jq '.slaves | .[] | .hostname ,.unreserved_resources'`

Show `reserved_resources_full` all reservations for all agents

`curl -H "Authorization: token=$(dcos config show core.dcos_acs_token)" https://leader.mesos/mesos/slaves/state.json --insecure | jq`

## DC/OS diagnostics

dcos-3dt.service is the diagnostics utility for DC/OS systemd components. This service runs on every host, tracking the internal state of the systemd unit. The service runs in two modes, with or without the -pull argument. If running on a master host, it executes /opt/mesosphere/bin/3dt -pull which queries Mesos-DNS for a list of known masters in the cluster, then queries a master (usually itself) :5050/statesummary and gets a list of agents.

From this complete list of cluster hosts, it queries all 3DT health endpoints :1050/system/health/v1/health. This endpoint returns health state for the DC/OS systemd units on that host. The master 3DT processes, along with doing this aggregation also expose /system/health/v1/ endpoints to feed this data by unit or node IP to the DC/OS user interface.

To create a DC/OS diagnostic bundle run these commands from the DC/OS CLI (DC/OS 1.8+),

```
dcos node diagnostics create all
dcos node diagnostics --status
(wait for progress to reach 100%)
dcos node diagnostics download <bundle_name>
```
## DC/OS System Services

System services are installed in `/opt/mesosphere`

See systemd scripts here `/etc/systemd/system`

Configuration is in `/opt/mesosphere/etc`

To view logs for a service use journalctl i.e.) `journalctl -u dcos-mesos-slave -f | less`

## DC/OS Service Manual Cleanup

Use `mesosphere/janitor` docker to manually cleanup mesos role's, principal's, and zookeeper namespaces.

1. Kill the secondary scheduler of the service by removing it from marathon Ex) `dcos marathon app remove hdfs`
2. Kill any zombie tasks running for service first (see `pgrep` and `pkill`)
3. Use `mesosphere/janitor` docker to manually cleanup mesos role's, principal's, and zookeeper namespaces.  Run `mesosphere/janitor` on any docker host or VPN (auth token required for VPN)

```
docker run mesosphere/janitor /janitor.py -r sample-role -p sample-principal -z sample-zk --auth_token=<token>

Ex)

docker run mesosphere/janitor /janitor.py -r hdfs-role -p hdfs-principal -z dcos-service-hdfs
```

### mesos-dns

Show all registered entries

http://leader.mesos/mesos_dns/v1/enumerate

### DC/OS API's ref

https://dcos.io/docs/1.10/api/master-routes/

https://dcos.io/docs/1.10/api/agent-routes/

## Spark

Run spark job that logs to HDFS for history server

```
dcos spark run --submit-args="--conf spark.eventLog.enabled=true --conf spark.eventLog.dir=hdfs://hdfs/history --class org.apache.spark.examples.SparkPi https://downloads.mesosphere.com/spark/assets/spark-examples_2.11-2.0.1.jar 30"
```

## Kafka

Use Kafka CLI scripts on DC/OS
NOTE: Lookup Kafka directory on zookeeper using Exhibitor

`kafka-topics.sh --describe --zookeeper master.mesos:2181/dcos-service-confluent-kafka`

### Quick producer consumer tests

```
./kafka-topics.sh \
--zookeeper leader.mesos:2181/dcos-service-kafka \
--create \
--topic replicated-topic \
--partitions 6 \
--replication-factor 3

# get a broker address with `dcos kafka endpoints broker`
./kafka-producer-perf-test.sh \
--topic replicated-topic \
--num-records 6000 \
--record-size 100 \
--throughput 1000 \
--producer-props bootstrap.servers=10.0.0.203:1025

printf 'group.id=test-consumer-group\n' >> consumer.properties

./kafka-console-consumer.sh \
--consumer.config consumer.properties \
--from-beginning \
--topic replicated-topic \
--bootstrap-server 10.0.0.203:1025
```

## VPN

### Setup on Ubuntu

Ubuntu NetworkManager doesn't update `/etc/resolv.conf` with DHCP push params when openvpn connection made

Disabling NetworkManager's own dnsmasq.

Edit `/etc/NetworkManager/NetworkManager.conf`

     #dns=dnsmasq

and restart NetworkManager

    sudo restart network-manager

Source:
https://serverfault.com/questions/528773/networkmanager-is-not-changing-etc-resolv-conf-after-openvpn-dns-push

### Packages for Ubuntu OpenVPN support

https://help.ubuntu.com/community/NetworkManager

### Troubleshooting connection

Clues to failure (i.e. cert failures) can be seen in syslog.

Tail syslog on server and client: `sudo tail -f /var/log/syslog`

## DC/OS CLI

List nodes

`dcos node`

### `dcos node ssh`

Open shell to leader through public master

```
dcos node ssh --leader --proxy-ip=18.221.52.123 --option IdentityFile=~/.ssh/fdp-seglo-us-east-2.pem --user=ubuntu`
```

Open shell to mesos agent

```
dcos node ssh --private-ip=10.0.0.13 --proxy-ip=18.221.52.123 --option IdentityFile=~/.ssh/fdp-seglo-us-east-2.pem --user=ubuntu
```

### `dcos task exec`

List contents of HDFS at `/`

> *NOTE:* When executing from the local CLI you must escape the inplace `$` references so it's not escaped at local CLI.

```
dcos --debug --log-level=debug task exec data-0-node sh -c "export JAVA_HOME=\$(realpath \$MESOS_SANDBOX/jre1.8*) ; \$MESOS_SANDBOX/hadoop-*/bin/hadoop fs -ls hdfs://hdfs/"
```

Get an interactive bash shell within a running container, in this case an HDFS data node.  Get the task name using `dcos task`

```
dcos task exec --interactive --tty data-0-node bash
```

## SSH

### Kill a hung session

Kill a hanging ssh session: `~.`

https://unix.stackexchange.com/a/2920

### Jump hosts with private keys

Add key to local `ssh-agent` then use `-J` jumphost switch on `ssh`

```
seglo@slice  ~  ssh-add ~/.ssh/fdp-jenkins-us-east-2.pem
Identity added: /home/seglo/.ssh/fdp-jenkins-us-east-2.pem (/home/seglo/.ssh/fdp-jenkins-us-east-2.pem)
seglo@slice  ~  ssh -J centos@jumphost centos@some-other-host
```

### Propagate ssh-agent identities to hosts

Add key to `ssh-agent` with `ssh-add privkey`.  Use `-A` with ssh: `ssh -A host`

## Shell

### `jq`

https://jqplay.org/

### `du`

Top 10 dirs in CWD

`du -aBM 2>/dev/null | sort -nr | head -n 50 | more`

Size of dirs in CWD

`du -d 1 -h .`

### `pkill`|`pgrep`

Kill or grep process name processes that contain `kafka-executor.jar` in command: `sudo pkill -u root -f kafka-executor.jar`

Examples)

```
# Grep process id and full process name.  **Must match something returned by process name, it can get truncated because of the long class path**
pkill -u nobody -f kafka_2.11-1.0.0 -a

# Kill Kafka java processes
sudo pkill -u nobody -f kafka_2.11-1.0.0

# Kill confluent-kafka java processes
sudo pkill -u root -f kafka-executor.jar
```

### Looping over DC/OS nodes

Template to loop over all private agents in DC/OS cluster and ssh

```
readarray -t nodes < <(dcos node --json | jq -r '.[] | if .type == "agent" and .attributes.public_ip == null then .hostname else null end | select(length > 0)')
for ip in ${nodes[@]}; do
  echo $ip
  ssh-keygen -f ~/.ssh/known_hosts -R $ip
  ssh-keyscan -H $ip >> ~/.ssh/known_hosts
  ssh centos@$ip "echo hello from $ip"
done
```

#### jq strings for different node types
```
# leader master
.[] | if .type == "master (leader)" then .ip else null end | select(length > 0)

# all masters
.[] | if .type == "master (leader)" then .ip else null end , if .type == "master" then .ip else null end | select(length > 0)

# all agents
.[] | if .type == "agent" then .hostname else null end | select(length > 0)

# all private agents
.[] | if .type == "agent" and .attributes.public_ip == null then .hostname else null end | select(length > 0)

# all public agents
.[] | if .type == "agent" and .attributes.public_ip then .hostname else null end | select(length > 0)

# all nodes
.[] | if .type == "master (leader)" then .ip else null end , if .type == "master" then .ip else null end , if .type == "agent" then .hostname else null end | select(length > 0)
```

Add nodes to known_hosts
```
readarray -t nodes < <(dcos node --json | jq -r '.[] | if .hostname == null then .ip else .hostname end')
for ip in ${nodes[@]}; do
  echo $ip
  ssh-keygen -f ~/.ssh/known_hosts -R $ip
  ssh-keyscan -H $ip >> ~/.ssh/known_hosts
  ssh centos@$ip "echo foobar"
done
```

Add a public key to all authorized_keys on all DC/OS nodes

```
readarray -t nodes < <(dcos node --json | jq -r '.[] | if .type == "master (leader)" then .ip else null end , if .type == "master" then .ip else null end , if .type == "agent" then .hostname else null end | select(length > 0)')
for ip in ${nodes[@]}; do
  echo $ip
  ssh-keygen -f ~/.ssh/known_hosts -R $ip
  ssh-keyscan -H $ip >> ~/.ssh/known_hosts
  ssh centos@$ip "echo ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC8DUxFowJ+z/+53/Uz3DKQ1ciA7KnyNeRX1XJBDLxRugpoIy2Er9J+I38ReakxdJ4FvenLjBiTUMiSBHd9pV33x87z2t2f/Unaj2a+q1khxqFI3KTaEUE1ZqCCARE8zWQ7Y2jGR6n0GFIdlzCa2xYS/844b90Fls89PFJYJ3u7Yq+FyK+pTVjLyhNvSinbuZIwNEZApQITbPPAMx77XC9w+FqhHTITRtorEv7JjL81DHL4Qs31oK0KWoCFW5NwtlnSyR5+oNU/UQ9xEPBPHWvZ4c2OOO9PRxx0CeToCoGpCSnmanMuLtwcqG3lI/D1jpBI7BGLWMGmBvuqXywvFruZTzN5ynYHk59bciSe7FuxvEenQzXMCnePT0almJg3KytmVhzzLytFyehnxE2WGQATVB4Dc9BXtvGYUhLqm2Eo2TQ8hSfzs0b45L4j3uDfeFUkAGygWZdBtU5WA6XxSTCh5yUplRrTBMZKxbI/w9BwoQ1c7LvccTmouZE6pFQap/LECL1hm6HWFVHxkVNTISVKDTOdkvsqzcp8MTUCmGQSia/oRxpRNZZDXd6hatgVqnjYi/DSOxZ0JY4GtXo6RE2O+Ba6aYnnSYS3/csbCnt4KdW6GrxVgXu1561akll7kk2Vf0pMbubQgGlM7sCNnBLWpnsu7CRXRJPAoGnfellm2w== yury.gribkov@gmail.com | tee -a ~/.ssh/authorized_keys"
done
```
