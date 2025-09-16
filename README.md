# Specification: Execution System for MicroModels and Transformations


## 1. Scope

This specification defines the execution system for models described by the MicroModel and Transformation Ontology ([mmut.ttl](https://github.com/TheOpenMMTLab/mmut-rdf-model/blob/main/py_mmut_rdf/mmut.ttl)). The ontology is available in the [mmut-rdf-model repository](https://github.com/TheOpenMMTLab/mmut-rdf-model).

The ontology provides the following main concepts:
- **MicroModel**: Abstract base for data or model artifacts, with subclasses such as BinaryMicroModel, RDFMicroModel, SysMLMicroModel.
- **Transformation**: Abstract base for processing steps, with subclasses such as PythonScriptTransformation.
- **TaskDefinition**: Describes the configuration for concrete execution steps, including references to Docker images, commands, parameters, and environment variables.
- **ContainerProperties**: Specifies properties of a container, such as the image, command sequence, and environment.
- **Environment**: Defines environment variables as key-value pairs for container execution.

In the ontology, each Transformation can define its input and output MicroModels. This allows the relationships between models and transformations to be represented in a graph.

The model also includes TaskDefinitions, which configure the concrete execution of tasks by specifying the Docker image, command sequence, parameters, and environment variables. TaskDefinitions define how models are loaded and stored (generally referred to as Model Adapters) and how transformations are executed.

From these definitions, an **Execution Graph** is implicitly derived. It is a directed acyclic graph (DAG) that determines the order in which Model Adapters and Transformations are executed and serves as the input to the execution environment.


## 2. Terms and Definitions

| Term                | Definition                                                                                  |
|---------------------|--------------------------------------------------------------------------------------------|
| MicroModel          | An abstract entity representing a data or model artifact. Subclasses include BinaryMicroModel, RDFMicroModel, SysMLMicroModel. |
| Transformation      | An abstract processing step that consumes one or more MicroModels and produces new MicroModels. Subclasses include PythonScriptTransformation. |
| TaskDefinition      | A class describing the configuration for a concrete execution step, including container properties. |
| ContainerProperties | Properties of a container, such as image, command sequence, and environment.               |
| Environment         | An environment for executing containers, defined as a set of key-value pairs.              |
| KeyValuePair        | A key-value pair used for environment variables or configuration.                          |
| Execution Graph     | An implicitly derived DAG specifying the execution order of Model Adapters and Transformations. |
| Orchestration       | The automated scheduling and execution of transformations in the correct order.            |


## 3. Requirements

### 3.1 Functional Requirements

- **FREQ-1:** The system **must** accept a model as input, described using the MicroModel and Transformation Ontology (including MicroModels, Transformations, TaskDefinitions, ContainerProperties, and Environment).
- **FREQ-2:** The system **must** derive a topological order of Transformations from the Execution Graph defined in the model.
- **FREQ-3:** Each Transformation **must** be executed in an isolated runtime environment (Docker container) as specified by its TaskDefinition and ContainerProperties.
- **FREQ-4:** Transformations **must** consume input MicroModels and produce output MicroModels as defined in the ontology.
- **FREQ-5:** The system **must** automatically orchestrate execution of all Transformations according to their dependencies in the Execution Graph.
- **FREQ-6:** The system **should** track execution status of each Transformation (e.g., pending, running, succeeded, failed).


### 3.2 Non-functional Requirements

- **NFREQ-1: Portable and Environment-Independent Execution** Transformations and MicroModel Adapters **must** run in any Docker-supported environment without depending on specific infrastructure or orchestration technology. 

- **NFREQ-2: Extensibility** New MicroModels or Transformations **should** be addable via the ontology without modifying the core orchestration logic.


## 4. Interfaces

### 4.1 Input

The model description is provided as an RDF graph using the MicroModel and Transformation Ontology. It includes:
- MicroModels (including subclasses such as BinaryMicroModel, RDFMicroModel, SysMLMicroModel)
- Transformations (including subclasses such as PythonScriptTransformation)
- TaskDefinitions, each referencing:
	- ContainerProperties (specifying the Docker image, command sequence, and environment)
	- Environment (a set of KeyValuePairs for environment variables)
- The Execution Graph (DAG) describing dependencies between Transformations
- Configuration for orchestration and container execution


**Example (Turtle excerpt):**

```turtle
@prefix mmut: <http://frittenburger.de/ontology/mmut#> .

ex:task1 a mmut:TaskDefinition ;
	mmut:hasContainerProperties ex:containerProps1 .

ex:containerProps1 a mmut:ContainerProperties ;
	mmut:image "my-transform:latest" ;
	mmut:hasCommandSequence ("python" "main.py" "--input" "/shared/models/input.csv") ;
	mmut:hasEnvironment ex:env1 .

ex:env1 a mmut:Environment ;
	mmut:hasKeyValuePair ex:kv1, ex:kv2 .
ex:kv1 a mmut:KeyValuePair ; mmut:key "API_KEY" ; mmut:value "..." .
ex:kv2 a mmut:KeyValuePair ; mmut:key "GLOBAL_CONFIG" ; mmut:value "/config/global.yaml" .
```


### 4.2 Executable Containers (Adapter and Transformations)

Each MicroModel Adapter and each Transformation is executed as a Docker container according to its TaskDefinition and associated ContainerProperties and Environment.

**Execution:**
- The orchestrator starts the container and passes the command sequence as specified in the model (mmut:hasCommandSequence).
- Environment variables are set as specified by the Environment (mmut:hasEnvironment, mmut:KeyValuePair).
- A shared directory (e.g., `/shared/models`) is mounted into the container, providing access to MicroModels and configuration files.

**Example Docker invocation:**

```bash
docker run --rm \
	-e API_KEY=... \
	-e GLOBAL_CONFIG=/config/global.yaml \
	-v /host/path/models:/shared/models \
	my-transform:latest \
	python main.py --input /shared/models/input.csv
```

Inputs and outputs are exchanged via the shared model mount. Configuration and secrets are provided via environment variables. The command and its parameters are modeled in the ontology for each Task.

## 5. Behavior

1. **Graph Analysis:**
	- The system parses the model description.
	- A topological ordering of tasks (model adapters and transformations) is computed.

2. **Execution:**
	- Tasks without unmet dependencies are launched.
	- Each task runs in a Docker container.


3. **MicroModel Access:**
	- Inputs are consumed as MicroModels by Transformations.
	- Outputs are produced as MicroModels by Transformations.

4. **Orchestration:**
	- The orchestrator schedules and monitors tasks.
	- Failures are detected and marked in the execution status.
	- Dependent tasks are executed only if all predecessors succeed.


**Example execution flow:**

1. Model is loaded as RDF-Turtle (.ttl).
2. Topological sorting results in: t1 → t2 → t3
3. t1 is started, loads m1 via Adapter, persists m2.
4. t2 starts after successful completion of t1, etc.


## 6. Error Handling

- **Task failure:** If a Task (Model Adapter or Transformation) fails during execution, it is marked as failed in the Execution Graph (DAG). All dependent tasks in the graph will not be executed.

- **Invalid execution graph:** Cycles or other invalid structures in the Execution Graph are detected during model loading. If detected, execution is rejected entirely, and no tasks are started.


## 7. Conformance Criteria

An implementation is conformant if it:

- **CC-1:** Executes tasks in the correct order derived from the graph.
- **CC-2:** Runs each tasks in a Docker container.
- **CC-3:** Uses TaskDefinitions and ContainerProperties as specified in the ontology for executing Transformations and MicroModel adapters.
- **CC-4:** Automates orchestration of the execution.
- **CC-5:** Tracks and reports execution status correctly.


## 8. Security Aspects

- Secrets required for execution (such as API keys or credentials) are not included directly in the model. Instead, the model specifies a key or identifier for each secret.
- At runtime, the orchestrator uses these keys to retrieve the actual secret values from a SecretStore (e.g., HashiCorp Vault, AWS Secrets Manager, Kubernetes Secrets).
- The resolved secrets are injected into the container environment as environment variables before execution.

## 9. Non-normative Notes

- Orchestration systems may include Prefect, GitLab pipelines, or any DAG-capable workflow engine.
- Performance optimizations (e.g., parallel execution) are permitted as long as dependency semantics are preserved.
- Logging and monitoring are recommended but not mandatory.




---




