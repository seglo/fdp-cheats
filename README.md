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

Show `unreserved_resources` of all hosts to check what's available.

`curl -H "Authorization: token=$(dcos config show core.dcos_acs_token)" https://leader.mesos/mesos/state.json  --insecure | jq '.slaves | .[] | .hostname ,.unreserved_resources'`

## DC/OS System Services

System services are installed in `/opt/mesosphere`

See systemd scripts here `/etc/systemd/system`

To view logs for a service use journalctl i.e.) `journalctl -u dcos-mesos-slave -f | less`

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

Kill a hanging ssh session: `~.`

https://unix.stackexchange.com/a/2920

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
done
```

Add my public key to all authorized_keys

```
readarray -t nodes < <(dcos node --json | jq -r '.[] | if .hostname == null then .ip else .hostname end')
for ip in ${nodes[@]}; do
  echo $ip
  ssh-keygen -f ~/.ssh/known_hosts -R $ip
  ssh-keyscan -H $ip >> ~/.ssh/known_hosts
  ssh centos@$ip "echo ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCRnp7mbIrrt/xj7vNo1U8nQXc3rNVrcXSxcW+QVE9ZHMHndZxcEGBEAxreq613DkywbwZ4oyBFVTBsnTL8qxRDI3lRkGeHTECmYD8X25EiwWIT6GDsp1EYdDW8pix1cYXiuFbGbwrNmyfSUNVnNUmdnS61cnlVc6ONWG0IXJMzzKkpT1o0EwxZEsxl4/vP1qD5ffAdx4ZDfXMKAHrSXPyNqMJ7rix3sRk5b4orBMzvwq5OclkoUBF6kP7eGtZzSSBiY/awB0O+63NliWQ6Kr204Xh4qZgBy4zGcEP3IebwJQH6mb4PWaO3fRN96nO3PWcv5Z3dGm/lfTtF9BQHpex4Kml2wJ5GQ0AfuSHbbqkzZ6eXqwMzaJB/Zf2RYV/9sNpYJrMP145Em4+kM1ZPA+/YN8morT999cdjr2rSilQ8ykdxaaThW4LibhCH24jyg+C0tvAq//VWwsZusNNossyeeoLSYV9zQJ4lSVLyM+a8YWAUDI7pUuWUXGVUk8Bh4V8tcDRM0Pd7WAGLoaeXlgLAy6cFJJKJqNtob+Gh+SdVj4zZDV5H5TKMqcK7tXnuwvbGbSWxLxWE0dwWPis1pdaVU8VRaVk+da151eSN2xjBojjZsAT+RXIcTw61s3YkwC85LpN4sfisu8rl4HL0Rgp5/uv2NJQAMADN0q0dg7WUlQ== gerard.maas@gmail.com | tee -a ~/.ssh/authorized_keys"
  sleep 1
done
```
