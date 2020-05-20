# Camel K: Event Streaming Example

This demo uses several features from Camel K, Kafka and OpenShift to implement a system that handles different types of hazards and alert information.

## Before you begin

Make sure you check-out this repository from git and open it with [VSCode](https://code.visualstudio.com/).

Instructions are based on [VSCode Didact](https://github.com/redhat-developer/vscode-didact), so make sure it's installed
from the VSCode extensions marketplace.

From the VSCode UI, right-click on the `readme.didact.md` file and select "Didact: Start Didact tutorial from File". A new Didact tab will be opened in VS Code.

Make sure you've opened this readme file with Didact before jumping to the next section.

## Preparing the cluster

This example can be run on any OpenShift 4.3+ cluster or a local development instance (such as [CRC](https://github.com/code-ready/crc)). Ensure that you have a cluster available and login to it using the OpenShift `oc` command line tool.

You need to create a new project named `camel-k-event-streaming` for running this example. This can be done directly from the OpenShift web console or by executing the command `oc new-project camel-k-event-streaming` on a terminal window.

You need to install the Camel K operator in the `camel-k-event-streaming` project. To do so, go to the OpenShift 4.x web console, login with a cluster admin account and use the OperatorHub menu item on the left to find and install **"Red Hat Integration - Camel K"**. You will be given the option to install it globally on the cluster or on a specific namespace.
If using a specific namespace, make sure you select the `camel-k-event-streaming` project from the dropdown list.
This completes the installation of the Camel K operator (it may take a couple of minutes).

When the operator is installed, from the OpenShift Help menu ("?") at the top of the WebConsole, you can access the "Command Line Tools" page, where you can download the **"kamel"** CLI, that is required for running this example. The CLI must be installed in your system path.

Refer to the **"Red Hat Integration - Camel K"** documentation for a more detailed explanation of the installation steps for the operator and the CLI.

### Installing the AMQ Streams Operator

This example uses AMQ Streams, Red Hat's data streaming platform based on Apache Kafka.
We want to install it on a new project named `event-streaming-kafka-cluster`. 

You need to create the `event-streaming-kafka-cluster` project from the OpenShift web console or by executing the command `oc new-project event-streaming-kafka-cluster` on a terminal window. 

Now, we can go to the OpenShift 4.x WebConsole page, use the OperatorHub menu item on the left hand side menu and use it to find and install **"Red Hat Integration - AMQ Streams"**.
This will install the operator and may take a couple minutes to install.

### Installing the AMQ Broker Operator

The installation of the AMQ Broker follows the same isolation pattern as the AMQ Streams one. We will deploy it in a separate project and will
instruct the operator to deploy a broker according to the configuration.

You need to create the `event-streaming-messaging-broker` project from the OpenShift web console or by executing the command `oc new-project event-streaming-messaging-broker` on a terminal window. 

Now, we can go to the OpenShift 4.x WebConsole page, use the OperatorHub menu item on the left hand side menu and use it to find and install **"Red Hat Integration - AMQ Broker"**.
This will install the operator and may take a couple minutes to install.

### Installing OpenShift Serverless

This demo also needs OpenShift Serverless (Knative) installed and working.

Refer to the [OpenShift Serverless Documentation](https://docs.openshift.com/container-platform/4.3/serverless/installing_serverless/installing-openshift-serverless.html) for instructions on how to install it on your cluster.

## Requirements

<a href='didact://?commandId=vscode.didact.validateAllRequirements' title='Validate all requirements!'><button>Validate all Requirements at Once!</button></a>

**OpenShift CLI ("oc")**

The OpenShift CLI tool ("oc") will be used to interact with the OpenShift cluster.

[Check if the OpenShift CLI ("oc") is installed](didact://?commandId=vscode.didact.cliCommandSuccessful&text=oc-requirements-status$$oc%20help&completion=Checked%20oc%20tool%20availability "Tests to see if `oc help` returns a 0 return code"){.didact}

*Status: unknown*{#oc-requirements-status}

**Connection to an OpenShift cluster**

In order to execute this demo, you will need to have an OpenShift cluster with the correct access level, the ability to create projects and install operators as well as the Apache Camel K CLI installed on your local system.

[Check if you're connected to an OpenShift cluster](didact://?commandId=vscode.didact.requirementCheck&text=cluster-requirements-status$$oc%20get%20project$$NAME&completion=OpenShift%20is%20connected. "Tests to see if `kamel version` returns a result"){.didact}

*Status: unknown*{#cluster-requirements-status}

**Apache Camel K CLI ("kamel")**

Apart from the support provided by the VS Code extension, you also need the Apache Camel K CLI ("kamel") in order to
access all Camel K features.

[Check if the Apache Camel K CLI ("kamel") is installed](didact://?commandId=vscode.didact.requirementCheck&text=kamel-requirements-status$$kamel%20version$$Camel%20K%20Client&completion=Apache%20Camel%20K%20CLI%20is%20available%20on%20this%20system. "Tests to see if `kamel version` returns a result"){.didact}

*Status: unknown*{#kamel-requirements-status}

**Knative installed on the OpenShift cluster**

The cluster also needs to have Knative installed and working.

[Check if Knative is installed](didact://?commandId=vscode.didact.requirementCheck&text=kservice-project-check$$oc%20api-resources%20--api-group=operator.knative.dev$$knativeservings&completion=Verified%20Knative%20services%20installation. "Verifies if Knative is installed"){.didact}

*Status: unknown*{#kservice-project-check}

### Optional Requirements

The following requirements are optional. They don't prevent the execution of the demo, but may make it easier to follow.

**VS Code Extension Pack for Apache Camel**

The VS Code Extension Pack for Apache Camel by Red Hat provides a collection of useful tools for Apache Camel K developers,
such as code completion and integrated lifecycle management. They are **recommended** for the tutorial, but they are **not**
required.

You can install it from the VS Code Extensions marketplace.

[Check if the VS Code Extension Pack for Apache Camel by Red Hat is installed](didact://?commandId=vscode.didact.extensionRequirementCheck&text=extension-requirement-status$$redhat.apache-camel-extension-pack&completion=Camel%20extension%20pack%20is%20available%20on%20this%20system. "Checks the VS Code workspace to make sure the extension pack is installed"){.didact}

*Status: unknown*{#extension-requirement-status}


## Understanding the Demo Scenario

This demo simulates a global hazard alert system. The simulator consumes data from a public APIs available on the internet
as well as user-provided data that simulates different types of hazards (ie.: crime, viruses, natural hazards and others).
The system consumes data from OpenAQ API to query air pollution information consume, present information and warn the user when certain incidents happen.
![Diagram](docs/Overview.png)


## Understanding the Infrastructure
This demo spans over 3 namespaces in the cluster, one hosting the Streaming services, one hosting the broker, and a main namespace for all the system applications.

In the Streaming namespace, you will have 3 topics listening to the events (pollution, crime and timeline). In the Broker namespace, you will have two topics listening to the notification and alarm events. 

In the main namespace, this is where you run all the Camel K microservices and fuctions. 

![Diagram](docs/Diagram.png)

## 1. Creating the AMQ Streams Cluster

We switch to the `event-streaming-kafka-cluster` project to create the Kafka cluster:

```oc project event-streaming-kafka-cluster```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20project%20event-streaming-kafka-cluster&completion=Project%20changed. "Switched to the project that will run AMQ Streams "){.didact})

The next step is to use the operator to create an AMQ Streams cluster. This can be done with the command:

```oc create -f infra/kafka/clusters/event-streaming-kafka-cluster.yaml```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20create%20-f%20infra%2Fkafka%2Fclusters%2Fevent-streaming-kafka-cluster.yaml&completion=Created%20the%20AMQ%20Streams%20cluster. "Creates the AMQ Streams cluster"){.didact})

Depending on how large your OpenShift cluster is, this may take a little while to complete. Let's run this command and wait until the cluster is up and running.

```oc wait kafka/event-streaming-kafka-cluster --for=condition=Ready --timeout=600s```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20wait%20kafka%2Fevent-streaming-kafka-cluster%20--for=condition=Ready%20--timeout=600s&completion=Wait%20for%20the%20Kafka%20cluster. "Wait for the Kafka cluster"){.didact})

You can can check the state of the cluster by running the following command:


```oc get kafkas -n event-streaming-kafka-cluster event-streaming-kafka-cluster```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20get%20kafkas%20-n%20event-streaming-kafka-cluster%20event-streaming-kafka-cluster&completion=Check%20if%20the%20cluster%20was%20created. "Check if the cluster was created"){.didact})

Once the AMQ Streams cluster is created. We can proceed to the creation of the AMQ Streams topics:


```oc apply -f infra/kafka/clusters/topics/```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20apply%20-f%20infra%2Fkafka%2Fclusters%2Ftopics%2F&completion=Created%20topics. "Create topics"){.didact})

```oc get kafkatopics```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20get%20kafkatopics&completion=Collected%20Kafka%20topics. "Collect Kafka topics"){.didact})

At this point, if all goes well, we should our AMQ Streams cluster up and running with several topics.

## 2. Creating the AMQ Broker Cluster

To switch to the `event-streaming-messaging-broker` project, run the following command:

```oc project event-streaming-messaging-broker```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20project%20event-streaming-messaging-broker&completion=Switched%20to%20project%20for%20running%20the%20AMQ%20Broker. "Switched to the project that will run the AMQ Broker"){.didact})

Having already the operator installed and running on the project, we can proceed to create the broker instance:


```oc create -f infra/messaging/broker/instances/amq-broker-instance.yaml```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20create%20-f%20infra%2Fmessaging%2Fbroker%2Finstances%2Famq-broker-instance.yaml&completion=Created%20AMQ%20Broker%20instance. "Creates the AMQ Broker instance"){.didact})


We can use the `oc get activemqartermis` command to check if the AMQ Broker instance is created:

```oc get activemqartemises```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20get%20activemqartemises&completion=Verify%20the%20broker%20instances. "Verify the broker instances"){.didact})

If it was successfully created, then we can create the addresses and queues required for the demo to run:

```oc apply -f infra/messaging/broker/instances/addresses```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20apply%20-f%20infra%2Fmessaging%2Fbroker%2Finstances%2Faddresses&completion=Create%20the%20addresses. "Create the addresses"){.didact})


## 3. Deploying the Project

Now that the infrastructure is ready, we can go ahead and deploy the demo project. First, lets switch to the main project:

```oc project camel-k-event-streaming```
([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20project%20camel-k-event-streaming&completion=Switched%20to%20the%20demo%20project. "Switched to the demo project"){.didact})


```kamel install --skip-operator-setup --skip-cluster-setup --maven-repository https://maven.repository.redhat.com/ga```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kamel%20install%20--skip-operator-setup%20--skip-cluster-setup%20--maven-repository%20https://maven.repository.redhat.com/ga&completion=Switched%20to%20the%20demo%20project. "Switched to the demo project"){.didact})

### Initial Configuration

Most of the components of the demo use use the [./config/application.properties](didact://?commandId=vscode.open&projectFilePath=./config/application.properties&newWindow=false&completion=Ok. "Edit the secret configuration"){.didact} to read the configurations they need to run. This file already comes with
expected defaults, so no action should be needed.

#### Optional: Configuration Adjustments

*Note*: you can skip this step if you don't want to adjust the configuration

In case you need to adjust the configuration, the following 2 commands present information that will be required to configure the deployment:

```oc get services -n event-streaming-messaging-broker```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20get%20services%20-n%20event-streaming-messaging-broker&completion=Get%20the%20AMQ%20Broker%20services. "Get the AMQ Broker services"){.didact})

```oc get services -n event-streaming-kafka-cluster```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20get%20services%20-n%20event-streaming-kafka-cluster&completion=Get%20the%20AMQ%20Streams%20services. "Get the AMQ Streams services"){.didact})

They provide the addresses of the services running on the cluster and can be used to fill in the values on the properties file.

We start by opening the file [./config/application.properties](didact://?commandId=vscode.open&projectFilePath=./config/application.properties&newWindow=false&completion=Ok. "Edit the config map"){.didact} and editing the parameters. The content needs to be adjusted to point to the correct addresses of the brokers. It should be similar to this:

```
kafka.bootstrap.address=event-streaming-kafka-cluster-kafka-bootstrap.event-streaming-kafka-cluster:9092
messaging.broker.url=tcp://broker-hdls-svc.event-streaming-messaging-broker:61616
```

#### Creating the Secret

One of the components simulates receiving data from users and, in order to do so, authenticate the users. Because we normally don't want the credentials to be easily
accessible, it simulates checking the access control by reading a secret.

We can push the secret to the cluster using the following command:

```oc create secret generic example-event-streaming-user-reporting --from-file config/application.properties```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20create%20secret%20generic%20example-event-streaming-user-reporting%20--from-file%20config%2Fapplication.properties&completion=Created%20the%20secret%20configuration. "Creates the secret configuration"){.didact})

With this configuration secret created on the cluster, we have completed the initial steps to get the demo running.

### Pollution Report 

![Diagram](docs/Pollution.png)

#### Running the OpenAQ Consumer

Now we will deploy the first component of the demo: [./openaq-consumer/OpenAQConsumer.java](didact://?commandId=vscode.open&projectFilePath=./openaq-consumer/OpenAQConsumer.java&newWindow=false&completion=Ok. "View the source code"){.didact}

```kamel run openaq-consumer/OpenAQConsumer.java model/pollution/* --name open-aq-consumer```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kamel%20run%20openaq-consumer%2FOpenAQConsumer.java%20model%2Fpollution%2F*%20--name%20open-aq-consumer&completion=Started%20the%20OpenAQ%20Consumer. "Creates and starts the OpenAQ Consumer"){.didact})


**Details**: this starts an integration that consumes data from the [OpenAQ](https://docs.openaq.org/) API, splits each record and sends them to
our AMQ Stream instance. The demo addresses for the AMQ Streams broker is stored in the `example-event-streaming` which is inject into the demo
code and used to reach the instance.

#### Running the Pollution Bridge

This service consumes the pollution events and sends it to the timeline topic for consumption. Also sending notification and alarms to AMQ brokers.

```kamel run event-bridge/PollutionBridge.java model/common/Alert.java model/pollution/PollutionData.java --name pollution-bridge```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kamel%20run%20event-bridge%2FPollutionBridge.java%20model%2Fcommon%2FAlert.java%20model%2Fpollution%2FPollutionData.java%20--name%20pollution-bridge&completion=Run%20the%20Pollution%20bridge. "Run the Pollution bridge"){.didact})

#### Running the Timeline Bridge
This service consumes all the events to the web Front-end for display purpose

```kamel run event-bridge/TimelineBridge.java```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kamel%20run%20event-bridge%2FTimelineBridge.java&completion=Run%20the%20Timeline%20Bridge. "Run the Timeline Bridge"){.didact})


#### Running the Front-end

This web front end queries the timeline bridge service and displays the events collected at the time. We will use
OpenShift build services to build a container with the front-end and run it on the cluster.

The front-end image leverages the official [Apache Httpd 2.4](https://access.redhat.com/containers/?tab=tech-details#/registry.access.redhat.com/rhscl/httpd-24-rhel7) image from Red Hat's container registry.

We can proceed to creating the build configuration and starting the build within the OpenShift cluster. The
following command replaces the URL for the timeline API on the Javascript code and launches an image build.


```URL=$(oc get ksvc timeline-bridge -o 'jsonpath={.status.url}') ; cat ./front-end/Dockerfile| oc new-build --docker-image="registry.access.redhat.com/rhscl/httpd-24-rhel7:latest" --to=front-end --build-arg="URL=$URL" -D -```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$URL%3D%24(oc%20get%20ksvc%20timeline-bridge%20-o%20%27jsonpath%3D%7B.status.url%7D%27)%20%3B%20cat%20.%2Ffront-end%2FDockerfile%7C%20oc%20new-build%20--docker-image%3D%22registry.access.redhat.com%2Frhscl%2Fhttpd-24-rhel7%3Alatest%22%20--to%3Dfront-end%20--build-arg%3D%22URL%3D%24URL%22%20-D%20-&completion=Created%20the%20build%20configuration. "Creates the build configuration"){.didact})


With the build complete, we can go ahead and create a deployment for the front-end:

```oc apply -f front-end/front-end.yaml```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20apply%20-f%20front-end%2Ffront-end.yaml&completion=Deployed%20the%20front-end. "Deploys the front-end"){.didact})

Now, let's get the front-end URL so that we can open it on the browser.
To find the public API for the service, we can run the following command:

```oc get routes front-end-external -o 'jsonpath={.spec.port.targetPort}://{.spec.host}'```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20get%20routes%20front-end-external%20-o%20%27jsonpath=%7B.spec.port.targetPort%7D:%2F%2F%7B.spec.host%7D%27&completion=Found%20the%20front-end%20URL. "Gets the front-end URL"){.didact})

Open this URL on the browser and we can now access the front-end, and you should be able to see the pollution now showing in the browser. If pm > 25 it will display as RED and yellow for the rest. 

![Diagram](docs/Pollution-result.png)

### Crime Report 
![Diagram](docs/Crime.png)

### Running the GateKeeper

This service leverages [knative eventing channels](https://knative.dev/docs/eventing/channels/) to operate. Therefore, we need to create
them on the OpenShift cluster. To do so we can execute the following command:

```oc apply -f infra/knative/channels/audit-channel.yaml```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20apply%20-f%20infra%2Fknative%2Fchannels%2Faudit-channel.yaml&completion=Create%20Knative%20eventing%20channel. "Create knative eventing channel"){.didact})


The [Gatekeeper service](didact://?commandId=vscode.open&projectFilePath=./audit-gatekeeper/GateKeeper.java&newWindow=false&completion=Ok. "View the source code"){.didact} simulates a service that is used to audit accesses to the system. It leverages knative support from Camel-K.

```kamel run audit-gatekeeper/GateKeeper.java```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kamel%20run%20audit-gatekeeper%2FGateKeeper.java&completion=Run%20the%20GateKeeper%20audit. "Run the GateKeeper audit"){.didact})



### Running the Event Report System 


The [Event Report System](didact://?commandId=vscode.open&projectFilePath=./user-report-system/UserReportSystem.java&newWindow=false&completion=Ok. "View the source code"){.didact}  simulates a service that is used to receive user-generated reports on the system. It receives events sent by the user and sends them to the AMQ Streams instance. To run this component execute the following command:

```kamel run user-report-system/EventReportSystem.java model/common/Data.java --name event-report-system```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kamel%20run%20user-report-system%2FUserReportSystem.java%20model%2Fcommon%2FData.java%20--name%20user-report-system&completion=Run%20the%20User%20Report%20System. "Run the User Report System"){.didact})



#### Running the Crime Bridge

```kamel run event-bridge/CrimeBridge.java model/common/* --name crime-bridge```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$kamel%20run%20event-bridge%2FCrimeBridge.java%20model%2Fcommon%2F*%20--name%20crime-bridge&completion=Run%20the%20CrimeBridge. "Run the CrimeBridge"){.didact})



#### Report a crime incident from cloud event

Now that we launched all the services, let's check the state of our integrations. We can do
so with the command:

```URL=$(oc get ksvc event-report-system -o 'jsonpath={.status.url}')```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$URL=$(oc%20get%20ksvc%20event-report-system%20-o%20'jsonpath={.status.url}')%20&completion=Get%20report%20system%20URL. "Get report system URL"){.didact})


```curl --location --request POST $URL  --header 'ce-specversion: 1.0' --header 'ce-source: police-center' --header 'ce-type: hazard-event' --header 'ce-id: crime-123abc' --header 'content-type: application/json' --header 'Content-Type: application/json' --data-raw '{ "user": { "name": "user1" }, "report": { "type": "crime", "alert": "true", "measurement": "crime", "location": "20 W 34th St, New York" } }'```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$curl%20--location%20--request%20POST%20$URL%20--header%20'ce-specversion:%201.0'%20--header%20'ce-source:%20police-center'%20--header%20'ce-type:%20hazard-event'%20--header%20'ce-id:%20crime-123abc'%20--header%20'content-type:%20application/json'%20--header%20'Content-Type:%20application/json'%20--data-raw%20'{%20"user":%20{%20"name":%20"user1"%20},%20"report":%20{%20"type":%20"crime",%20"alert":%20"true",%20"measurement":%20"crime",%20"location":%20"20%20W%2034th%20St,%20New%20York"%20}%20}'&completion=Create%20a%20cloudevent%20for%20crime. "Create a cloudevent for Crime"){.didact})

![Diagram](docs/Crime-result.png)


## 4. Uninstall

To cleanup everything, execute the following command:

```oc delete project camel-k-event-streaming event-streaming-messaging-broker event-streaming-kafka-cluster```

([^ execute](didact://?commandId=vscode.didact.sendNamedTerminalAString&text=camelTerm$$oc%20delete%20project%20camel-k-event-streaming%20event-streaming-messaging-broker%20event-streaming-kafka-cluster&completion=Removed%20the%20projects%20from%20the%20cluster. "Cleans up the cluster after running the projects"){.didact})
