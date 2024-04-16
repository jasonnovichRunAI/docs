---
title: Cluster Health and Troubleshooting
summary: This article describes the troubleshooting steps to take in order to diagnose and resolve issues you may find in your cluster.
authors:
    - Gal Revach
    - Jason Novich
date: 2024-Jan-17
---

This troubleshooting guide helps you diagnose and resolve issues you may find in your cluster.
The cluster status is displayed in the Run:ai Contol Plane. See [Cluster Status](../runai-setup/cluster-setup/cluster-install.md#cluster-status) a list of possible statuses.

## Cluster is disconnected

When a cluster's status shows *Disconnected*, this means that no communication from the Run:ai cluster services reaches the Run:ai Control Plane.

This may reflect a networking issue from or to your Kubernetes cluster regardless of Run:ai components. In some cases, it may indicate an issue with one or more Run:ai services that communicate with the Control Plane. These are:

* Cluster sync (`cluster-sync`)
* Agent (`runai-agent`)
* Assets sync (`assets-sync`)

### Troubleshooting actions

Use the following steps to troubleshoot the issue:

1. Check that the Run:ai services that communicate with the Control Plane are up and running. Run the following command:

    `kubectl get pods -n runai | grep -E 'runai-agent|cluster-sync|assets-sync'`

    If one or more of the services are not running, see [Cluster has Service issues](#cluster-has-service-issues) for futher guidelines.

2. Check the network connection from the `runai` namespace in your cluster to the Control Plane.

    You can do that by running a connectivity check pod. This pod can be a simple container with basic network troubleshooting tools, such as `curl` or `get`. Use the following command to create the pod and determine if it can establish connections to the necessary Control Plane endpoints:

    ```bash
    kubectl run control-plane-connectivity-check -n runai --image=wbitt/network-multitool --command -- /bin/sh -c 'curl -sSf <control-plane-endpoint> > /dev/null && echo "Connection Successful" || echo "Failed connecting to the Control Plane"'
    ```

    Replace `control-plane-endpoint` with the URL of the Control Plane in your environment.

    If the pod has failed to connect to the Control Plane, check for potential network issues. Use the following guidelines:
  
    * Ensure that the network policies in your Kubernetes cluster allow communication between the Control Plane and the Run:ai services that communicate with the Control Plane.
    * Check both Kubernetes Network Policies and any network-related configurations at the infrastructure level.
    * Verify that the required ports and protocols are not blocked.

3. Verify Run:ai services logs. Inspecting the logs of the Run:ai services that communicate with the CP is essential to identify any error messages.

    Run the following command on each one of the services:

    ```bash
    kubectl logs deployment/runai-agent -n runai
    kubectl logs deployment/cluster-sync -n runai
    kubectl logs deployment/assets-sync -n runai
    ```

    Try to identify the problem from the logs. If you cannot resolve the issue, continue to the next step.

4. If the issue persists, contact Run:ai support for assistance.

    !!! Note
        The previous steps can be used if you installed the cluster and the status is stuck in *Waiting to connect* for a long time.

## Cluster has *service issues*

When a cluster's status shows *Service issues*, this means that one or more Run:ai services that are running in the cluster are not available.

1. Run the following command to verify which are the non-functioning services, and to get more details about deployment issues and resources required by these services that may not be ready (for example, ingress was not created or is unhealthy):

    ```bash
    kubectl get runaiconfig -n runai runai -ojson | jq -r '.status.conditions | map(select(.type == "Available"))'
    ```

    The list of non-functioning services is also available on the UI Clusters page.

2. After determining the non-functioning services, use the following guidelines to further investigate the issue.

    Run the following to get all the Kubernetes events and look for recent failures:

    ```bash
    Kubectl get events  -A
    ```

    If a required resource was created but not available or unhealthy, check the details by running:

    ```bash
    Kubectl describe <resource_type> <name>
    ```

3. If the issue persists, contact Run:ai support for assistance.

## Cluster has *missing prerequisites*

When a cluster's status displays *Missing prerequisites*, it indicates that at least one of the [Mandatory Prerequisites](../runai-setup/cluster-setup/cluster-prerequisites.md#prerequisites-in-a-nutshell) has not been fulfilled. In such cases, Run:ai services may not function properly.

If you have ensured that all prerequisites are installed and the status still shows *Missing prerequisites*, follow these steps:

1. Check the message in the Control Plane for further details regarding the missing prerequisites.
2. Inspect the [runai-public ConfigMap](#runai-public-configmap) and look for the `dependencies.required` field to obtain detailed information about the missing resources.
3. If the issue persists, contact Run:ai support for assistance.

## General tests to verify the Run:ai cluster health

Use the following tests regularly to determine the health of the Run:ai cluster, regardless of the cluster status and the troubleshooting options previously described.

### Verify that data is sent to the cloud

Log in to `<company-name>.run.ai/dashboards/now`.

* Verify that all metrics in the overview dashboard are showing.
* Verify that all metrics are showing in the Nodes view.
* Go to **Projects** and create a new Project. Find the new Project using the CLI command: `runai list projects`

### Verify that the Run:ai services are running

Run:

```bash
kubectl get runaiconfig -n runai runai -ojson | jq -r '.status.conditions | map(select(.type == "Available"))'
```

Verify that all the Run:ai services are available and have all their required resources available.

Run:

```bash
kubectl get pods -n runai
kubectl get pods -n monitoring
```

Verify that all pods are in `Running` status and a ready state (1/1 or similar).

Run:

```bash
kubectl get deployments -n runai
```

Check that all deployments are in a ready state (1/1).

Run:

```bash
kubectl get daemonset -n runai
```

A *Daemonset* runs on every node. Some of the Run:ai daemon-sets run on all nodes. Others run only on nodes that contain GPUs. Verify that for all daemonsets the *desired* number is equal to *current* and to *ready*.

### Submit a job using the command-line interface

Submitting a Job allows you to verify that the Run:ai scheduling service is running properly.

1. Make sure that the Project you have created has a quota of at least 1 GPU.
2. Run:

    ```bash
    runai config project <project-name>
    runai submit -i gcr.io/run-ai-demo/quickstart -g 1
    ```

3. Verify that the Job is in a *Running* state by running:

    ```bash
    runai list jobs
    ```

4. Verify that the Job is showing in the Jobs area at `<company-name>.run.ai/jobs`.

### Submit a job using the user interface

Log into the Run:ai user interface, and verify that you have a `Researcher` or `Research Manager` role.
Go to the `Jobs` area. On the top right, press the button to create a Job. Once the form opens, you can submit a Job.

## Advanced troubleshooting

### Run:ai public ConfigMap

Run:ai services use the `runai-public` ConfigMap to store information about the cluster status. This ConfigMap can be helpful in troubleshooting issues with Run:ai services.
Inspect the ConfigMap by running:

```bash
kubectl get cm runai-public -oyaml
```

### Resources not deployed / System unavailable / Reconciliation failed

1. Run the [Preinstall diagnostic script](cluster-prerequisites.md#pre-install-script) and check for issues.
2. Run

```
   kubectl get pods -n runai
   kubectl get pods -n monitoring
```

Look for any failing pods and check their logs for more information by running `kubectl describe pod -n <pod_namespace> <pod_name>`.