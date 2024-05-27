---
title: "Kubernetes training"
author: [Szymon Byczek]
date: "17/05/2024"
version: 0.1
status: "Draft"
copyright year: "2024"
---
**Content:**
- [1. Read docs (read-docs)](#1-read-docs-read-docs)
- [2. Install RKE2](#2-install-rke2)
  - [2.1. Install server node](#21-install-server-node)
  - [2.2. Install worker node](#22-install-worker-node)
- [3. Install kubectl](#3-install-kubectl)
- [4. Get kubeconfig](#4-get-kubeconfig)
- [5. Kubectl autocomplete](#5-kubectl-autocomplete)
- [6. kubectl help](#6-kubectl-help)
- [7. First pod](#7-first-pod)
  - [7.1. List logs](#71-list-logs)
  - [7.2. Opening console in a pod](#72-opening-console-in-a-pod)
- [8. Labeling](#8-labeling)
  - [8.1. Set label](#81-set-label)
  - [8.2. Labeling nodes](#82-labeling-nodes)
- [9. Namespaces](#9-namespaces)
  - [9.1. Creating namespaces](#91-creating-namespaces)
  - [9.2. Listing pods in all namespaces](#92-listing-pods-in-all-namespaces)
  - [9.3. Managing objects in other namespaces](#93-managing-objects-in-other-namespaces)
  - [9.4 Port-Forward](#94-port-forward)
- [10. Exposing pods - services](#10-exposing-pods---services)


# 1. Read docs (read-docs)

1. Open web browser
2. Open URL address: [https://kubernetes.io](https://kubernetes.io)
3. Select `Documentation` in the top menu. Links on the left side can be helpful in navigation
4. Please check `Tasks, `Tutorials`, and `Getting Started``

# 2. Install RKE2

## 2.1. Install server node

1. Connect via SSH server node machine
2. Optional - create a file with proxy settings

   ```bash
   student@server$ cat <<EOF | sudo tee /etc/default/rke2-server
   HTTP_PROXY=http://<proxy-url>:<port>
   HTTPS_PROXY=<proxy-url>:<port>
   NO_PROXY=127.0.0.0/8,10.42.0.0/16,10.43.0.0/16,.svc,.cluster.local
   CONTAINERD_HTTP_PROXY=<proxy-url>:<port>
   CONTAINERD_HTTPS_PROXY=<proxy-url>:<port>
   CONTAINERD_NO_PROXY=127.0.0.0/8,10.42.0.0/16,10.43.0.0/16,.svc,.cluster.local
   EOF
   ```

3. Run server installer
This installs the rke2-server service on Linux.

4. Enable the rke2-server service
   
   ```bash
   root@server$ systemctl enable rke2-server.service
   ```

5. Start rke2-server.service
   
   ```bash
   root@server$ systemctl start rke2-server.service
   ```

6. Check logs
   
   ```bash
   root@server$ journalctl -u rke2-server -f
   ```

7. Get token
   
   ```bash
   root@server$ cat /var/lib/rancher/rke2/server/node-token
   ```

## 2.2. Install worker node
1. Connect via SSH to the worker node machine
2. Create a file with proxy settings

   ```bash
   student@agent$ cat <<EOF | sudo tee /etc/default/rke2-agent
   HTTP_PROXY=http://<proxy-url>:<port>
   HTTPS_PROXY=http://<proxy-url>:<port>
   NO_PROXY=127.0.0.0/8,10.42.0.0/16,10.43.0.0/16,.svc,.cluster.local
   CONTAINERD_HTTP_PROXY=<proxy-url>:<port>
   CONTAINERD_HTTPS_PROXY=<proxy-url>:<port>
   CONTAINERD_NO_PROXY=127.0.0.0/8,10.42.0.0/16,10.43.0.0/16,.svc,.cluster.local
   EOF
   ```

3. Run agent installer
   
   ```bash
   student@agent$ curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -
   ```
   **NOTE** This installs rke2-server service on Linux.

4. Enable the rke2-server service
   
   ```bash
   student@agent$ sudo systemctl enable rke2-agent.service
   ```

5. Configure the rke2-agent service
   
   ```bash
   student@agent$ sudo mkdir -p /etc/rancher/rke2/ 
   student@agent$ sudo vim /etc/rancher/rke2/config.yaml
   ```
   Paste below content:
   
   ```bash
   server: https://<server>:9345
   token: <token from server node>
   ```
   **NOTE**: The rke2 server process listens on port 9345 for new nodes to register. The Kubernetes API is still served on port 6443, as normal. Please change `<server>` to rke2 server IP.
   
6. Start rke2-agent.service
   
   ```bash
   student@agent$ sudo systemctl start rke2-agent.service
   ```

7. Check logs
   
   ```bash
   student@agent$ sudo journalctl -u rke2-agent -f
   ```

# 3. Install kubectl
1. Connect via SSH to the server machine
2. Install via `binary` needed packages. Other methods of installing can be found at https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux 
    
   ```bash
   student@server$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   student@server$ curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
   student@server$ echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
   ```

   <p><details><summary><b>Output</b></summary>

   ```
   kubectl: OK
   ```
   </details></p>
    
   ```bash
   student@server$ sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
   ```

# 4. Get kubeconfig
1. Find the kubeconfig on the server node. See the path `/etc/rancher/rke2/rke2.yaml`
    
   ```bash
   student@server$ mkdir -p $HOME/.kube
   student@server$ sudo cp -i /etc/rancher/rke2/rke2.yaml $HOME/.kube/config
   student@server$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```
2. Create env variable `KUBECONFIG`
   
   ```bash
   export KUBECONFIG=$HOME/.kube/config
   ```
3. Check if a connection to the cluster is working
   INFO: from now on use a console, where kubectl is installed.

   ```bash
   kubectl cluster-info
   ```

   <p><details><summary><b>Output</b></summary>

    ```
    Kubernetes control plane is running at https://127.0.0.1:6443
    CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/rke2-coredns-rke2-coredns:udp-53/proxy

    For further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
    ```
   </details></p>

   ```bash
   kubectl get nodes
   ```

   <p><details><summary><b>Output</b></summary>

    ```
    NAME               STATUS   ROLES                       AGE     VERSION
    server             Ready    control-plane,etcd,master   3d22h   v1.24.9+rke2r2
    agent              Ready    <none>                      3d10h   v1.24.9+rke2r2
    ```
   </details></p>

   Get detailed info about the node

   ```bash
   kubectl describe nodes server
   ``` 

   <p><details><summary><b>Output</b></summary>
    
    ```
    Name:               k8s-plrnd-server-1
    Roles:              control-plane,etcd,master
    Labels:             beta.kubernetes.io/arch=amd64
                        beta.kubernetes.io/instance-type=rke2
                        beta.kubernetes.io/os=linux
                        egress.rke2.io/cluster=true
                        kubernetes.io/arch=amd64
                        kubernetes.io/hostname=k8s-plrnd-server-1
                        kubernetes.io/os=linux
                        node-role.kubernetes.io/control-plane=true
                        node-role.kubernetes.io/etcd=true
                        node-role.kubernetes.io/master=true
                        node.kubernetes.io/instance-type=rke2
    Annotations:        etcd.rke2.cattle.io/node-address: 192.168.1.176
                        etcd.rke2.cattle.io/node-name: k8s-plrnd-server-1-    13b86b32
                        flannel.alpha.coreos.com/backend-data:     {"VNI":1,"VtepMAC":"32:76:ed:0c:ae:8a"}
                        flannel.alpha.coreos.com/backend-type: vxlan
                        flannel.alpha.coreos.com/kube-subnet-manager: true
                        flannel.alpha.coreos.com/public-ip: 192.168.1.176
                        node.alpha.kubernetes.io/ttl: 0
                        projectcalico.org/IPv4Address: 192.168.1.176/24
                        projectcalico.org/IPv4VXLANTunnelAddr: 10.42.137.0
                        rke2.io/encryption-config-hash: start-    f8b83041cfd6d2cc8d5592d3480bfa84ed907e1b700a74080cd840fd87e5ec22
                        rke2.io/hostname: k8s-plrnd-server-1
                        rke2.io/internal-ip: 192.168.1.176
                        rke2.io/node-args:
                          ["server","--token","********","--tls-san","k8s-my.company.local","--tls-san","k8s-plrnd","--tls-    san","192.168.1.101","--cni","cali...
                        rke2.io/node-config-hash:     ONUB3HMSZHTAYP2S7GQAS6EHCOADJYHTQ6D3TOYWKQIUN5VQJMUQ====
                        rke2.io/node-env: {}
                        volumes.kubernetes.io/controller-managed-attach-    detach: true
    CreationTimestamp:  Tue, 15 Nov 2022 10:49:23 +0100
    Taints:             CriticalAddonsOnly=true:NoExecute
    Unschedulable:      false
    Lease:
      HolderIdentity:  k8s-plrnd-server-1
      AcquireTime:     <unset>
      RenewTime:       Tue, 24 Jan 2023 22:38:27 +0100
    Conditions:
      Type                 Status  LastHeartbeatTime                     LastTransitionTime                Reason                       Message
      ----                 ------  -----------------                 ---------    ---------                ------                       -------
      NetworkUnavailable   False   Sat, 21 Jan 2023 13:47:34 +0100   Sat, 21     Jan 2023 13:47:34 +0100   CalicoIsUp                   Calico is running     on this node
      MemoryPressure       False   Tue, 24 Jan 2023 22:36:30 +0100   Sat, 21     Jan 2023 13:45:59 +0100   KubeletHasSufficientMemory   kubelet has     sufficient memory available
      DiskPressure         False   Tue, 24 Jan 2023 22:36:30 +0100   Sat, 21     Jan 2023 13:45:59 +0100   KubeletHasNoDiskPressure     kubelet has no disk     pressure
      PIDPressure          False   Tue, 24 Jan 2023 22:36:30 +0100   Sat, 21     Jan 2023 13:45:59 +0100   KubeletHasSufficientPID      kubelet has     sufficient PID available
      Ready                True    Tue, 24 Jan 2023 22:36:30 +0100   Sat, 21     Jan 2023 13:45:59 +0100   KubeletReady                 kubelet is posting     ready status. AppArmor enabled
    Addresses:
      InternalIP:  192.168.1.176
      Hostname:    my-server-1
    Capacity:
      cpu:                4
      ephemeral-storage:  264096568Ki
      hugepages-1Gi:      0
      hugepages-2Mi:      0
      memory:             8125220Ki
      pods:               110
    Allocatable:
      cpu:                4
      ephemeral-storage:  256913141149
      hugepages-1Gi:      0
      hugepages-2Mi:      0
      memory:             8125220Ki
      pods:               110
    System Info:
      Machine ID:                 d4d32285669b4059827fa0c09a3ff017
      System UUID:                69a880c6-cbc8-dc44-4613-92560000cb94
      Boot ID:                    855a7a16-6397-47d7-aba3-c84f8f6bf0a0
      Kernel Version:             5.4.0-137-generic
      OS Image:                   Ubuntu 20.04.5 LTS
      Operating System:           linux
      Architecture:               amd64
      Container Runtime Version:  containerd://1.6.8-k3s1
      Kubelet Version:            v1.24.7+rke2r1
      Kube-Proxy Version:         v1.24.7+rke2r1
    PodCIDR:                      10.42.0.0/24
    PodCIDRs:                     10.42.0.0/24
    ProviderID:                   rke2://k8s-plrnd-server-1
    Non-terminated Pods:          (9 in total)
      Namespace                       Name                                           CPU Requests  CPU Limits      Memory Requests  Memory Limits  Age
      ---------                   ---    -                                           ------------  ----------  ----    -----------  -------------  ---
      calico-system               calico-node-    kl8h4                              0 (0%)        0 (0%)      0     (0%)           0 (0%)         70d
      kube-system                 cloud-controller-manager-k8s-plrnd-server-    1    100m (2%)     0 (0%)      128Mi (1%)       0 (0%)         70d
      kube-system                 etcd-k8s-plrnd-server-    1                        200m (5%)     0 (0%)      512Mi (6%)       0     (0%)         70d
      kube-system                 kube-apiserver-k8s-plrnd-server-    1              250m (6%)     0 (0%)      1Gi (12%)        0 (0%)             70d
      kube-system                 kube-controller-manager-k8s-plrnd-server-    1     200m (5%)     0 (0%)      256Mi (3%)       0 (0%)         70d
      kube-system                 kube-proxy-k8s-plrnd-server-    1                  250m (6%)     0 (0%)      128Mi (1%)       0     (0%)         70d
      kube-system                 kube-scheduler-k8s-plrnd-server-    1              100m (2%)     0 (0%)      128Mi (1%)       0 (0%)             70d
      kube-system                 rke2-coredns-rke2-coredns-58fd75f64b-    j54fv     100m (2%)     100m (2%)   128Mi (1%)       128Mi (1%)     14h
      tigera-operator             tigera-operator-5dd8cf7c89-    sh8wt               0 (0%)        0 (0%)      0 (0%)           0     (0%)         70d
    Allocated resources:
      (Total limits may be over 100 percent, i.e., overcommitted.)
      Resource           Requests      Limits
      --------           --------      ------
      cpu                1200m (30%)   100m (2%)
      memory             2304Mi (29%)  128Mi (1%)
      ephemeral-storage  0 (0%)        0 (0%)
      hugepages-1Gi      0 (0%)        0 (0%)
      hugepages-2Mi      0 (0%)        0 (0%)
    Events:              <none>
    ```
   </details></p>

# 5. Kubectl autocomplete
 1. Update ~/.bashrc   
    ```bash
    source <(kubectl completion bash) # set up autocomplete in bash into the current shell, bash-completion package should be installed first.
    echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanently to your bash shell.
    ```
 2. Use TAB to complete a description of the node
    ```
    kubectl desc<TAB> n<TAB> 
    kubectl -n kube-s<TAB> g<TAB> po<TAB>
    ```
# 6. kubectl help
 1. Get general help
    ```
    kubectl -h
    ```
 2. Get help from a specific command
    ```
    kubectl create -h
    kubectl create service -h
    ```
# 7. First pod
 1. Run a command to create the first pod
    ```
    kubectl run test --image=nginx --port=80
    ```
 2. Check the status of the pod
    ```
    kubectl get pods
    ```
   <p><details><summary><b>Output</b></summary>

     NAME   READY   STATUS              RESTARTS   AGE 
     test   0/1     ContainerCreating   0          11s

   </details></p>

 3. Wait for output change. Parameter `-w` can be extended to `--watch`
    ```
    kubectl get pods -w
    ```

 4. Press `Ctrl+c` to abort watching

 5. Get additional info on pods, like IP and hosting node
    ```bash
    kubectl get pods -o wide
    ```
 6. Get detailed information about specific pods. Please check the status, node, and ports
    ```bash
    kubectl describe pod test
    ```
    It is feasible to use the `grep` command to filter information
    ```bash
    kubectl describe pod test | grep Status:
    ```
 7. List descriptor of a pod as YAML
    ```bash
    kubectl get pod test -o yaml
    ```
 8. Create a second pod with the name `test2` by creating a manifest file from the pod `test`
    ```bash
    kubectl get pod test -o yaml > test-pod.yaml
    cp test-pod.yaml test2-pod.yaml
    ```
    Edit file in any editor, e.g. `vim` or `nano`. Then remove unneeded data: `creationTimestamp`, `resourceVersion`, `uid`, and entire `status` section, then change the *name*
    ```bash
    vim test2-pod.yaml
    ```

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      annotations:
        cni.projectcalico.org/containerID: 57e7397e13437096e34778f4569a15e06cbb89a5fdaab042b9cbff1973889e4c
        cni.projectcalico.org/podIP: 10.42.165.78/32
        cni.projectcalico.org/podIPs: 10.42.165.78/32
        kubernetes.io/psp: global-unrestricted-psp
      creationTimestamp: "2023-01-24T20:35:41Z"
      labels:
        run: test                                  # <-- change to test2
      name: test                                   # <-- change to test2
      namespace: default
      resourceVersion: "38817794"                  # <-- remove this line
      uid: 7a96dcd8-3a17-4a7c-b8af-7f5b09351435    # <-- and this one too
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: test
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-kdxgj
          readOnly: true
      dnsPolicy: ClusterFirst
      enableServiceLinks: true
      nodeName: k8s-plrnd-worker-1
      preemptionPolicy: PreemptLowerPriority
      priority: 0
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: default
      serviceAccountName: default
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoExecute
        key: node.kubernetes.io/not-ready
        operator: Exists
        tolerationSeconds: 300
      - effect: NoExecute
        key: node.kubernetes.io/unreachable
        operator: Exists
        tolerationSeconds: 300
      volumes:
      - name: kube-api-access-kdxgj
        projected:
          defaultMode: 420
          sources:
          - serviceAccountToken:
              expirationSeconds: 3607
              path: token
          - configMap:
              items:
              - key: ca.crt
                path: ca.crt
              name: kube-root-ca.crt
          - downwardAPI:
              items:
              - fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
                path: namespace
    status:                         # <-- remove everything till the end
      conditions:
      - lastProbeTime: null
        lastTransitionTime: "2023-01-24T20:35:41Z"
        status: "True"
        type: Initialized
      - lastProbeTime: null
        lastTransitionTime: "2023-01-24T20:35:49Z"
        status: "True"
        type: Ready
      - lastProbeTime: null
        lastTransitionTime: "2023-01-24T20:35:49Z"
        status: "True"
        type: ContainersReady
      - lastProbeTime: null
        lastTransitionTime: "2023-01-24T20:35:41Z"
        status: "True"
        type: PodScheduled
      containerStatuses:
      - containerID: containerd://f2e854ee4d2508ed66b736476aa85be586f6545fba5dbf504b1e2703ba8e245e
        image: docker.io/library/nginx:latest
        imageID: docker.io/library/nginx@sha256:b8f2383a95879e1ae064940d9a200f67a6c79e710ed82ac42263397367e7cc4e
        lastState: {}
        name: test
        ready: true
        restartCount: 0
        started: true
        state:
          running:
            startedAt: "2023-01-24T20:35:49Z"
       hostIP: 192.168.1.190
       phase: Running
       podIP: 10.42.165.78
       podIPs:
       - ip: 10.42.165.78
       qosClass: BestEffort
       startTime: "2023-01-24T20:35:41Z"        # <-- 
    ```
1. Create a pod from a manifest. Use 1<sup>st</sup> or 2<sup>nd</sup> command:
    
    ```bash
    kubectl create -f test2-pod.yaml
    ```
    or

    ```bash
    kubectl apply -f test2-pod.yaml
    ```

    <p><details><summary><b>Output</b></summary>
    
    ```
    pod/test2 created
    ```
    </details></p>
2.  Check if a pod is created, on which node, and what is the IP
    ```bash
    kubectl get pods -o wide
    ```
3.  Delete pod `test2` either 1st or 2nd command
    ```bash
    kubectl delete pod test2
    ```
    or 
    ```bash
    kubectl delete -f test2-pod.yaml
    ```
4.  Recreate pod `test2` and recheck the IP

    ```bash
    kubectl create -f test-pod2.yaml
    kubectl get pod -o wide
    ```
    IP should be different

5.  Delete pod `test2`

## 7.1. List logs

1. List logs of the `test` pod
   ```bash
   kubectl logs pods/test 
   ```
   
   In case of displaying logs of a specific container in a pod, put its name after the pod's name
   ```bash
   kubectl logs pods/test test
   ```

## 7.2. Opening console in a pod
1. To open consol in existing pod use command `exec` and run shell command after `--`
   ```bash
   kubectl exec pods/test -it -- /bin/bash
   root@test:/# ls
   ```
   
   Exit from the pod by `exit` command.

   ```bash
   root@test:/# exit
   ```
2. Sometimes there is a need to create a pod, run the command, and delete the pod, e.g. for debugging
   ```bash
   kubectl run -it --rm --restart=Never busybox --image=gcr.io/google-containers/busybox -- hostname
   ```
   `--rm` removes the pod just after the command is executed.

   `--restart=Never` changes restarting policy from `Always` (default value) to `Never`. With Never the exit code of the container process is returned.

  <p><details><summary><b>Output</b></summary>

  ```
  busybox
  pod "busybox" deleted
  ```
  </details></p>

# 8. Labeling

## 8.1. Set label

1. Set label to an existing pod with `keys: app, rel`, and `values: web, test`
   ```bash
   kubectl label pod test app=web rel=test
   ```
2. Describe the pod and check the labels

   ```bash
   kubectl describe po test
   ```
   <p><details><summary><b>Output</b></summary>

   ```
   Name:         test
   Namespace:    default
   Priority:     0
   Node:         k8s-plrnd-worker-1/192.168.1.190
   Start Time:   Thu, 26 Jan 2023 07:20:03 +0100
   Labels:       app=curl
                 rel=prod
                 run=test
   ```
  </details></p>

3. Show labels
   ```bash
   kubectl get pod --show-labels
   ```
   <p><details><summary><b>Output</b></summary>

   ```
   NAME   READY   STATUS    RESTARTS   AGE     LABELS
   test   1/1     Running   0          7d15h   app=curl,rel=prod,run=test
   ```
   </details></p>

4. Show label values by key
   ```bash
   kubectl get pod -L app,rel
   ```
   <p><details><summary><b>Output</b></summary>

   ```
   NAME   READY   STATUS    RESTARTS   AGE     APP    REL  
   test   1/1     Running   0          7d15h   curl   prod
   ```
   </details></p>

5. Create descriptor from pod `test`
   ```bash
   kubectl get pod test -o yaml > pod-labels.yaml
   ```

6. Edit `pod-labels.yaml`  change value `prod` into the `canary`

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     annotations:
       cni.projectcalico.org/containerID:    af4172c0c4edc164d11eb2b51606f92d4f8d2d776fa889ac565016e2df63789c
       cni.projectcalico.org/podIP: 10.42.165.124/32
       cni.projectcalico.org/podIPs: 10.42.165.124/32
       kubernetes.io/psp: global-unrestricted-psp
     creationTimestamp: "2023-01-26T06:20:03Z"
     labels:
       app: curl
       rel: prod              <-- Change into canary
       run: test
     name: test
     namespace: default
   ```
7. Apply descriptor change
   ```bash
   kubectl apply -f pod-labels.yaml
   ```
   Simillar action can be achieved by this command
   ```bash
   kubectl label pod rel=canary --overwrite
   ```
   <p><details><summary><b>Output</b></summary>

   ```
   pod/test labeled
   ```
   </details></p>

8. Check the description of the pod `test`
   ```bash
   kubectl describe po test | grep rel:
   ```
9. Copy pod descriptor 
   ```bash
   cp pod-labels.yaml test2-labels.yaml
   vim test2-labels.yaml
   ```
   Change `metadata.name` to *test2* and set label `rel=prod`, then save the file

10. Create pod `test2`
    ```bash
    kubectl create -f test2-labels.yaml
    ```

11. List pods with labels
    ```bash
    kubectl get po --show-labels
    ```
    <p><details><summary><b>Output</b></summary>

    ```
    NAME    READY   STATUS    RESTARTS   AGE     LABELS
    test    1/1     Running   0          7d16h   app=web,rel=canary,run=test
    test2   1/1     Running   0          99s     app=web,rel=prod,run=test2
    ```
    </details></p>

12. Display labels by key rel
    ```bash
    kubectl get po -L rel
    ```
13. Filter pods by key and value
    ```bash
    kubectl get po -l rel=prod
    ```
14. Filter pods by key
    ```bash
    kubectl get po -l rel
    ```
15. Label pod `test2` with key env=prod
    ```bash
    kubectl label pod test2 env=prod
    ```
16. Select all pods with key label `env`
    ```bash
    kubectl get po -l env
    ```
    <p><details><summary><b>Output</b></summary>

    ```
    NAME    READY   STATUS    RESTARTS   AGE
    test2   Running      0         17h
    ```
    </details></p>

17. Select all pods without key label `env` 
    ```bash
    kubectl get pod -l '!env' 
    ```
    **! has to be put in single quotes**

18. Change query to use `'rel in (prod,dev)'`
19. Change query to use `'rel notin (prod)'`
20. Change query to use `'rel!=prod'`
21. Change query to use `rel=prod`
22. Change query to use two conditions `rel=prod,app=web`
23. Delete pod with label `run=test2`

    ```
    kubectl delete po -l run=test2
    ```

## 8.2. Labeling nodes
1. Label agent node with `gpu=true`. If needed list nodes with `kubectl get nodes` 
   ```bash
   kubectl label node <agent name> gpu=true
   ```
2. List nodes with the label 'gpu=true'
   ```bash
   kubectl get nodes -l gpu=true
   ```
# 9. Namespaces

1. Get the list of namespaces
   ```bash
   kubectl get ns
   ```
   alternatively
   ```bash
   kubectl get namespaces
   ```
   <p><details><summary><b>Output</b></summary>

   ```
   NAME                   STATUS   AGE
   calico-system          Active   80d
   cattle-system          Active   74d
   default                Active   80d
   kube-system            Active   80d
   ...
   ```
   </details></p>

2. Get all pods in namespace `kube-system`
   ```bash
   kubectl -n kube-system get po
   ```
   Instead of `-n` user can use `--namespace` just after kubectl or at the end.
3. Select pod Kubernetes with autocomplete
   ```bash
   kubectl -n kub<tab>-sy<tab> get po ku<tab>
   ```
   Try to do the same thing with setting namespace at the end 

## 9.1. Creating namespaces
1. Create the namespace `test` with a command

   ```bash
   kubectl create ns test
   ```
2. Create namespace `test2` with yaml descriptor
   ```bash
   kubectl create ns test -o yaml --dry-run=client > ns-test2.yaml
   ```
3. Open content of ns-test2.yaml
   ```bash
   vim ns-test2.yaml
   ```
4. Change value in `metadata.name` into `test2` and save the file.
5. Apply the file
   ```bash
   kubectl create -f ns-test2.yaml 
   ```
6. List all namespaces and check if new namespaces `test` and `test2` are created
   ```bash
   kubectl get ns
   ```
7. Delete namespace `test2`
   ```bash
   kubectl delete ns test2
   ```
   **Deleting a namespace can be done only when there are no resources in it.**
8. Try to create a namespace with: `.`, `-`, digits.
9. Delete those namespaces (from the previous point) with space-separated names. Use the autocomplete feature.
   ```bash
   kubectl delete ns namespace1 namespace2
   ```
## 9.2. Listing pods in all namespaces
1. Get pods in all namespaces
   ```bash
   kubectl get po -all-namespaces
   ```
   ** Instead of `--all-namespaces` `-A` can be used
   ```bash
   kubectl get po -A
   ```

## 9.3. Managing objects in other namespaces

1. Create a pod in the namespace `test`
   ```bash
   kubectl -n test run test --image nginx --port 80
   ```
2. Create a file `test-test-pod.yaml` with content
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: test2
   spec:
     containers:
     - image: nginx
       imagePullPolicy: Always
       name: test
       ports:
       - containerPort: 80
         protocol: TCP
   ```
3. Apply the file in the namespace `test`
   ```bash
   kubectl -n test apply -f test-test-pod.yaml
   ```
4. Check pods in the namespace `test`
   ```bash
   kubectl -n test get pod
   ```
5. Change kubectl context to namespace `test`
   ```bash
   kubectl config set-context --current --namespace=test
   ```
6. List pods
   ```bash
   kubectl get po
   ```
   All pods from the `test` namespace should be returned
7. Using aliases
   ```bash
   alias kcd='kubectl config set-context --current'
   ```
   Use an alias to change the default namespace
   ```bash
   kcd --namespace=default
   ```
8. Check current namespace
   ```bash
   kubectl config current-context
   ```
   <p><details><summary><b>Output</b></summary>

   ```
   default
   ```
   </details></p>
## 9.4 Port-Forward
1. Create port-forward connection to pod `test`
   ```bash
   kubectl -n test port-forward pod/test 8000:80
   ```
2. Open a new console in the `server` node and curl to `localhost` on port `8000`
   ```bash
   curl 127.0.0.1:8000
   ```
   Close the console with curl
   Close port-forward on the first terminal with `Ctrl+c`

# 10. Exposing pods - services
1. Check help and take a look at examples
   ```bash
   kubectl expose pod --help
   ```

2. Create service with type `NodePort`
   ```bash
   kubectl expose pod test --type NodePort
   ```
3. Get all services 
   ```bash
   kubectl get service
   ```
   
   <p><details><summary><b>Output</b></summary>

   ```
   NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE     SELECTOR
   test          NodePort    10.43.188.56    <none>        80:31202/TCP,443:32407/TCP   2m39s   name=test
   ```
   </details></p>

4. Check the description of the service `test`
   ```bash
   kubectl describe svc test
   ```

   <p><details><summary><b>Output</b></summary>

   ```
   Name:              test
   Namespace:         default
   Labels:            run=test
   Annotations:       <none>
   Selector:          run=test
   Type:              ClusterIP
   IP Family Policy:  SingleStack
   IP Families:       IPv4
   IP:                10.43.24.23
   IPs:               10.43.24.23
   Port:              http  80/TCP
   TargetPort:        80/TCP
   Endpoints:         10.42.165.124:80
   Session Affinity:  None
   Events:            <none>
   ```
   </details></p>

5. Check if a user can connect to the pod via service

   ```bash
   curl --noproxy '*' http://<server-ip>:31202
   ```


