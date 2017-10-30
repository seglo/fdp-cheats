## Hosts

ZooKeeper Exhibitor: http://master.mesos/exhibitor/
ZooKeeper Host: `master.mesos:2181` (each DC/OS master has a ZK node)

## Marathon

Marathon runs as a service on DC/OS master nodes.

Get logs

`journalctl -u dcos-marathon -f | less`

Check for failed resource offers:

`journalctl -u dcos-marathon | grep resource`

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
dcos --debug --log-level=debug task exec data-0-node sh -c "export JAVA_HOME=\$(realpath \$MESOS_SANDBOX/jre1.8*) ; \$MESOS_SANDBOX/hadoop-*/bin/hadoop fs -ls hdfs://hdfs/""
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

### `pkill`

Kill processes that contain `kafka-executor.jar` in command: `sudo pkill -u root -f kafka-executor.jar`

### Looping over DC/OS nodes

Template to loop over all nodes in DC/OS cluster and ssh

```
readarray -t nodes < <(dcos node --json | jq -r ".[].hostname" )
for ip in ${nodes[@]}; do
  ssh ubuntu@$ip "echo hello from $ip"
done
```

Kill Kafka java processes on specified machines

```
for x in 13.58.5.25 13.58.2.124; do
  # kill kafka brokers
  ssh -i ~/.ssh/fdp-seglo-us-east-2.pem ubuntu@$x "sudo pkill -u nobody -f kafka_2.11-0.11.0.0"
  sleep 10
done
```

Kill confluent-kafka java processes on specified machines

```
for x in 10.8.0.18 10.8.0.12 10.8.0.21 10.8.0.20 10.8.0.16 10.8.0.25 10.8.0.24; do
  # kill kafka brokers
  ssh ubuntu@$x "sudo pkill -u root -f kafka-executor.jar"
  sleep 10
done
```
Add nodes to known_hosts
```
readarray -t nodes < <(dcos node --json | jq -r ".[].hostname" )
for ip in ${nodes[@]}; do
  ssh-keyscan -H $ip >> ~/.ssh/known_hosts
done
```

Add my public key to all ubuntu authorized_keys

```
readarray -t nodes < <(dcos node --json | jq -r ".[].hostname" )
for ip in ${nodes[@]}; do
  ssh ubuntu@$ip "echo ssh-rsa AAAAB3NzaC1yc2EAAAABJQAAAQEAlN1YpqoIyvrrv2p+ngx39n7EcSk7LpQ/jlSqTb21A5LAwOOQZGB1KPtEXCiek262eMuzpLRqrxBjOP4yqrJxpn9Rj4ZrXdGKu/ddxnnjQHu3/LpMUtr2v4LEgMsWzQWTEnPkIMTZbupVCv/+osJH9Grs4soxm5pXa/6+FClbldWIQoFfli9BaQ//0T2GXg15pXFCprMcEzw+PrtguN5mffFgTtS3G2SsEvjQZPiCu7K0yv4Cxzhu1VqgGZV4QNuUkUBPWIGnxEXvqESXErqcOpLSYslaDvYa6Sn720VRD0fM0LqL/2KSxSx4STXujk9Wzg20VQ3QA5cIIugUiwvBzw== seglo@randonom.com | tee -a /home/ubuntu/.ssh/authorized_keys"
  sleep 1
done
```
