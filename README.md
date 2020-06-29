# k8s-kafka-flink
This is a one stop shop on how to set up Apache Flink with a Apache Kafka connector in Kubernetes. The goal with this tutorial is to push an event to Kafka, process it in Flink and push the processed event back to Kafka on a separate topic.

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
kubectl apply -f strimzi.yml --namespace kafka
```

Wait for the `strimzi-cluster-operator` to start (`STATUS: Running`):
```
kubectl get pods --namespace kafka -w
```

Now Strimzi should be installed onto the cluster. Next we will provision the Kafka cluster.

#### Provision the Kafka cluster

Apply the `kafka-persistent-single.yml`:
```
kubectl apply -f kafka-persistent-single.yml --namespace kafka
```

Wait for everything to startup, it might take a few minutes:
```
kubectl get pods --namespace kafka -w 
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

It is also possible to just call `kubectl get po -n kafka` and manually get the 

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


## Deploy Apache Flink to Kubernetes

No fancy operator is used to manage Flink. Instead we are just deploying a simple Flink yml. If you are interested in Flink, you can read more about it at the [Apache Flink homepage](https://ci.apache.org/projects/flink/flink-docs-release-1.10/).

Again, browse to the `k8s` directory of the repo.

Create the Flink namespace:
```
kubectl create ns flink
```

Deploy the `flink.yml` to the Kubernetes cluster:
```
kubectl apply -f flink.yml -n flink
```

Wait until Flink boots properly:
```
kubectl get pods --namespace flink -w
```

Now Flink should be running.

### Verify Flink setup

A `NodePort` is again used to expose the Flink UI. To get the port call:
```
kubectl get service flink-jobmanager-rest -o=jsonpath='{.spec.ports[0].nodePort}{"\n"}'\n -n flink
```

Using this port, you should be ablet to reach the Flink UI. Head into your browser and put `<node-ip>:<flink-port>` in your address field.

## Deploy job to Flink

The job that will be deployed to Flink is a simple example Flink application. What it does is to add a prefix to to the event that is consumed.

For the job in this project the [Project Template for Scala](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/projectsetup/scala_api_quickstart.html) was used. I had to do some minor modifications to comply with my local SBT and Scala setup. You will have to install both SBT and Scala. These are the versions that are used in this project:
- SBT version 1.3.12
- Scala version 2.12.11

Head over to the `flink-job` directory in one of your terminals.
Then build a JAR file, simply run:
```
sbt assembly
```

When the assembly is complete you should have a fresh jar in `target/scala-2.12/flink-job-assembly-0.1-SNAPSHOT.jar`.

Next step is to submit the job to Flink. You can either do this throug the Flink UI through the "Submit New Job" menu option. But I will do it using the Flink CLI.
For this tutorial download the "Apache Flink 1.10.1 for Scala 2.12" from [here](https://flink.apache.org/downloads.html).

Unzip the package:
```
tar xzf flink-1.10.1-bin-scala_2.12.tgz
cd flink-1.10.1
```

Get the Flink kubernetes node port:
```
kubectl get service flink-jobmanager-rest -o=jsonpath='{.spec.ports[0].nodePort}{"\n"}'\n -n flink
```

Upload the flink-job jar:
```
# set the <node-ip> and <flink-port>
# set <path-to-repo> to the k8s-kafka-flink repo
bin/flink run -m <node-ip>:<flink-port> \
    --class dev.chrisp.Job \ 
    <path-to-repo>/k8s-kafka-flink/flink-job/target/scala-2.12/flink-job-assembly-0.1-SNAPSHOT.jar \ 
    --input-topic input \
    --output-topic output \
    --bootstrap.servers  my-cluster-kafka-bootstrap.kafka:9092 \
    --zookeeper.connect my-cluster-zookeeper-client.kafka:2181 \
    --group.id flink
```

Basically the arguments to the Flink job are pretty self-descriptive.

Head over to the Flink UI and list "Running Jobs". You should see a task in "Running" state. If you got this far you should be ready to process data!

## Generate some data

Same thing as for the Kafka validation, open two terminal windows and browse to you Kafka install directory.
**Note** that the topic names are changed. Now `input` is used for producing, and `output` for consuming.

In terminal 1, we will consume messages:
```
# set the <node-ip> and <bootstrap-port>
bin/kafka-console-consumer.sh --bootstrap-server <node-ip>:<bootstrap-port> --topic output --from-beginning
```

In terminal 2, we produce messages:
```
# set the <node-ip> and <bootstrap-port>
bin/kafka-console-producer.sh --broker-list <node-ip>:<bootstrap-port> --topic input
```

When you produce message (just type anything into the Kafka producer prompt) you will see that event is pushed to the output topic with an additional prefix.
## Troubleshooting

In Kubernetes you can look at the logs for any pod:

```
# get the pods name (use namespace kafka or flink)
kubectl get pods --namespace kafka

# get logs
kubectl get logs <pod-name> --namespace kafka
```

Flink logs are also available through the UI. Browse to the "Task Managers" or "Job Manager" and click the "Logs" tab.