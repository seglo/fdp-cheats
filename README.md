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

## DC/OS CLI

List nodes

`dcos node`

### `dcos node ssh`

Open shell to leader through public master

`dcos node ssh --leader --proxy-ip=18.221.52.123 --option IdentityFile=~/.ssh/fdp-seglo-us-east-2.pem --user=ubuntu`

Open shell to mesos agent

`dcos node ssh --private-ip=10.0.0.13 --proxy-ip=18.221.52.123 --option IdentityFile=~/.ssh/fdp-seglo-us-east-2.pem --user=ubuntu`

## SSH

Kill a hanging ssh session: `~.`

https://unix.stackexchange.com/a/2920

## Shell

`jq` - https://jqplay.org/

Kill confluent-kafka java processes on specified machines

```
for x in 10.8.0.18 10.8.0.12 10.8.0.21 10.8.0.20 10.8.0.16 10.8.0.25 10.8.0.24; do
  echo $x
  # kill kafka brokers
  ssh ubuntu@$x "sudo pkill -u root -f kafka-executor.jar"
  sleep 10
done
```

Add nodes to known_hosts

```
readarray -t nodes < <(dcos node --json | jq -r ".[].hostname" )
for ip in ${nodes[@]}; do
  echo $ip
  ssh-keyscan -H $ip >> ~/.ssh/known_hosts
done
```

Add my public key to all ubuntu authorized_keys

```
readarray -t nodes < <(dcos node --json | jq -r ".[].hostname" )
for ip in ${nodes[@]}; do
  echo $ip
  ssh ubuntu@$ip "echo ssh-rsa AAAAB3NzaC1yc2EAAAABJQAAAQEAlN1YpqoIyvrrv2p+ngx39n7EcSk7LpQ/jlSqTb21A5LAwOOQZGB1KPtEXCiek262eMuzpLRqrxBjOP4yqrJxpn9Rj4ZrXdGKu/ddxnnjQHu3/LpMUtr2v4LEgMsWzQWTEnPkIMTZbupVCv/+osJH9Grs4soxm5pXa/6+FClbldWIQoFfli9BaQ//0T2GXg15pXFCprMcEzw+PrtguN5mffFgTtS3G2SsEvjQZPiCu7K0yv4Cxzhu1VqgGZV4QNuUkUBPWIGnxEXvqESXErqcOpLSYslaDvYa6Sn720VRD0fM0LqL/2KSxSx4STXujk9Wzg20VQ3QA5cIIugUiwvBzw== seglo@randonom.com | tee -a /home/ubuntu/.ssh/authorized_keys"
  sleep 1
done
```
