---
title: "Kubernetes basics"
author: [Szymon Byczek]
date: "28/05/2024"
version: 0.1
status: "Draft"
---
**Content:**
- [1. ReplicaSet](#1-replicaset)
- [2. DeamonSet](#2-deamonset)
- [3. Deployment](#3-deployment)
  - [3.1. Deployment strategies](#31-deployment-strategies)
    - [3.1.1. Check ReplicaSet](#311-check-replicaset)
- [4. Cleanup](#4-cleanup)



# 1. ReplicaSet
1. Create replicaset (rs) descriptor
    ```bash
    vim rs-test.yaml
    ```
    Paste the content and save
    ```yaml
    apiVersion: apps/v1
    kind: ReplicaSet
    metadata:
      name: frontend
      namespace: test
      labels:
        app: guestbook
        tier: frontend
    spec:
      # modify replicas according to your case
      replicas: 3
      selector:
        matchLabels:
          tier: frontend
      template:
        metadata:
          labels:
            tier: frontend
        spec:
          containers:
          - name: nginx
            image: nginx
    ```
2. Apply the file
   ```bash
   kubectl apply -f rs-test.yaml
   ```
3. List replicaset
   ```bash
   kubectl get rs -n test
   ```
4. Describe created replicaset
   ```bash
   kubectl -n test describe rs frontend
   ```
5. List pods in the `test` namespace

   ```bash
   kubectl get po -n test
   ```

   <p><details><summary><b>Output</b></summary>

    ```
    NAME             READY   STATUS    RESTARTS   AGE
    curl             1/1     Running   0          12d
    frontend-4qbll   1/1     Running   0          11m
    frontend-5pnbw   1/1     Running   0          11m
    frontend-bkpsj   1/1     Running   0          11m
    ```
   </details></p>

6. Check labels of pods
   ```bash
   kubectl -n test get po --show-labels
   ```
7. Rescale to 5 pods
   ```bash
   kubectl -n test scale replicaset frontend --replicas 5
   ```
8. Check pods
   ```bash
   kubectl get pods -n test
   ```
   There should be 5 pods of frontend
9. Check replicaset
   ```bash
   kubectl -n test get rs
   ```
10. Edit `rs-test.yaml` and change `replicas: 4`
    ```bash
    kubectl apply -f rs-test.yaml
    ```
11. Check `rs` status and pods. In describe output check the status at the bottom. It should display history of creation and deletion of pods. 
    ```bash
    kubectl -n test get po --show-labels
    kubectl -n test get rs
    kubectl -n test describe rs frontend
    ```
12. Delete one of the pods `frontend` then check the pods.
    ```bash
    kubectl -n test delete pod frontend-5x72v
    kubectl -n test get po -o wide -w
    kubectl -n test describe rs frontend
    ```
    **NOTE**:To stop watching press `Ctrl+c`
13. Add new pod with label `tier=frontend`
    ```bash
    kubectl -n test run frontend --image nginx --labels tier=frontend
    kubectl -n test get po --show-labels -w
    ```
14. Describe replicaset and check the events at the bottom 
    ```bash
    kubectl -n test describe rs frontend
    ```
15. Edit replicaSet's `frontend` spec.template.spec.container.image to alpine
    ```bash
    kubectl -n test edit rs frontend
    ```
    ```yaml
    spec:
      ...
      template:
        ...
        spec:
          containers:
          - name: alpine
            image: alpine
    ```
16. Get pods and check `AGE` in the namespace `test`
    ```bash
    kubectl -n test get po
    ```
17. Describe a replicaset pod and check the image
    ```bash
    kubectl -n test describe po frontend-<id>
    ```
18. Delete all pods and recheck the image in description
    ```bash
    kubectl -n test scale rs frontend --replicas 0 
    kubectl -n test get rs                # wait till the number of pods will get to 0
    kubectl -n test scale rs frontend --replicas 4
    kubectl -n test describe po frontend-<newId>
    ```    
19. Delete replicaset without deleting pods
    ```bash
    kubectl -n test delete rs frontend --cascade=orphan
    kubectl -n test get po
    kubectl -n test get rs
    ```
21. Recreate replica set
    ```bash
    kubectl -n test apply -f rs-test.yaml
    ```
22. Delete replicaset with pods
    ```bash
    kubectl -n test delete rs frontend
    kubectl -n test get po 
    kubectl -n test get rs
    ```
# 2. DeamonSet
1. Create daemonset (ds) descriptor  
    ```bash
    vim ds-test.yaml
    ```
    Paste the content nad save
    ```yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: test
      namespace: test
      labels:
        k8s-app: fluentd-logging
    spec:
      selector:
        matchLabels:
          name: fluentd-elasticsearch
      template:
        metadata:
          labels:
            name: fluentd-elasticsearch
        spec:
          tolerations:
          # these tolerations are to have the daemonset runnable on control plane nodes
          # remove them if your control plane nodes should not run pods
          - key: node-role.kubernetes.io/control-plane
            operator: Exists
            effect: NoSchedule
          - key: node-role.kubernetes.io/master
            operator: Exists
            effect: NoSchedule
          - key: CriticalAddonsOnly
            operator: Exists
            effect: NoExecute
          containers:
          - name: fluentd-elasticsearch
            image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
            volumeMounts:
            - name: varlog
              mountPath: /var/log
          terminationGracePeriodSeconds: 30
          volumes:
          - name: varlog
            hostPath:
              path: /var/log
    ```
2. Apply the file
   ```bash
   kubectl apply -f ds-test.yaml
   ```
3. List daemonset
   ```bash
   kubectl get ds -n test
   ```
4. Describe created daemonset
    ```bash
    kubectl -n test describe ds test
    ```
5. List pods in `test` namespace
   ```bash
   kubectl get po -n test -o wide
   ```

   <p><details><summary><b>Output</b></summary>

    ```
    NAME         READY   STATUS    RESTARTS   AGE   IP              NODE                 NOMINATED NODE   READINESS GATES
    test-gx7qr   1/1     Running   0          30s   10.42.165.95    k8s-plrnd-worker-1   <none>           <none>
    test-mkjkr   1/1     Running   0          35s   10.42.181.29    k8s-plrnd-worker-2   <none>           <none>
    test-w7gxk   1/1     Running   0          55s   10.42.137.48    k8s-plrnd-server-1   <none>           <none>
    ```
   </details></p>

6. Check labels of pods
   ```bash
   kubectl -n test get po --show-labels
   ```
7. Delete a pod `test-<any id>` then check the pods
    ```bash
    kubectl -n test delete pod test-<any id>
    kubectl -n test get po -o wide -w
    ```
    To stop watching press `Ctrl+c`
8. Add a new pod with label `name=fluentd-elasticsearch`
    ```bash
    kubectl -n test run test1 --image quay.io/fluentd_elasticsearch/fluentd:v2.5.2 --labels name=fluentd-elasticsearch
    kubectl -n test get po --show-labels -w
    ```
    Outcome:
    A new pod should not be created. Deamon set allows creating no more than 1 pod per node. The existance of the second one on a node is forbidden.
9. Describe daemonset 
    ```bash
    kubectl -n test describe ds test
    ```
10. Delete daemonset without removing pods
    ```bash
    kubectl -n test delete ds test --cascade=orphan
    kubectl -n test get po
    kubectl -n test get ds
    ```
11. Recreate daemonset
    ```bash
    kubectl -n test apply -f ds-test.yaml
    ```
12. Delete daemonset with pods
    ```bash
    kubectl -n test delete ds test
    kubectl -n test get po 
    kubectl -n test get ds
    ```
13. Create new daemonset `test-nodeselector`
    ```bash
    vim ds-nodeselector.yaml
    ```
    paste yaml to the file
    ```yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: test-nodeselector
      namespace: test
      labels:
        k8s-app: fluentd-logging
    spec:
      selector:
        matchLabels:
          name: fluentd-elasticsearch
      template:
        metadata:
          labels:
            name: fluentd-elasticsearch
        spec:
          tolerations:
          # these tolerations are to have the daemonset runnable on control plane nodes
          # remove them if your control plane nodes should not run pods
          - key: node-role.kubernetes.io/control-plane
            operator: Exists
            effect: NoSchedule
          - key: node-role.kubernetes.io/master
            operator: Exists
            effect: NoSchedule
          - key: CriticalAddonsOnly
            operator: Exists
            effect: NoExecute
          containers:
          - name: fluentd-elasticsearch
            image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
            volumeMounts:
            - name: varlog
              mountPath: /var/log
          nodeSelector:                   #nodeSelector
            disk: ssd
          terminationGracePeriodSeconds: 30
          volumes:
          - name: varlog
            hostPath:
              path: /var/log
    ```
14. Apply daemon set declaration
    ```bash
    kubectl apply -f ds-nodeselector.yaml
    ```
15. Get ds `test-nodeselector` and check number of pods
    ```bash
    kubectl -n test get ds test-nodeselector
    ```
16. Set label disk=ssd to a node `worker`
    ```bash
    kubectl label nodes worker disk=ssd
    ```
17. Get pods 
    ```bash
    kubectl -n test get po -o wide
    ```
18. Delete ds `test-nodeselector`
# 3. Deployment
1. Create deployment with `command`
   ```bash
   kubectl -n test create deployment test-deployment --image=nginx:1.26 --replicas=10 --port=80
   kubectl -n test get deployment.apps
   kubectl -n test get pod --show-labels
   ```
   alternative way is to use yaml file
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: test-deployment
     labels:
       app: nginx
   spec:
     replicas: 10
     minReadySeconds": 10
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: nginx:1.22
           ports:
           - containerPort: 80
   ```
2. Get history of rollouts
   ```bash
   kubectl -n test rollout history deployment test-deployment
   ```
   <p><details><summary><b>Output</b></summary>

   ```
   deployment.apps/test-deployment 
   REVISION  CHANGE-CAUSE
   1         <none>
   ```
   </details></p>
3. Set `change-cause` in revision history
   ```bash
   kubectl -n test  annotate deployment/test-deployment kubernetes.io/change-cause="Initial deploy, version 1.22"
   ```
4. Check again rollout history
   <p><details><summary><b>Output</b></summary>
   
   ```
   deployment.apps/test-deployment 
   REVISION  CHANGE-CAUSE
   1         Initial deploy, version 1.22
   ```
   </details></p>
5. Update image to version 1.23
   ```bash
   kubectl -n test patch deployment test-deployment -p '{"spec": {"minReadySeconds": 10}}'
   kubectl -n test annotate deployment/test-deployment kubernetes.io/change-cause="Update minReadySeconds: 10"
   kubectl -n test rollout history deployment test-deployment
   kubectl -n test set image deployment/test-deployment nginx=nginx:1.23
   kubectl -n test annotate deployment/test-deployment kubernetes.io/change-cause="Update image to 1.23"
   kubectl -n test get pod -w   #when finishes update press Ctrl+c
   kubectl -n test rollout status deployment/test-deployment
   ```
6. To see details of revision 2

   ```bash
   kubectl -n test rollout history deployment/test-deployment --revision=2
   ```
   <p><details><summary><b>Output</b></summary>
   
   ```yaml
   deployment.apps/test-deployment with revision #3
   Pod Template:
     Labels:       app=test-deployment
           pod-template-hash=d99c7466d
     Annotations:  kubernetes.io/change-cause: image updated to 1.23
     Containers:
      nginx:
       Image:      nginx:1.23
       Port:       <none>
       Host Port:  <none>
       Environment:        <none>
       Mounts:     <none>
     Volumes:      <none>
   ```
   </details></p>
7. Scale deployment `test-deployment` to 3
   ```bash
   kubectl -n test scale deployment test-deployment --replicas 3
   kubectl -n test annotate deployment/test-deployment kubernetes.io/change-cause="replicas 3"
   ```
8. Check resplicaSet and pods
   ```bash
   kubectl -n test get rs
   kubectl -n test get po --show-labels
   ```
9. Check description of deployment
   ```bash
   kubectl -n test describe deployment test-deployment
   ```
10. Rollback to revision 1
    ```bash
    kubectl -n test rollout undo deployment/test-deployment --to-revision=1
    kubectl -n test annotate deployment/test-deployment kubernetes.io/change-cause="rollback to rev=1"
    kubectl -n test rollout status deployment/test-deployment
    kubectl -n test rollout history deployment/test-deployment
    ```
11. Pause rollout to avoid immedaite change of the rollout
    ```bash
    kubectl -n test rollout pause deployment/test-deployment
    kubectl -n test scale deployment/test-deployment --replicas 10
    kubectl -n test set image deployment/test-deployment nginx=nginx:1.24
    kubectl -n test get po -w
    kubectl -n test annotate deployment/test-deployment kubernetes.io/change-cause="set image version to 1.24"
    kubectl -n test rollout history deployment/test-deployment
    ```
12. Resume deployment and check the replicaSet
    ```bash
    kubectl -n test rollout resume deployment/test-deployment
    kubectl -n test get rs -w     # when finishes press Ctrl+c
    kubectl -n test get po --show-labels
    ```
13. List events in a namespace `test`
    ```bash
    kubectl -n test get events
    ```
    Output:
    List of events is displayed.

## 3.1. Deployment strategies
1. Create yaml descriptor of `test-deployment` to file `recreate-deployment.yaml`
   ```bash
   kubectl -n test get deployment test-deployment -o yaml > recreate-deployment.yaml
   ```
2. Open descriptor file `recreate-deployment.yaml`
   ```bash
   vim recreate-deployment.yaml
   ```
3. Change in the descriptor name of the deployment into `recreate-deployment` and change `spec.strategy.type` into `Recreate`
   ```yaml
   metadata:
     name: recreate-deployment
   ```
4. Change strategy into  `Recreate` and make sure image is in version `nginx:1.22`, remove `RollingUpdate` entry with subentries `maxUnavailable` and `maxSurge`
   ```yaml
      spec:
      ...
        image: nginx:1.22
      ...
      RollingUpdate:                  # REMOVE THOSE LINES
        maxUnavailable: 25%           #
        maxSurge: 25%                 #
      ...
        strategy:
          type: Recreate
   ```
   Apply manifest:
   ```
   kubectl apply -f recreate-deployment.yaml
   ```
5. Set image `nginx:1.26` and list pods with `--wait`

   ```bash
   kubectl -n test set image deployment/recreate-deployment nginx=nginx:1.26
   kubectl -n test get po --show-labels -w
   ```
   Output
   All pods should be removed at the same time and recreated after all pods are terminated.
### 3.1.1. Check ReplicaSet
1. List all replicaset and find the one that are a subset of deployment.
   ```bash
   kubectl -n test get rs
   ```
   Output:
   There should be ReplicaSets which are named as deployments.
2. Remove deployment
   ```bash
   kubectl -n test delete deployment recreate-deployment
   ```
3. Check pods
   ```bash
   kubectl -n test get pod -w
   ```
4. When all pods are deleted check if replicaset still exists
   ```bash
   kubectl -n test get rs
   ```
# 4. Cleanup
1. Remove any deployments and replicasets created in this module.
   ```bash
   kubectl -n test get all
   ```
   Output should list all resources in teh namespace `test`
   ```bash
   kubectl -n test delete rs <rs_name1> <rs_name2>
   kubectl -n test delete deployment <depl_name1> <depl_name2>
   ```
   
