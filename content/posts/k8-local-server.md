---
author: "Saket Gupta"
title: "Access K8 cluster from Local Machine via Jump Server"
date: 2022-01-29T23:25:56+05:30
draft: false
description: ""
tags: [
    "Kubernetes",
]
---

There are times when you want to access K8 cluster hosted on remote machines on your local system. If this K8 cluster is using AWS EKS, Azure AKS, or Google GKE it is common practice to have a bastion host which is used to access this cluster.

To solve this problem, I am accessing these cluster via a SOCKS5 proxy. The problem with SOCKS5 proxy is that *HTTP Rest* based (get, describe, etc.) will work out of the box but commands using *SPDY2* (exec, attach, port-forward) won't work. 

There is already [PR](https://github.com/kubernetes/kubernetes/pull/105632) which will migrate these commands from SPDY2 to websockets. Till this gets released we will explore methods using kubectl with or without this functionality.
## Using Socks5 Proxy and old kubectl
This method requires Polipo as http proxy to wrap SOCKS5. 

<!-- Refer this [link](https://github.com/kubernetes/kubernetes/issues/58065). -->

In this menthod polipo converts SOCKS5 proxy to HTTP proxy

### Requirements
- Polipo
- kubectl

Polipo cannot be installed on Mac using brew as it has been deprecated. 

To install follow the following steps:
1. Edit Polipo's brew config
    ```console
    brew edit polipo
    ```
2. Polipo.rb file will open in your default editor. Remove the line starting with **disable!** and save.
3. Install polipo using brew
    ```console
    brew install polipo
    ```
4. Revert changes done in Step 2 and save.

### Steps to Follow
- Create a SOCKS5 tunnel to jump server
    ```console
    ssh -D 8013 -qCN [jump_server] 
    
    ```
- Create HTTP proxy using polipo
    ```console
    polipo \
        socksProxyType=socks5 \
        socksParentProxy=127.0.0.1:8013  \
        proxyPort=7770 \
        logFile=/dev/stdout \
        logLevel=0xFF
    ```

- Add server config in .kube/config. For the specific server add proxy port as below:
    ```yaml
    apiVersion: v1
    - cluster:
        certificate-authority-data: [certificate]
        server: https://xyz.ab7.us-east-1.eks.amazonaws.com
        proxy-url: socks5://localhost:8013
    name: arn:aws:eks:us-east-1:1234:cluster/test-cluster
    contexts:
    - context:
        cluster: arn:aws:eks:us-east-1:1234:cluster/test-cluster
        user: arn:aws:eks:us-east-1:1234:cluster/test-cluster
    name: arn:aws:eks:us-east-1:1234:cluster/test-context
    current-context: arn:aws:eks:us-east-1:1234:cluster/test-context
    kind: Config
    preferences: {}
    users:
    - name: arn:aws:eks:us-east-1:1234:cluster/test-cluster
    user:
        exec:
        apiVersion: client.authentication.k8s.io/v1alpha1
        provideClusterInfo: true
        args:
        - --region
        - us-east-1
        - eks
        - get-token
        - --cluster-name
        - test-context
        command: "aws"
        env: null
    ```
- **(optional)** If you are using AWS EKS and do not have your IAM role added to cluster you can use your jump server's credential by using *aws-cli* over ssh.
For this follow steps below:
    - Create a file in your path
        ```console
        touch /usr/local/bin/awsn
        chmod +x /usr/local/bin/awsn
        ```
    - Add following contents
        ```bash
        #!/bin/sh
        ssh [jump_server] aws "$@"
        ```
    - Add following config file
        ```yaml
        apiVersion: v1
        - cluster:
            certificate-authority-data: [certificate]
            server: https://xyz.ab7.us-east-1.eks.amazonaws.com
            proxy-url: socks5://localhost:8013
        name: arn:aws:eks:us-east-1:1234:cluster/test-cluster
        contexts:
        - context:
            cluster: arn:aws:eks:us-east-1:1234:cluster/test-cluster
            user: arn:aws:eks:us-east-1:1234:cluster/test-cluster
        name: arn:aws:eks:us-east-1:1234:cluster/test-context
        current-context: arn:aws:eks:us-east-1:1234:cluster/test-context
        kind: Config
        preferences: {}
        users:
        - name: arn:aws:eks:us-east-1:1234:cluster/test-cluster
        user:
            exec:
            apiVersion: client.authentication.k8s.io/v1alpha1
            provideClusterInfo: true
            args:
            - --region
            - us-east-1
            - eks
            - get-token
            - --cluster-name
            - test-context
            command: "awsn"
            env: null
        ```
- Use kubectl commands as follows. Here **7770** is the proxy port setup in polipo.
    ```console
    env HTTPS_PROXY=http://127.0.0.1:7770 kubectl exec [pod_name] -n [namespace_name] -- [command]
    ```


## Using SOCKS5 proxy and new kubectl

### Requirements
- kubectl (with this [PR](https://github.com/kubernetes/kubernetes/pull/105632))
- go compiler (if kubectl needs to be build)

 If latest kubectl is not available use following method:

 ```shell
 git clone https://github.com/kubernetes/kubernetes.git
 # Build kubectl
 go build -o ~/bin/kubectl ./cmd/kubectl
 # Create symlink in your path
 sudo ln -s ~/bin/kubectl /usr/local/bin/kubectl
 ```

 ### Steps to Follow
 - Create SOCKS5 tunnel
    ```console
    ssh -D 8013 -qCN [jump_server]
    ```
- Run kubectl commands as follows
  ```console
  kubectl exec [pod_name] -n [namespace_name] -- [command]
  ```

## Conclusion
Here we talked about how to access remote cluster from jump servers. In future article, we will explore on how to test your source code inside K8 cluster.
Till that time Happy Coding!!
