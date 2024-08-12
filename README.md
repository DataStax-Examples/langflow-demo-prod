# Langflow in Production

The aim of this repository is to provide some example patterns for bringing Langflow into an Enterprise Production envrionment.
It currently provides examples for:

1. Docker containers
2. Kubernetes (with Helm)
3. [TODO] Terraform

## Base Principles and Assumptions

* Langflow will be used as an API by various clients (JavaScript, React, Python, Java, etc.).
* Production Langflow should not provide front-end UI access.
* Langflow's basic authentication mechanics are not Enterprise-ready:
  * Langflow is to be configured as "open" and accessible to any caller
  * Langflow services are configured behind an API gateway and/or private networking
* Langflow can be configured to use a common centralized Postgres database:
  * This is appropriate for development environments
  * This is less appropriate for post-development environments, as the whole environment
    would be upgraded at once as a Flow is updated.
  * This repository takes the opinion that it is better to gradually introduce new code 
    (Flows) to an environment, as it means that faulty code can be rolled back without
    having affected all users. 
  * Feel free to configure a Postgres database per environment – though this repository
    does not detail this, the configuration process is well-documented elsewhere.

## Limitations and Indemnification

* This is a general-purpose guide, provided under the [MIT License](LICENSE.md). You 
should take this as an instructive starting point for your own implementation and 
apply your own Enterprise architucture patterns to it.

## Langflow Version

This repo is currently based on Langflow v1.0.14; future versions of Langflow may change
the advice.

## Contributions

Contributions welcome, please raise a Pull Request!

# Process Overview

## Development

Flows are developed using a Langflow front-end GUI, sharing a common Postgres database. Configuring this
is out of scope of this repository, you may start with the [Langflow configuration page](https://docs.langflow.org/configuration-authentication).

### Langflow Upgrades
As Langflow versions are updated currently each flow needs to be reviewed:

* Components may need to be updated - a ⚠️ icon is shown on the top of such components.
* Any code customizations need to be re-implemented after update.

### Flow Updates

Flow version management is done manually, via Export to a version control system such as Git.

### Packaging 

Langflow does not currently provide a direct means of packaging or importing packages. How to manage
this is addressed in the Promtion section below.

## Higher Environments

Post-Development environments are not expected to have a front-end GUI, but rather run in "backend-only" mode. 
How these environments are configured will be dicussed later, but for now simply understand that the 
environment specification includes Flows to be executed from that environment.

**Note:** A few Langflow components use the Langflow Postgres database (such as `Chat Memory`) to persist state 
across sessions. As mentioned earlier, this repository does not make use of a shared Postgres database – each 
runtime operates in isolation from other runtimes. You should use components like `Store Message` (with External
Memory) to manage state explicitly.

# Deploying Runtime Backends

**Note:** It is most important that the Langflow versions are aligned: the exported `.json` files do not make
explicit mention of the Langflow version, but the Langflow runtime is, at present, very sensitive to version
mis-matches. Be sure that the version you are importing is compatible to the runtime version.

## Environment Variables

Most Flows make use of "Global Variables" to store things like API keys. Within the runtime backend, these
are specified via environment variables. For example, if your flow references a variable named `GOOGLE_API_KEY`, 
you should set an environment variable named `GOOGLE_API_KEY` with the appropriate value. 

Managing these environment variables is out of scope of this repository.

## Docker

Langflow maintains a Docker Repository at https://hub.docker.com/r/langflowai/langflow , which is referenced
by the included [Dockerfile](Dockerfile). Note these two lines:

```
COPY flows /app/flows
ENV LANGFLOW_LOAD_FLOWS_PATH=/app/flows
```

Here, files are copied from the local `flows` directory into `/app/flows`, and when the container starts, 
all of the flows specified in the path pointed to by the `LANGFLOW_LOAD_FLOWS_PATH` are imported into the 
runtime. 

**Note:** If using Docker, you should be sure to specify the `Endpoint Name` on each Flow's Settings page. Flow IDs
are currently generated and updated during the import process, with the included Flow ID being replaced. 
Using `Endpoint Name` controls this value, in addition to being more user-friendly.

To build a Docker image, then, you:

1. Put the appropriate version of all of the exported Flow `.json` files into the `flows/` directory;
2. Run `docker build` with the appropriate tag;
3. Push the Docker image to your container registry.

And then run the Docker container as normal, forwarding API requests to the container port `7680`. For example, a
local image:tag `mylangflow:latest` could be run with:

```bash
docker run -d -p 7860:7860 mylangflow:latest
```

## Kubernetes (with Helm)

This asssumes you have a Kubernetes cluster, and have both `kubectl` and `helm` installed and configured. 

Langflow maintains a Helm chart for Langflow runtime at https://github.com/langflow-ai/langflow-helm-charts/tree/main/charts/langflow-runtime . 

An stripped-down example [values.yaml](values.yaml) file is provided within this repository, but the definitive version is in the 
[langflow-helm-charts repo](https://github.com/langflow-ai/langflow-helm-charts/blob/main/charts/langflow-runtime/values.yaml).

A few points:

* In the `image:` parameter you can specify just `tag:` (which default to the Langflow Docker `langflowai/langflow-backend` repo), 
  and also specify `repository:` if you wanted to use your own image;
  * If you default to the Langflow image, you should also specify `downloadFlows:`. 
    * Each `url:` should correspond with a downloadable URL. On GitHub, it is advised to use a tag or a SHA, as 
      CDN caching has been observed to result in stale flows being downloaded.
    * An additional parameter `endpoint:` allows you to override (or specify) the endpoint name of the flow.    
    * Note that `basicAuth` and `headers.Authorization:` provide authentication mechanics to the download URL, 
      and this is not currently configurable as a K8s secret.
  * If necessary you can specify an `imagePullSecrets` entry following the standard convention for K8s image pull secrets.
* Environment variables for each pod are specified in the `env:` parameter; these are either plaintext values or else
  reference K8s secrets. The environment variable `name:` should correspond with the "Global Variable" value referenced 
  by your flows. For example, if your flow references a variable named `GOOGLE_API_KEY`, you should set an environment 
  variable named `GOOGLE_API_KEY` with the appropriate 

## Installing and Updating Helm Charts

To install the Helm charts:

```bash
helm repo add langflow https://langflow-ai.github.io/langflow-helm-charts
helm repo update
```

And to update/upgrade to the latest chart versions is just:

```bash
helm repo update
```

## Installing and Upgrading Chart Values

Assuming a namespace `langflow` exists, install the chart values file named `values.yaml` the first time with:

```bash
helm install -n langflow langflow-runtime langflow/langflow-runtime -f values.yaml
```

To upgrade to updated `values.yaml`:

```bash
helm upgrade -n langflow langflow-runtime langflow/langflow-runtime -f values.yaml
```

Most of the time, an `upgrade` triggers a deployment restart, but not always. You can manually restart the 
deployment with:

```bash
kubectl rollout restart deployment -n langflow langflow-runtime
```

## Exposing Langflow service

In a Kubernetes environment, backend services such as the Langflow API are not typically exposed (via an Ingress)
to the outside world (i.e. your computer). However, for testing purposes you may wish to create a temporary 
port forward:

```bash
kubectl port-forward -n langflow service/langflow-runtime 7860:7860
```

# Simple Testing

Assuming you have a backend runtime available on `localhost` port `7860`, and have both `curl` and `jq` available on
the command line, you can do some basic testing like:

```bash
curl -X POST \
    "http://127.0.0.1:7860/api/v1/run/basic-memory-chatbot?stream=false" \
    -H 'Content-Type: application/json'\
    -d '{"input_value": "what is my name",
    "output_type": "chat",
    "input_type": "chat",
    "session_id": "9c5ac5fe-caae-471e-a3cb-b0a69428af95"
}' | jq '.outputs[0].outputs[0].results.message.text'
```

You may find it easier to use a tool like Postman rather than the command line!

## Included Example Flows

### Installing 

There are three environment variables that need to be set:

1. `ASTRA_DB_API_ENDPOINT`
2. `ASTRA_DB_APPLICATION_TOKEN`
3. `OPENAI_API_KEY`

If you are using the provided `values.yaml`, there should be two secrets named thusly:

DataStax Astra DB Token:
```
apiVersion: v1
kind: Secret
metadata:
  name: astra-db
type: Opaque
data:
  ASTRA_DB_API_ENDPOINT: <base64-encoded Astra DB Endpoint>
  ASTRA_DB_APPLICATION_TOKEN: <base64-encoded Astra Token>
```

And OpenAI API Key:
```
apiVersion: v1
kind: Secret
metadata:
  name: openai-api-key
type: Opaque
data:
  OPENAI_API_KEY: <base64-encoded OpenAI API key>
```

### Running

Included in the repo is a file [example.html](example.html) which makes use of the Langflow [embedded chat widget](https://github.com/langflow-ai/langflow-embedded-chat), 
and assumes the API endpoint is available on https://localhost:7860 (i.e. port forwarding is enabled). 

It's usage is fairly simple: a table of available session UUIDs can be maintained at the top, and available chatbots can be interacted with using
these session identifiers. 

