# Argo Workflow 简介

CRD 共有三种，分别是 Workflow，WorkflowTemplate，ClusterWorkflowTemplate 和 CronWorkflow。

## Workflow

### Template

Argo Workflow 使用不同类型的 template 声明需要运行的任务，目前共有 6 种类型的 Template，分别是 container, script, dag, steps, resource, suspend。

在开始任务时，需要指定一个 template 作为入口，以执行后续任务。

ContainerTemplate 包含一个 container 字段，该字段的内容与 k8s 中接受的内容相同。

```typescript
interface ArgoContainerTemplate {
  name: string
  container: K8sContainer
}
```

ScriptTemplate 是 container 的一个糖，其中 source 为执行的命令。命令执行结果会被自动导出为 argo 中的变量。

```typescript
interface ArgoScriptTemplate {
  name: string
  script: K8sContainer & { source: string }
}
```

ResourceTemplate 可以对集群中的 CRD 进行操作，例如创建/删除/更新 ConfigMap。其中 manifest 字段为资源的 yml 定义。

```typescript
interface ArgoResourceTemplate {
  name: string
  resource: {
    action: 'get' | 'create' | 'apply' | 'delete' | 'replace' | 'patch'
    manifest: string
  }
}
```

SuspendTemplate 指定休眠时间。

```typescript
interface ArgoSuspendTemplate {
  name: string
  suspend: {
    duration: string
  }
}
```

StepTemplate 按顺序执行步骤。注意此处 steps 中为一嵌套数组，第一层数组中的任务会被顺序执行，第二层数组中的任务并发执行。

```typescript
interface ArgoStepTemplate {
  name: string
  steps: {
    name: string
    template: string
  }[][]
}
```

DagTemplate 有向无环图执行。dependencies 字段指定了当前任务的前置条件，数组中值为其他任务的任务名。必须在所有前置任务执行完成之后，当前任务才可以被执行。

```typescript
interface ArgoDagTemplate {
  name: string
  dag: {
    tasks: {
      name: string
      dependencies?: string[]
      template: string
    }
  }
}
```

### Input & Output

Parameter 的表达方式通常如下

```typescript
interface ArgoArguments {
  parameters: {
    name: string
    value: any
  }[]
}

```

#### Example 1 变量声明

在 spec 中声明了变量 `message`，并在 whalesay 任务中以 `{{inputs.parameters.message}}` 进行了调用。

```yml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-parameters-
spec:
  entrypoint: whalesay
  arguments:
    parameters:
      - name: message
        value: hello world
  templates:
    - name: whalesay
      inputs:
        parameters:
          - name: message
      container:
        image: docker/whalesay
        command: [ cowsay ]
        args: [ "{{inputs.parameters.message}}" ] 
```

#### Example 2 上下文变量调用

在 dag 中声明了变量 `template-param-1`，并在 script 任务中以 `{{inputs.parameters.template-param-1}}` 进行了调用。

```yml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: example-
spec:
  entrypoint: main
  arguments:
    parameters:
    - name: workflow-param-1
  templates:
  - name: main
    dag:
      tasks:
      - name: step-A 
        template: step-template-A
        arguments:
          parameters:
          - name: template-param-1
            value: "{{workflow.parameters.workflow-param-1}}"

  - name: step-template-A
    inputs:
      parameters:
        - name: template-param-1
    script:
      image: alpine
      command: [/bin/sh]
      source: |
          echo "{{inputs.parameters.template-param-1}}"
```

#### Example 3 步骤间变量调用

在后续步骤中可以对之前步骤的 output 进行调用。

```yml
dag:
  tasks:
  - name: step-A 
    template: step-template-A
    arguments:
      parameters:
      - name: template-param-1
        value: "{{workflow.parameters.workflow-param-1}}"
  - name: step-B
    dependencies: [step-A]
    template: step-template-B
    arguments:
      parameters:
      - name: template-param-2
        value: "{{tasks.step-A.outputs.parameters.output-param-1}}"
```
