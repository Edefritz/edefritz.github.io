---
layout: post
title: Kafka, AVRO and TypeScript?
categories: [KafkaJS, JavaScript, AVRO, TypeScript, NodeJS, Kafka]
---

# Kafka, AVRO and TypeScript?

Kafka is a very popular message broker system and used in a lot of companies right now. However, I always assumed that most Kafka users are also using JVM based clients like Java, Scala and Kotlin.

Recently I was playing around with a project in TypeScript and since it would have been handy to dump the results directly into Kafka, I saw that there is also a Kafka Client for JS. And it even works with AVRO.

# Example
Here is a simple example for an AVRO producer and consumer:

Set up a new node project and install these two dependencies. The schema registry is required to work with AVRO schemas.

```bash
npm install kafkajs
npm install @kafkajs/confluent-schema-registry
```

Then create a new file. This example is in TypeScript but in JS it would work more or less in a similar way.
First import all the dependencies and configure all Kafka related settings.


```ts
import { Kafka } from "kafkajs";
import {
  SchemaRegistry,
  readAVSCAsync,
} from "@kafkajs/confluent-schema-registry";
import { exit } from "process";

const TOPIC = "my_topic";

// configure Kafka broker
const kafka = new Kafka({
  clientId: "some-client-id",
  brokers: ["localhost:29092"],
});

// If we use AVRO, we need to configure a Schema Registry
// which keeps track of the schema
const registry = new SchemaRegistry({
  host: "http://localhost:8085",
});

// create a producer which will be used for producing messages
const producer = kafka.producer();

const consumer = kafka.consumer({
  groupId: "group_id_1",
});

// declaring a TypeScript type for our message structure
declare type MyMessage = {
  id: string;
  value: number;
};
```
Now we need to make sure we can encode messages in AVRO. Therefore we need to be able to read a schema from a file an register it in the schema registry.

This is how the schema in this example will look like. Pretty straightforward, two fields called id which is a string and value which is an integer. 
Insert this to a file called schema.avsc, we will use the confluent-schema-registry package to read it and register the schema in the schema registry.

```avsc
{
  "name": "example",
  "type": "record",
  "namespace": "com.my.company",
  "doc": "Kafka JS example schema",
  "fields": [
    {
      "name": "id",
      "type": "string"
    },
    {
      "name": "value",
      "type": "int"
    }
  ]
}
```

Here is the function which we will use to read an avro schema from a file and register it in the schema registry.
```ts
// This will create an AVRO schema from an .avsc file
const registerSchema = async () => {
  try {
    const schema = await readAVSCAsync("./avro/schema.avsc");
    const { id } = await registry.register(schema);
    return id;
  } catch (e) {
    console.log(e);
  }
};
```

This is how we can build a producer. Before pushing a message (of type MyMessage which we defined above) we will encode it usind the AVRO schema from the registry.
```ts
// push the actual message to kafka
const produceToKafka = async (registryId: number, message: MyMessage) => {
  await producer.connect();

  // compose the message: the key is a string
  // the value will be encoded using the avro schema
  const outgoingMessage = {
    key: message.id,
    value: await registry.encode(registryId, message),
  };

  // send the message to the previously created topic
  await producer.send({
    topic: TOPIC,
    messages: [outgoingMessage],
  });

  // disconnect the producer
  await producer.disconnect();
};
```

Before we can produce a message, we need to create a topic. It also checks if the topic is already present in case you run this multiple times. You can skip this if the topic is already present.
```ts
// create the kafka topic where we are going to produce the data
const createTopic = async () => {
  try {
    const topicExists = (await kafka.admin().listTopics()).includes(TOPIC);
    if (!topicExists) {
      await kafka.admin().createTopics({
        topics: [
          {
            topic: TOPIC,
            numPartitions: 1,
            replicationFactor: 1,
          },
        ],
      });
    }
  } catch (error) {
    console.log(error);
  }
};
```

Now we create our producer and consumer functions which publish an example message and consume it again.
```ts
const produce = async () => {
  await createTopic();
  try {
    const registryId = await registerSchema();
    // push example message
    if (registryId) {
      const message: MyMessage = { id: "1", value: 1 };
      await produceToKafka(registryId, message);
      console.log(`Produced message to Kafka: ${JSON.stringify(message)}`);
    }
  } catch (error) {
    console.log(`There was an error producing the message: ${error}`);
  }
};

async function consume() {
  await consumer.connect();

  await consumer.subscribe({
    topic: TOPIC,
    fromBeginning: true,
  });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      if (message.value) {
        const value: MyMessage = await registry.decode(message.value);
        console.log(value);
      }
    },
  });
}
```

And finally we execute both functions one after another.
```ts
produce()
  .then(() => consume())
```

The console should print something like:
```
Produced message to Kafka: {"id":"1","value":1}
Consumed message from Kafka: Example { id: '1', value: 1 }
```

I created a [repository](https://github.com/Edefritz/kafkajs_avro_demo) to demo this example. There is a docker-compose file which takes care of setting up a Kafka Broker and a Schema Registry.