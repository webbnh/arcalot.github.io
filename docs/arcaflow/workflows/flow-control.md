# Using Flow Control Mechanics

Flow control allows the workflow author to build a workflow with a decision tree based on supported flow logic. These flow control operations are not implemented by plugins, but are part of the workflow engine itself.

## Foreach Loops

Foreach loops allow for running a sub-workflow with iterative inputs from a parent workflow. A sub-workflow is a complete Arcaflow workflow file with its own input and output schemas as described in this section. The inputs for the sub-workflow are provided as a list, where each list item is an object that matches the sub-workflow input schema.

!!! tip
    A complete functional example is available in the [arcaflow-workflows](https://github.com/arcalot/arcaflow-workflows/tree/main/examples/sub-workflow-foreach) repository.

In the parent workflow file, the author can define an input schema with the list that will contain the input object that will be passed to the sub-workflow. For example:

```yaml title="workflow.yaml"
input:
  root: RootObject
  objects:
    RootObject:
      id: RootObject
      properties:
        loop:
          type:
            type_id: list
            items:
              type_id: object
              id: loop_id
              properties:
                loop_id:
                  type:
                    type_id: integer
                param_1:
                  type:
                    type_id: integer
                param_2:
                  type:
                    type_id: string
```

Then in the `steps` section of the workflow, the sub-workflow can be defined as a step with the `loop` list object from above passed to its input.

The parameters for the sub-workflow step are:

  - `kind` - The type of loop (currently only foreach is supported)
  - `items` - A list of objects to pass to the sub-workflow (the [expression language](expressions.md) allows to pass this from the input schema per the above example)
  - `workflow` - The file name for the sub-workflow (this should be in the same directory as the parent workflow)
  - `parallelism` - The number of sub-workflow loop iterations that will run in parallel

```yaml title="workflow.yaml"
steps:
  sub_workflow_loop:
    kind: foreach
    items: !expr $.input.loop
    workflow: sub-workflow.yaml
    parallelism: 1
```

The input yaml file for the above parent workflow would provide the list of objects to loop over as in this example:

```yaml title="input.yaml"
loop:
  - loop_id: 1
    param_1: 10
    param_2: "a"
  - loop_id: 2
    param_1: 20
    param_2: "b"
  - loop_id: 3
    param_1: 30
    param_2: "c"
```

The sub-workflow file then has its complete schema and set of steps as in this example:

```yaml title="sub-workflow.yaml"
version: v0.2.0
input:
  root: RootObject
  objects:
    RootObject:
      id: RootObject
      properties:
        loop_id:
          type:
            type_id: integer
        param_1:
          type:
            type_id: integer
        param_2:
          type:
            type_id: string
steps:
  my_plugin:
    plugin: 
      deployment_type: image
      src: path/to/my_plugin:1
    input:
      param_1: !expr $.input.param_1
  my_other_plugin:
    plugin: 
      deployment_type: image
      src: path/to/my_other_plugin:1
    input:
      param_2: !expr $.input.param_2
outputs:
  success:
    loop_id: !expr $.input.loop_id
    my_plugin: !expr $.steps.my_plugin.outputs.success
    my_other_plugin: !expr $.steps.my_other_plugin.outputs.success
```

### Reduce Repetition with `bindConstants()`

The builtin function [`bindConstants()`](expressions.md#functions) allows you to 
avoid repeating input variables for a `foreach` subworkflow. In the example 
below, the input variable `name`'s value is repeated across each iteration in 
this input. This results in a more repetitive input and schema definition. This 
section will show you how to simplify it. 

#### Workflow and Input Before `bindConstants()`

```yaml title="input-repeated.yaml"
iterations:
  - loop_id: 1
    repeated_inputs:
      name: mogo
      ratio: 3.14
  - loop_id: 2
    repeated_inputs:
      name: mogo
      ratio: 3.14
  - loop_id: 3
    repeated_inputs:
      name: mogo
      ratio: 3.14
  - loop_id: 4
    repeated_inputs:
      name: mogo
      ratio: 3.14
```

```yaml title="workflow.yaml"
version: v0.2.0
input:
  root: RootObject
  objects:
    RootObject:
      id: RootObject
      properties:
        iterations:
          type:
            type_id: list
            items:
              id: SubRootObject
              type_id: ref
              namespace: $.steps.foreach_loop.execute.inputs.items
            
steps:
  foreach_loop:
    kind: foreach
    items: !expr $.input.iterations
    workflow: subworkflow.yaml
    parallelism: 1

outputs:
  success:
    fab_four: !expr $.steps.foreach_loop.outputs.success.data
```

```yaml title="subworkflow.yaml"
version: v0.2.0
input:
  root: SubRootObject
  objects:
    SubRootObject:
      id: SubRootObject
      properties:
        loop_id:
          type:
            type_id: integer
        repeated_inputs:
          type:
            type_id: ref
            id: RepeatedInputs
    RepeatedInputs:
      id: RepeatedInputs
      properties:
        name:
          type:
            type_id: string
        ratio:
          type:
            type_id: float            

steps:
  example:
    plugin:
      deployment_type: image
      src: quay.io/arcalot/arcaflow-plugin-template-python:0.4.0
    input:
      name: !expr $.input.repeated_inputs.name

outputs:
  success:
    loop_id: !expr $.input.loop_id
    ratio: !expr $.input.repeated_inputs.ratio
    beatle: !expr $.steps.example.outputs.success
```

#### Reduced Repetition Workflow

Here we restructure the input, factoring out the repeated `name` and `ratio` 
entries in the list and placing them into a single field; we will use 
`bindConstants()` to construct the `foreach` list with repeated entries.

```yaml title="input.yaml"
repeated_inputs: 
  name: mogo
  ratio: 3.14
iterations:
  - loop_id: 1
  - loop_id: 2
  - loop_id: 3
  - loop_id: 4
```

To use the generated values from `bindConstants()`, a new schema representing 
these bound values must be added to the input schema section of our 
`subworkflow.yaml`, `input`. This new schema's ID will be the ID of the schema 
that defines the items in your list, in this case `SubRootObject` and the 
schema name that defines your repeated inputs, in this case `RepeatedValues`, 
for more information see 
[Generated Schema Names](schemas.md#generated-combined-schema-names). This 
creates our new schema ID, `SubRootObject__RepeatedValues`. You are required 
to use this schema ID because it is generated from the names of your other 
schemas.

```yaml title="workflow.yaml"            
steps:
  foreach_loop:
    kind: foreach
    items: !expr 'bindConstants($.input.iterations, $.input.repeated_inputs)'
    workflow: subworkflow.yaml
    parallelism: 1
```

To use `bindConstants()` with an `outputSchema` in your workflow, you need to 
reference the schema of the list items returned by `bindConstants()`, see 
[Generated Schema Name](schemas.md#generated-combined-schema-names).

See the [full workflow](https://github.com/arcalot/arcaflow-workflows/blob/492e30ffbea6ce902e6e7ec050c4d1be307b6d73/basic-examples/bind-constants/workflow.yaml#L28).
