---
id: process-deploy
title: "Deploy a process"
description: "Let's analyze the prerequisites and code to deploy a process using Java."
---

## Related resources

- [Process basics](../../components/concepts/processes.md)
- [BPMN introduction](../../components/modeler/bpmn/bpmn-primer.md)

## Prerequisites

Run the Zeebe broker with endpoint `localhost:26500` (default).

## ProcessDeployer.java

[Source on GitHub](https://github.com/camunda-cloud/zeebe/tree/develop/samples/src/main/java/io/camunda/zeebe/example/process/ProcessDeployer.java)

```java
final DeploymentEvent deploymentEvent =
        client.newDeployResourceCommand()
            .addResourceFromClasspath("demoProcess.bpmn")
            .send()
            .join();
```

## demoProcess.bpmn

[Source on GitHub](https://github.com/camunda-cloud/zeebe/tree/develop/samples/src/main/resources/demoProcess.bpmn)

Download the XML and save it in the Java classpath before running the example. Open the file with Desktop Modeler for a graphical representation.

<!--
```xml
{{#include ../../../../samples/src/main/resources/demoProcess.bpmn}}
```
-->
