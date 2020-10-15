# Java SCORE Examples

This repository contains SCORE (Smart Contract for ICON) examples written in Java.

## Requirements

You need to install JDK 11 or later version. Visit [OpenJDK.net](http://openjdk.java.net/) for prebuilt binaries.
Or you can install a proper OpenJDK package from your OS vendors.

In macOS:
```
$ brew tap AdoptOpenJDK/openjdk
$ brew cask install adoptopenjdk11
```

In Linux (Ubuntu 18.04):
```
$ sudo apt install openjdk-11-jdk
```

## How to Run

### 1. Build the project

```
$ ./gradlew build
```
The compiled jar bundle will be generated at `./hello-world/build/libs/hello-world-0.1.0.jar`.

### 2. Optimize the jar

You need to optimize your jar bundle before you deploy it to local or ICON networks.
This involves some pre-processing to ensure the actual deployment successful.

`gradle-javaee-plugin` is a Gradle plugin to automate the process of generating the optimized jar bundle.
Run the `optimizedJar` task to generate the optimized jar bundle.

```
$ ./gradlew optimizedJar
```
The output jar will be located at `./hello-world/build/libs/hello-world-0.1.0-optimized.jar`.

### 3. Deploy the optimized jar

#### Using `goloop` CLI command

Now you can deploy the optimized jar to ICON networks that support the Java SCORE execution environment.
Assuming you are running a local network that is listening on port 9082 for incoming requests,
you can create a deploy transaction with the optimized jar and deploy it to the local network as follows.

```
$ goloop rpc sendtx deploy ./hello-world/build/libs/hello-world-0.1.0-optimized.jar \
    --uri http://localhost:9082/api/v3 \
    --key_store <your_wallet_json> --key_password <password> \
    --nid 3 --step_limit=1000000 \
    --content_type application/java \
    --param name=Alice
```

**[Note]** The content type should be `application/java` instead of `application/zip` to differentiate it with the Python SCORE deployment.

#### Using `deployJar` extension

Starting with version `0.7.2` of `gradle-javaee-plugin`, you can also use the `deployJar` extension to specify all the information required for deployment.

```groovy
deployJar {
    endpoints {
        local {
            uri = 'http://localhost:9082/api/v3'
            nid = 3
        }
    }
    keystore = rootProject.hasProperty('keystoreName') ? "$keystoreName" : ''
    password = rootProject.hasProperty('keystorePass') ? "$keystorePass" : ''
    parameters {
        arg('name', 'Alice')
    }
}
```

Now you can run `deployToLocal` task as follows.

```
$ ./gradlew hello-world:deployToLocal -PkeystoreName=<your_wallet_json> -PkeystorePass=<password>

> Task :hello-world:deployToLocal
>>> deploy to http://localhost:9082/api/v3
>>> optimizedJar = ./hello-world/build/libs/hello-world-0.1.0-optimized.jar
>>> keystore = <your_wallet_json>
Succeeded to deploy: 0x699534c9f5277539e1b572420819141c7cf3e52a6904a34b2a2cdb05b95ab0a3
SCORE address: cxd6d044b01db068cded47bde12ed4f15a6da9f1d8
```

**[Note]** If you want to deploy to Gangnam testnet, use the following configuration for the endpoint and run `deployToGangnam` task.
```groovy
deployJar {
    endpoints {
        gangnam {
            uri = 'https://gicon.net.solidwallet.io/api/v3'
            nid = 7
        }
        ...
    }
}
```

### 4. Verify the execution

Check the deployed SCORE address first using the `txresult` command.
```
$ goloop rpc txresult <tx_hash> --uri http://localhost:9082/api/v3
{
  ...
  "scoreAddress": "cxaa736426a9caed44c59520e94da2d64888d9241b",
  ...
}
```

Then you can invoke `getGreeting` method via the following `call` command.
```
$ goloop rpc call --to <score_address> --method getGreeting --uri http://localhost:9082/api/v3
"Hello Alice!"
```

## Java SCORE Structure

### Comparison to Python SCORE

| Name               | Python SCORE                 | Java SCORE                  |
|--------------------|------------------------------|-----------------------------|
| External decorator | `@external`                  | `@External`                 |
| - (readonly)       | `@external(readonly=True)`   | `@External(readonly=true)`  |
| Payable decorator  | `@payable`                   | `@Payable`                  |
| Eventlog decorator | `@eventlog`                  | `@EventLog`                 |
| - (indexed)        | `@eventlog(indexed=1)`       | `@EventLog(indexed=1)`      |
| fallback signature | `def fallback`               | `void fallback()`           |
| SCORE initialize   | override `on_install` method | define a public constructor |
| Default parameters | native language support      | `@Optional`                 |

**[NOTE]** All external Java methods must have a `public` modifier, and should be instance methods.

### How to invoke a external method of another SCORE

One SCORE can invoke an external method of another SCORE using the following APIs.

```java
// [package score.Context]
public static Object call(Address targetAddress, String method, Object... params);

public static Object call(BigInteger value,
                          Address targetAddress, String method, Object... params);
```

The following example is for calling `tokenFallback`.
```java
if (_to.isContract()) {
    Context.call(_to, "tokenFallback", _from, _value, dataBytes);
}
```

## References

* [SCORE API document](https://www.javadoc.io/doc/foundation.icon/javaee-api)
* [Gradle plugin for JavaEE](https://github.com/icon-project/gradle-javaee-plugin)
* [`goloop` CLI command reference](https://github.com/icon-project/goloop/blob/master/doc/goloop_cli.md)

## License

This project is available under the [Apache License, Version 2.0](LICENSE).
