---
id: business-rule-tasks
title: "Business rule tasks"
description: "A business rule task is used to model the evaluation of a business rule."
---

A business rule task is used to model the evaluation of a business rule; for example, a decision
modeled in [Decision Model and Notation](https://www.omg.org/dmn/) (DMN).

![task](assets/business-rule-task.png)

:::note
Camunda Cloud supports alternative task implementations for the business rule task. See the
[Alternative task implementation](#alternative-task-implementation) section below, if you want to
use your own implementation for a business rule task. The sections before only apply to the DMN
decision implementation.
:::

When the process execution arrives at a business rule task, a decision is evaluated using the
internal DMN decision engine. Once the decision is made, the process instance continues.

If the decision evaluation is unsuccessful, an [incident](/components/concepts/incidents.md) is
raised at the business rule task. When the incident is resolved, the decision is evaluated again.

## Defining a called decision

A called decision links the business rule task to a DMN decision. It can be defined using the
`zeebe:calledDecision` extension element.

A business rule task must define the DMN decision id of the called decision as `decisionId`.
Usually, the `decisionId` is defined as a static value (e.g. `shipping_box_size`), but it can also
be defined as an [expression](/components/concepts/expressions.md) (e.g. `= "shipping_box_size_" +
countryCode`). The expression is evaluated on activating the business rule task (or when an incident
at the business rule task is resolved), after input mappings have been applied. The expression must
result in a `string`.

A business rule task must define the process variable name of the decision result as
`resultVariable`. The result of the decision will be stored in this variable. The `resultVariable`
is defined as a static value.

## Variable mappings

By default, the variable defined by `resultVariable` is merged into the process instance. This
behavior can be customized by defining an output mapping at the business rule task.

All variables in scope of the business rule task are available to the decision engine when the
decision is evaluated. Input mappings can be used to transform the variables into a format accepted
by the decision.

:::note
Input mappings are applied on activating the business rule task (or when an incident at the business
rule task is resolved), before the decision evaluation. When an incident is resolved at the business
rule task, then the input mappings are applied again before evaluating the decision. This can affect
the result of the decision.
:::

For more information about this topic visit the documentation about [Input/output variable
mappings](/components/concepts/variables.md#inputoutput-variable-mappings).

## Alternative task implementation
A business rule task does not have to evaluate a decision modeled with DMN. Instead, you can also
use [job workers](/components/concepts/job-workers.md) to implement your business rule task.

An alternative task implementation can be defined using the `zeebe:taskDefinition` extension element.

Business rule tasks with an alternative task implementation behave exactly like [service
tasks](/components/modeler/bpmn/service-tasks/service-tasks.md). The differences between these task
types are the visual representation (i.e. the task marker) and the semantics for the model.

When a process instance enters a business rule task with alternative task implementation, it creates
a corresponding job and waits for its completion. A job worker should request jobs of this job type
and process them. When the job is completed, the process instance continues.

:::note
Jobs for business rule tasks are not processed by Zeebe itself. To process them, you must provide a
job worker.
:::

A business rule task must define a [job type](/components/modeler/bpmn/service-tasks/service-tasks.md#task-definition) the same way as a service task does. This
specifies the type of job that workers should subscribe to (e.g. DMN).

Use [task headers](/components/modeler/bpmn/service-tasks/service-tasks.md#task-headers) to pass static parameters to the job
worker (e.g. the key of the decision to evaluate).

Define [variable mappings](/components/concepts/variables.md#inputoutput-variable-mappings)
the [same way as a service task does](/components/modeler/bpmn/service-tasks/service-tasks.md#variable-mappings)
to transform the variables passed to the job worker, or to customize how the variables of the job merge.

:::tip Community Extension

Take a look at the [Zeebe DMN Worker](https://github.com/camunda-community-hub/zeebe-dmn-worker).
This is a community extension providing a job worker to evaluate DMN decisions. You can run it, or
use it as a blueprint for your own job worker.

:::

### XML representation

A business rule task with a called decision:

```xml
<bpmn:businessRuleTask id="determine-box-size" name="Determine shipping box size">
  <bpmn:extensionElements>
    <zeebe:calledDecision  decisionId="shipping_box_size" resultVariable="boxSize" />
  </bpmn:extensionElements>
</bpmn:businessRuleTask>
```

A business rule task with an alternative task implementation and a custom header:

```xml
<bpmn:businessRuleTask id="calculate-risk" name="Calculate risk">
  <bpmn:extensionElements>
    <zeebe:taskDefinition type="DMN" />
    <zeebe:taskHeaders>
      <zeebe:header key="decisionRef" value="risk" />
    </zeebe:taskHeaders>
  </bpmn:extensionElements>
</bpmn:businessRuleTask>
```

### References

- [DMN decision](/components/modeler/dmn/dmn.md)
- [Job handling](/components/concepts/job-workers.md)
- [Variable mappings](/components/concepts/variables.md#inputoutput-variable-mappings)
