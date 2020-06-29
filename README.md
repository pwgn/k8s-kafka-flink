# k8s-kafka-flink
This is a one stop shop on how to set up Flink with a Kafka connector in Kubernetes.

1. Deploy Kafka and Flink to Kubernetes
2. Deploy job to Flink
3. Generate some data

## MicroK8s
In these examples [MicroK8s](https://microk8s.io/) has been used. Follow their [doc](https://microk8s.io/docs) to set it up.

Do not forget to enable some required extensions:

```
microk8s enable dns storage
```

When Kubernetes is setup locally you are good to go!

## Setup Apache Kafka

To run Kafka on Kubernetes [Strimzi](https://strimzi.io) is used in this setup. Strimzi makes it a lot simpler to run Kafka in a Kubernetes cluster. Strimzi provides some operators to manage Kafka and the related components. For the purpose of this guide the details are not too relevant, but if you are interested you can read more about Strimzi here:

- [Overview](https://strimzi.io/docs/operators/latest/overview.html)
- [Quick start guide](https://strimzi.io/docs/operators/latest/quickstart.html)

### Deploy Kafka to Kubernetes

This is done in two steps:
1. Install Strimzi
2. Provision the kafka cluster

First, move into the `k8s` dir:
```
cd k8s
```

#### Install Strimzi


Create Kafka namespace:
```
kubectl create ns kafka
```

Create Strimzi cluster operator:
```
kubectl apply -f strimzi.yml -n kafka
```

Wait for the `strimzi-cluster-operator` to start (`STATUS: Running`):
```
kubectl get po -n kafka -w
```

Now Strimzi should be installed onto the cluster. Next we will provision the Kafka cluster.

#### Provision the Kafka cluster

Apply the `kafka-persistent-single.yml`:
```
kubectl apply -f kafka-persistent-single.yml  -n kafka
```

Wait for everything to startup, it might take a few minutes:
```
kubectl get po -n kafka -w 
```

#### Verify the Kafka setup

For this particular experiment I wanted to explore how to connect to the Kafka cluster from the outside. To do that a `NodePort` was setup in the `kafka-persistent-single.yml`. Strimzi has a good blog post about [Accessing Kafka](https://strimzi.io/blog/2019/04/17/accessing-kafka-part-1/).

First, get your Kubernetes node `Name`:
```
kubectl get nodes
```

Next, get your node `InternalIP`:
```
# Replace <NodeName> with your node name
kubectl get node <NodeName> -o=jsonpath='{range .status.addresses[*]}{.type}{"\t"}{.address}{"\n"}'
```

Fetch the port of your Kafka external bootstrap service:
```
kubectl get service my-cluster-kafka-external-bootstrap -o=jsonpath='{.spec.ports[0].nodePort}{"\n"}'\n -n kafka
```

By now you should have:
- Your Kubernetes node ip adress
- The port of the Kafka bootstrap service

If you have the Kafka CLI installed locally already its good. Otherwise, [download it](https://kafka.apache.org/quickstart#quickstart_download), follow the download step only.

Finally, we can do the actual validation by producing / consuming some messages. Open two terminal windows, and browse to your kafka installation folder.

In terminal 1, we will consume messages:
```
# set the <node-ip> and <bootstrap-port>
bin/kafka-console-consumer.sh --bootstrap-server <node-ip>:<bootstrap-port> --topic my-topic --from-beginning
```

In terminal 2, we produce messages:
```
# set the <node-ip> and <bootstrap-port>
bin/kafka-console-producer.sh --broker-list <node-ip>:<bootstrap-port> --topic my-topic
```

Post some messages in terminal 2, and they should pop up in terminal 1. GG!


## Setup flink

## Deploy job to flink

## Generate some data
