# Red Hat OpenShift Streams for Apache Kafka: Async API Pizza-Order Demo

## Setting up OpenShift Streams for Apache Kafka

1. Create the Kafka instance
```
rhoas kafka create --name samurai-pizza-kafkas --provider aws --region eu-west-1
```

2. Create a Service Account. Note that this saves the service account's client-id and client-secret in the `pizza-sa-secret.json` file on your local filesystem. Make sure to store the secret in a safe place.
```
rhoas service-account create --short-description=pizza-service-account --file-format json --output-file pizza-sa-secret.json --overwrite
```

3. Create a topic named `pizza-order`:
```
rhoas kafka topic create --name pizza-order --partitions 10
```
4. Set the ACLs to allow the service account to consume from the newly created topic:
```
rhoas kafka acl grant-access --consumer --service-account $USER --topic pizza-order --group pizza-order-subscriber
```

## Install AsyncAPI tooling

- `npm install -g @asyncapi/cli`
- `npm install -g @asyncapi/generator`


## Generating the Java consumer for "pizza-order" topic

```
ag rhosak-pizza-subscribe-with-schema.yaml @asyncapi/java-template -o generated-java --force-write -p server=production
```

NOTE: that this automatically picks up the security scheme and configures SASL_SSL & PLAIN

NOTE:Note that the source dir of this maven project has been set to the base-dir (which is a bit weird), and this seems to be a bit of a problem for VSCode ...

- Set the maven.compiler source and target version in the POM file to 11.
- Configure the username and password in the "env.json" file.
- Configure the "String id" variable in the PizzaOrderSubscriber constructor and set it to the group id "pizza-order-subscriber"
- Run: `mvn clean package -DskipTests`
- Start the application: `java -cp target/asyncapi-java-generator-0.1.0.jar com.ibm.mq.samples.jms.DemoSubscriber`

--------------- Generating the Spring Cloud Stream consumer for "pizza-order" topic ----------

```
ag rhosak-pizza-order-subscribe.yaml -o generated-spring-cloud-stream -p view=provider -p binder=kafka -p actuator=true -p artifactId=PizzaOrder -p groupId=com.redhat.rhosak -p javaPackage=com.redhat.rhosak.pizzaorder -p username=default -p password=default @asyncapi/java-spring-cloud-stream-template
```

- Add the following configuration to a new "application.properties" file in "src/main/resources"
```
spring.cloud.stream.kafka.binder.brokers={rhosak kafka instance bootstrap-server url}
spring.cloud.stream.kafka.binder.configuration.security.protocol=SASL_SSL
spring.cloud.stream.kafka.binder.configuration.sasl.mechanism=PLAIN
spring.cloud.stream.kafka.binder.configuration.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="{username}" \
  password="{password}";
```

- Add `"group: pizza-order-subscriber"` under `orderPizza-in-0` bindings in the "src/main/resources/application.yml" file.

- Run: `mvn clean package -DskipTests`
- Start the application: `java -jar ./target/PizzaOrder-1.0.0.jar`

------------------------------------------------------------------------------------------------
