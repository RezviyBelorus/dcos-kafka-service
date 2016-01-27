# Kafka Framework

Management of Kafka clusters in DCOS

## Development

See [CONTRIBUTING.md](CONTRIBUTING.md) for the development guide.

## Audience

This user guide is intended for administrators and developers who wish to run Kafka in their DCOS cluster. For working on the Kafka Framework itself, see the [development guide](CONTRIBUTING.md).

The reader should already be fairly knowledgeable of [common Kafka terminology](https://kafka.apache.org/documentation.html), ie should know what things like "Brokers", "Topics", and "Partitions" are. The reader should have access to a live DCOS cluster, and should have installed the latest version of [dcos-cli](https://github.com/mesosphere/dcos-cli) to interact with it.

## Installation and Customization

### Default initial configuration

To start a basic test cluster with three brokers, run the following command with dcos-cli:

``` bash
$ dcos package install kafka
```

By default, this will create a new Kafka cluster named `kafka0`. Adding more clusters beyond `kafka0` requires customizing the `framework-name` at install time, as described below (TODO verify).

Additional customization would be needed before this cluster will be suitable for production use, but it should be plenty for testing/development as-is. Running clusters be [re-configured in-place using Marathon](#user-content-changing-configuration-in-flight).

### Custom initial configuration

The defaults may be customized by creating a JSON file, then passing it to `dcos package install` using the `--options parameter`. For example:

Sample JSON options file (sample.json):

``` json
{
  "kafka": {
    "framework-name": "my-kafka",
    "kafka-brokers": "10",
    "pv": "true",
    "placement-strategy": "NODE"
  }
}
```

Installing with sample.json's customizations:
``` bash
$ dcos package install --options=sample.json kafka
```

See [Configuration Options](#user-content-configuration-options) for a list of available fields which can be customized via an options JSON file when the Kafka cluster is created.

### Changing configuration in flight <a id="changing-configuration-in-flight"></a>

Once the cluster is already up and running, it may be customized in-place. The Kafka Scheduler will be running as a Marathon process, and can be reconfigured by changing values within Marathon.

1. View your Marathon dashboard at `http://your.dcos.host/marathon`
2. In the list of `Applications`, click the name of the Kafka framework to be updated.
3. Within the Kafka instance details view, click the `Configuration` tab, then click the `Edit` button.
4. In the dialog that appears, expand the `Environment Variables` section and update any field(s) to their desired value(s). For example, to [increase the number of Brokers](#user-content-broker-count), edit the value for `BROKER_COUNT`. Do not edit the value for `FRAMEWORK_NAME`.
5. Click `Change and deploy configuration` to apply any changes and cleanly reload the Kafka Framework scheduler. The Kafka cluster itself will persist across the change.

See [Configuration Options](#user-content-configuration-options) for a list of available fields which can be customized via Marathon while the Kafka cluster is running.

## Configuration Options <a id="configuration-options"></a>

The following describes commonly used features of the Kafka framework and how to configure them. View the [default `config.json` in DCOS Universe](https://github.com/mesosphere/universe/tree/version-1.x/repo/packages/K/kafka) to see an enumeration of all possible options.

### Framework Name

The name of this Kafka instance in DCOS. This is the only option that cannot be changed once the Kafka cluster is started; it can only be configured via the `dcos-cli --options` flag when first creating the Kafka instance.

**In dcos-cli options.json**: `framework-name` = string (default: `kafka0`)
**In Marathon**: The Framework Name cannot be changed after the cluster has started.

### Broker Count <a id="broker-count"></a>

Configure the number of brokers running in a given Kafka cluster. On initial creation the default is three brokers.

**In dcos-cli options.json**: `kafka-brokers` = integer (default: `3`)
**In Marathon**: `BROKER_COUNT` = integer

### Enable Persistent Volumes

By default Kafka Brokers will use the sandbox available to Mesos Tasks for storing data. This storage goes away on Task failure. So if a Broker crashes the data on it is lost forever. This is fine for dev environments. In production environments Kafka should be deployed with the following option enabled:

**In dcos-cli options.json**: `pv` = `true` or `false` (default: `false`)
**In Marathon**: `BROKER_PV` = `TRUE` or `FALSE`

### Configure Broker Placement Strategy

`ANY` will allow brokers to be placed on any node with sufficient resources, while `NODE` will ensure that all brokers within a given Kafka cluster are never colocated on the same node. In the future, this may also prevent brokers across multiple Kafka clusters from colocating, but this isn't yet the case.

**In dcos-cli options.json**: `placement-strategy` = `ANY` or `NODE` (default: `ANY`)
**In Marathon**: `PLACEMENT_STRATEGY` = `ANY` or `NODE`

## Cluster Maintenance

For ongoing maintenance of the Kafka cluster itself, the Kafka Framework exposes an HTTP API whose structure is designed to roughly match the tools provided by the Kafka distribution, such as `bin/kafka-topics.sh`.

The examples provided here use the "[HTTPie](http://httpie.org/)" commandline HTTP Client utility. These examples assume a DCOS host of `dcos.host` and a Kafka framework named `kafka0` (the default initial framework name). Replace these with appropriate values as needed.

### Connection Information

Kafka comes with many useful tools of its own. They often require either Zookeeper connection information, or the list of Broker endpoints. This information can be retrieved in an easily consumable formation from the /connection endpoint as below.

```
$ http dcos.host/service/kafka0/connection -pbH
GET /service/kafka0/connection HTTP/1.1
[...]

{
    "brokers": [
        "ip-10-0-0-233.us-west-2.compute.internal:9092",
        "ip-10-0-0-233.us-west-2.compute.internal:9093",
        "ip-10-0-0-233.us-west-2.compute.internal:9094"
    ],
    "zookeeper": "master.mesos:2181/kafka0"
}
```

### Broker Operations

#### Add Broker

Increase the `BROKER_COUNT` value via Marathon. New brokers should start automatically.

#### Remove Broker

Broker removal is currently a manual process.

**TODO**: Specify steps for manually removing a broker

#### List All Brokers

``` bash
$ http dcos.host/service/kafka0/brokers -pbH
GET /service/kafka0/brokers HTTP/1.1
[...]

[
    "0",
    "1",
    "2"
]
```
#### View Broker Details

``` bash
$ http dcos.host/service/kafka0/brokers/0 -pbH
GET /service/kafka0/brokers/0 HTTP/1.1
[...]

{
    "endpoints": [
        "PLAINTEXT://worker12398.dcos.host:9092"
    ],
    "host": "worker12398.dcos.host",
    "jmx_port": -1,
    "port": 9092,
    "timestamp": "1453854226816",
    "version": 2
}
```

#### Restart All Brokers

``` bash
$ http PUT dcos.host/service/kafka0/brokers -pbH
PUT /service/kafka0/brokers HTTP/1.1
[...]

[
    "broker-1__759c9fc2-3890-4921-8db8-c87532b1a033",
    "broker-2__0b444104-e210-4b78-8d19-f8938b8761fd",
    "broker-0__9c426c50-1087-475c-aa36-cd00d24ccebb"
]
```

#### Restart Single Broker

``` bash
$ http PUT dcos.host/service/kafka0/brokers/0 -pbH
PUT /service/kafka0/brokers HTTP/1.1
[...]

[
    "broker-0__9c426c50-1087-475c-aa36-cd00d24ccebb"
]
```

### Topic Operations

These operations mirror what's available using `bin/kafka-topics.sh`.

#### List Topics

``` bash
$ http dcos.host/service/kafka0/topics -pbH
GET /service/kafka0/topics HTTP/1.1
[...]

[
    "topic1",
    "topic0"
]
```

#### Create Topic

``` bash
$ http POST dcos.host/service/kafka0/topics name==topic1 partitions==3 replication==3 -pbH
POST /service/kafka0/topics?replication=3&name=topic1&partitions=3 [...]

{
    "exit_code": 0,
    "stderr": "",
    "stdout": "Created topic \"topic1\".\n"
}
```

#### View Topic Details

``` bash
$ http dcos.host/service/kafka0/topics/topic1 -pbH
GET /service/kafka0/topics/topic1 HTTP/1.1
[...]

{
    "partitions": [
        {
            "0": {
                "controller_epoch": 1,
                "isr": [
                    0,
                    1,
                    2
                ],
                "leader": 0,
                "leader_epoch": 0,
                "version": 1
            }
        },
        {
            "1": {
                "controller_epoch": 1,
                "isr": [
                    1,
                    2,
                    0
                ],
                "leader": 1,
                "leader_epoch": 0,
                "version": 1
            }
        },
        {
            "2": {
                "controller_epoch": 1,
                "isr": [
                    2,
                    0,
                    1
                ],
                "leader": 2,
                "leader_epoch": 0,
                "version": 1
            }
        }
    ]
}
```

#### View Topic Offsets

``` bash
$ http dcos.host/service/kafka0/topics/topic1/offsets -pbH
GET /service/kafka0/topics/topic1/offsets HTTP/1.1
[...]

[
    {
        "2": "0"
    },
    {
        "1": "0"
    },
    {
        "0": "0"
    }
]
```

#### Alter Topic Partition Count

``` bash
$ http PUT dcos.host/service/kafka0/topics/topic1 operation==partitions partitions==4 -pbH
PUT /service/kafka0/topics/topic1?operation=partitions&partitions=4
[...]

{
    "exit_code": 0,
    "stderr": "",
    "stdout": "WARNING: If partitions are increased for a topic that has a key, the partition logic or ordering of the messages will be affected\nAdding partitions succeeded!\n"
}
```

#### Alter Topic Config Value

``` bash
$ http PUT dcos.host/service/kafka0/topics/topic1 operation==config key==foo value==bar
```

#### Delete/Unset Topic Config Value

``` bash
$ http PUT dcos.host/service/kafka0/topics/topic1 operation==deleteConfig key==foo
```

#### Run Producer Test on Topic

``` bash
$ http PUT dcos.host/service/kafka0/topics/topic1 operation==producer-test messages==10 -pbH
PUT /service/kafka0/topics/topic1?operation=producer-test&messages=10
[...]

{
    "exit_code": 0,
    "stderr": "",
    "stdout": "10 records sent, 70.422535 records/sec (0.07 MB/sec), 24.20 ms avg latency, 133.00 ms max latency, 13 ms 50th, 133 ms 95th, 133 ms 99th, 133 ms 99.9th.\n"
}
```

#### Delete Topic

``` bash
$ http DELETE dcos.host/service/kafka0/topics/topic1 -pbH
DELETE /service/kafka0/topics/topic1?operation=delete HTTP/1.1
[...]

{
    "exit_code": 0,
    "stderr": "",
    "stdout": "Topic topic1 is marked for deletion.\nNote: This will have no impact if delete.topic.enable is not set to true.\n"
}
```

## TODO API for ACL changes? (kafka-acls.sh)
