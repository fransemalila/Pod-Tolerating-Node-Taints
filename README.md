# Kubernetes Taints and Tolerations Example

This document demonstrates how to use taints and tolerations in Kubernetes to control pod scheduling.  Taints are applied to nodes and tolerations are applied to pods.  Pods with tolerations can be scheduled on nodes with matching taints.

## Prerequisites

* A working Kubernetes cluster.
* `kubectl` command-line tool configured to connect to your cluster.

## Step 1: Adding a Taint to a Node

1.  **Inspect existing taints (or lack thereof) on a node:**

    We'll use a node named `node01` in this example.  Replace this with the name of a node in your cluster if needed.

    ```bash
    kubectl describe node node01 | grep "Taints"
    ```

    Initially, the output should be:

    ```
    Taints:             <none>
    ```

2.  **Add a taint to the node:**

    This command adds a taint to `node01` with the key `node-role.kubernetes.io/worker`, value `true`, and effect `NoSchedule`.  The `NoSchedule` effect means that pods without a corresponding toleration *cannot* be scheduled on this node.

    ```bash
    kubectl taint node node01 node-role.kubernetes.io/worker="true":NoSchedule
    ```

3.  **Verify the taint has been added:**

    ```bash
    kubectl describe node node01 | grep "Taints"
    ```

    The output should now be:

    ```
    Taints:             node-role.kubernetes.io/worker=true:NoSchedule
    ```

## Step 2: Deploying a Pod *Without* Tolerations

1.  **Create a `pod.yaml` file with the following content:**

    This pod definition *does not* include any tolerations.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: random-generator
    spec:
      containers:
      - image: k8spatterns/random-generator:1.0
        name: random-generator
    ```

2.  **Apply the pod definition:**

    ```bash
    kubectl apply -f pod.yaml
    ```

3.  **Check the pod status:**

    ```bash
    kubectl get pod random-generator
    ```

    You will see the pod stuck in the `Pending` state because it cannot be scheduled on `node01` due to the taint and lack of a matching toleration.  The output will resemble this:

    ```
    NAME               READY   STATUS    RESTARTS   AGE
    random-generator   0/1     Pending   0          5s
    ```

## Step 3: Deploying a Pod *With* Tolerations

1.  **Replace the contents of `pod.yaml` with the following definition:**

    This pod definition *includes* a toleration that matches the taint on `node01`.  Specifically, it tolerates any taint with the key `node-role.kubernetes.io/worker` regardless of the value, and specifically addresses the `NoSchedule` effect.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: random-generator
    spec:
      containers:
      - image: k8spatterns/random-generator:1.0
        name: random-generator
      tolerations:
      - key: node-role.kubernetes.io/worker
        operator: Exists
        effect: NoSchedule
    ```

2.  **Apply the updated pod definition:**

    ```bash
    kubectl apply -f pod.yaml
    ```

3.  **Check the pod status and node assignment:**

    ```bash
    kubectl get pod random-generator -o wide
    ```

    You will see the pod running, and the `NODE` column will show that it's scheduled on `node01`:

    ```
    NAME               READY   STATUS    RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
    random-generator   1/1     Running   0          10s     10.244.2.5   node01   <none>           <none>
    ```

## Explanation of Taints and Tolerations

*   **Taints:**  Are key-value pairs associated with a node.  They indicate that the node has certain properties (e.g., it's a dedicated worker node, it's undergoing maintenance).
*   **Tolerations:** Are key-value pairs associated with a pod.  They indicate that the pod is willing to be scheduled on nodes with matching taints.
*   **Effect:** Determines what happens to pods that *don't* have a toleration matching the taint.  Common effects include:
    *   `NoSchedule`:  Prevents new pods without matching tolerations from being scheduled on the node.
    *   `PreferNoSchedule`:  The scheduler will *try* to avoid placing pods without matching tolerations on the node, but isn't guaranteed to.
    *   `NoExecute`:  Pods without matching tolerations will be evicted from the node if they are already running there.

## Use Cases

Taints and tolerations are useful for:

*   **Dedicated Nodes:**  Ensuring that certain pods only run on specific nodes (e.g., GPU-enabled nodes).
*   **Node Maintenance:**  Tainting a node before maintenance to prevent new pods from being scheduled and gracefully evicting existing pods.
*   **Specialized Workloads:** Isolating specific workloads to nodes with specific resources or configurations.
*   **Evicting Pods from Unhealthy Nodes:** Tainting unhealthy nodes with `NoExecute` to trigger the scheduler to reschedule pods.