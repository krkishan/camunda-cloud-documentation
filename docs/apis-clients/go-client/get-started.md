---
id: get-started
title: "Go client - Getting started guide"
sidebar_label: "Getting started guide"
---

In this tutorial, you will learn how to use the Go client in a Go application to interact with Camunda Cloud.

You can find the complete source code on [GitHub](https://github.com/zeebe-io/zeebe-get-started-go-client).

## Prerequisites

- [Camunda Cloud account](/guides/getting-started/create-camunda-cloud-account.md)
- [Cluster](/guides/getting-started/create-camunda-cloud-account.md)
- [Client credentials](/guides/getting-started/setup-client-connection-credentials.md)
- [Modeler](/guides/getting-started/model-your-first-process.md)
- Go v1.13+ environment installed

## Set up a project

First, we need a new Go project. To do this, complete the following steps:

1. Create a new project using your IDE, or create a new Go module with the following command:

```
mkdir -p $GOPATH/src/github.com/zb-user/zb-example
cd $GOPATH/src/github.com/zb-user/zb-example
go mod init
```

2. To use the Zeebe Go client library, add the following dependency to your `go.mod`:

```
module github.com/zb-user/zb-example

go 1.17

require github.com/camunda-cloud/zeebe/clients/go v1.2.9
```

3. Set the connection settings and client credentials as environment variables:

```bash
export ZEEBE_ADDRESS='[Zeebe API]'
export ZEEBE_CLIENT_ID='[Client ID]'
export ZEEBE_CLIENT_SECRET='[Client Secret]'
export ZEEBE_AUTHORIZATION_SERVER_URL='[OAuth API]'
```

:::note
When you create client credentials in Camunda Cloud, you have the option to download a file with the lines above filled out for you.
:::

4. Create a `main.go` file inside the module and add the following lines to bootstrap the Zeebe client:

```go
package main

import (
	"context"
	"fmt"
	"github.com/camunda-cloud/zeebe/clients/go/pkg/zbc"
	"github.com/camunda-cloud/zeebe/clients/go/pkg/pb"
	"os"
)

func main() {
	client, err := zbc.NewClient(&zbc.ClientConfig{
		GatewayAddress: os.Getenv("ZEEBE_ADDRESS"),
	})

	if err != nil {
		panic(err)
	}

	ctx := context.Background()
	topology, err := client.NewTopologyCommand().Send(ctx)
	if err != nil {
		panic(err)
	}

	for _, broker := range topology.Brokers {
		fmt.Println("Broker", broker.Host, ":", broker.Port)
		for _, partition := range broker.Partitions {
			fmt.Println("  Partition", partition.PartitionId, ":", roleToString(partition.Role))
		}
	}
}

func roleToString(role pb.Partition_PartitionBrokerRole) string {
	switch role {
	case pb.Partition_LEADER:
		return "Leader"
	case pb.Partition_FOLLOWER:
		return "Follower"
	default:
		return "Unknown"
	}
}
```

5. Run the program.

```bash
go run main.go
```

You should see a similar output:

```
Broker 0.0.0.0 : 26501
  Partition 1 : Leader
```

## Model a process

Now, we need a simple process we can deploy. Later, we will extend the process with more functionality. For now, follow the steps below:

1. Open the [modeler](/guides/getting-started/model-your-first-process.md) of your choice and create a new BPMN diagram.

2. Add a start event named `Order Placed` and an end event named `Order Delivered` to the diagram. Then, connect the events.

![model-process-step-1](assets/order-process-simple.png)

3. Set the **id** (the BPMN process id), and mark the diagram as **executable**.

4. Save the diagram as `src/main/resources/order-process.bpmn` under the project's folder.

## Deploy a process

Next, we want to deploy the modeled process to the broker.

The broker stores the process under its BPMN process id and assigns a version.

```go
	// After the client is created
	ctx := context.Background()
	response, err := client.NewDeployResourceCommand().AddResourceFile("order-process.bpmn").Send(ctx)
	if err != nil {
		panic(err)
	}
	fmt.Println(response.String())
```

Run the program and verify the process deployed successfully.

You should see a similar output:

```
key:2251799813686743 processes:<bpmnProcessId:"order-process" version:3 processKey:2251799813686742 resourceName:"order-process.bpmn" >
```

## Create a process instance

We are ready to create our first instance of the deployed process.

A process instance is created by a specific version of the process, which can be set on creation.

```go
	// After the process is deployed.
	variables := make(map[string]interface{})
	variables["orderId"] = "31243"

	request, err := client.NewCreateInstanceCommand().BPMNProcessId("order-process").LatestVersion().VariablesFromMap(variables)
	if err != nil {
		panic(err)
	}

	ctx := context.Background()

	msg, err := request.Send(ctx)
	if err != nil {
		panic(err)
	}

	fmt.Println(msg.String())
```

Run the program and verify the process instance is created. You should see an output similar to below:

```
processKey:2251799813686742 bpmnProcessId:"order-process" version:3 processInstanceKey:2251799813686744
```

## See the process in action

Want to see how the process instance is executed? Follow the steps below:

1. Go to the cluster in Camunda Cloud and select it.
1. Click on the link to [Operate](/components/operate/userguide/basic-operate-navigation.md).
1. Select the process **order process**.

As you can see, a process instance has been started and finished.

## Work on a task

Now, we want to do some work within our process. Follow the steps below:

1. Add a few service tasks to the BPMN diagram and set the required attributes.

2. Extend your `main.go` file and activate a job. These are created when the process instance reaches a service task.

3. Open the BPMN diagram in the modeler. Insert three service tasks between the start and the end event.

- Name the first task `Collect Money`.
- Name the second task `Fetch Items`.
- Name the third task `Ship Parcel`.

![model-process-step-2](assets/order-process.png)

4. Set the **type** of each task, which identifies the nature of the work to be performed.

- Set the **type** of the first task to `payment-service`.
- Set the **type** of the second task to `fetcher-service`.
- Set the **type** of the third task to `shipping-service`.

The consolidated example looks as follows:

```go
package main

import (
	"context"
	"fmt"
	"github.com/camunda-cloud/zeebe/clients/go/pkg/entities"
	"github.com/camunda-cloud/zeebe/clients/go/pkg/worker"
	"github.com/camunda-cloud/zeebe/clients/go/pkg/zbc"
	"log"
	"os"
)

const ZeebeAddr = "0.0.0.0:26500"

var readyClose = make(chan struct{})

func main() {
	gatewayAddr := os.Getenv("ZEEBE_ADDRESS")
	plainText:= false

	if (gatewayAddr == "") {
		gatewayAddr = ZeebeAddr
		plainText = true
	}

	zbClient, err := zbc.NewClient(&zbc.ClientConfig{
		GatewayAddress:         gatewayAddr,
		UsePlaintextConnection: plainText,
	})

	if err != nil {
		panic(err)
	}

	// deploy process
	ctx := context.Background()
	response, err := zbClient.NewDeployResourceCommand().AddResourceFile("order-process-4.bpmn").Send(ctx)
	if err != nil {
		panic(err)
	}

	fmt.Println(response.String())

	// create a new process instance
	variables := make(map[string]interface{})
	variables["orderId"] = "31243"

	request, err := zbClient.NewCreateInstanceCommand().BPMNProcessId("order-process-4").LatestVersion().VariablesFromMap(variables)
	if err != nil {
		panic(err)
	}

	result, err := request.Send(ctx)
	if err != nil {
		panic(err)
	}

	fmt.Println(result.String())

	jobWorker := zbClient.NewJobWorker().JobType("payment-service").Handler(handleJob).Open()

	<-readyClose
	jobWorker.Close()
	jobWorker.AwaitClose()
}

func handleJob(client worker.JobClient, job entities.Job) {
	jobKey := job.GetKey()

	headers, err := job.GetCustomHeadersAsMap()
	if err != nil {
		// failed to handle job as we require the custom job headers
		failJob(client, job)
		return
	}

	variables, err := job.GetVariablesAsMap()
	if err != nil {
		// failed to handle job as we require the variables
		failJob(client, job)
		return
	}

	variables["totalPrice"] = 46.50
	request, err := client.NewCompleteJobCommand().JobKey(jobKey).VariablesFromMap(variables)
	if err != nil {
		// failed to set the updated variables
		failJob(client, job)
		return
	}

	log.Println("Complete job", jobKey, "of type", job.Type)
	log.Println("Processing order:", variables["orderId"])
	log.Println("Collect money using payment method:", headers["method"])

	ctx := context.Background()
	_, err = request.Send(ctx)
	if err != nil {
		panic(err)
	}

	log.Println("Successfully completed job")
	close(readyClose)
}

func failJob(client worker.JobClient, job entities.Job) {
	log.Println("Failed to complete job", job.GetKey())

	ctx := context.Background()
	_, err := client.NewFailJobCommand().JobKey(job.GetKey()).Retries(job.Retries - 1).Send(ctx)
	if err != nil {
		panic(err)
	}
}
```

In this example, we open a [job worker](/components/concepts/job-workers.md) for jobs of type `payment-service`.

The job worker will repeatedly poll for new jobs of the type `payment-service` and activate them subsequently. Each activated job will then be passed to the job handler, which implements the business logic of the job worker.

The handler will then complete the job with its result or fail the job if
it encounters a problem while processing the job.

When observing the Zeebe Monitor, you can see the process instance moved from the first service task to the next one.

When you run the example above, you should see a similar output to the following:

```
key:2251799813686751 processes:<bpmnProcessId:"order-process" version:4 processKey:2251799813686750 resourceName:"order-process.bpmn" >
processKey:2251799813686750 bpmnProcessId:"order-process" version:4 processInstanceKey:22517998136
86752
2019/06/06 20:59:50 Complete job 2251799813686760 of type payment-service
2019/06/06 20:59:50 Processing order: 31243
2019/06/06 20:59:50 Collect money using payment method: VISA
2019/06/06 20:59:50 Successfully completed job
```

## What's next?

- Learn more about the [concepts behind Zeebe](/components/concepts/what-is-camunda-cloud.md).
- Learn more about [BPMN processes](/components/modeler/bpmn/bpmn-primer.md).
