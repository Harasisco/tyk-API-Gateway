# What-is-an-API-gateway?

<p> An API gateway is a server that acts as an API front-end, receiving API requests, enforcing throttling and security policies, passing requests to the back-end service, and then passing the response back to the requester. </p>

## What is the difference between API gateway and reverse proxy?
<p> While both serve as intermediaries between clients and servers, an API Gateway is specialized for API management tasks, whereas a reverse proxy focuses on routing and optimizing traffic flow.</p>

# How to deploy a tyk-oss API Gateway

## Prerequisites

Before proceeding, ensure you have the following installed:

- Minikube
- Helm
- kubectl
- docker
- Ansible (Optional)

You can use my personal Ansible code to download all the dependencies:
```yaml
---
- name: Install Docker, Minikube, Kubernetes, and Helm
  hosts: localhost
  become: yes
  tasks:
    - name: Install epel-release
      when: ansible_pkg_mgr == "yum"
      package:
        name: epel-release
        state: present

    - name: Install dependencies
      package:
        name: "{{ item }}"
        state: present
      loop:
        - curl
        - wget
        - snapd

    - name: Start and enable snapd service
      service:
        name: snapd
        state: started
        enabled: yes

    - name: Restart snapd service
      service:
        name: snapd
        state: restarted

    - name: Add /snap/bin to PATH if snapd is installed
      shell: 'echo $PATH | grep -q "/snap/bin" || export PATH="$PATH:/snap/bin"'
    - name: To enable _classic_ snap support, create a symbolic link
      shell: 'ln -s /var/lib/snapd/snap /snap'

    - name: Install Docker if not already installed
      command: "curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh"
      args:
        creates: /usr/bin/docker
      ignore_errors: yes

    - name: Add user to docker group
      shell: 'usermod -aG docker $USER'

    - name: Start and enable Docker service
      service:
        name: docker
        state: started
        enabled: yes

    - name: Restart Docker service
      service:
        name: docker
        state: restarted

    - name: Install Minikube
      block:
        - name: Download Minikube
          get_url:
            url: "https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64"
            dest: "/usr/local/bin/minikube"
            mode: 0755

    - name: Install kubectl
      block:
        - name: Download kubectl
          shell: "curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl"
        - name: Move kubectl to /usr/local/bin
          shell: "chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl"

    - name: Install Helm
      block:
        - name: Install Helm snap
          snap:
            name: helm
            classic: yes

    - name: reset ssh connection to allow user changes to affect 'current login user'
      meta: reset_connection

``` 
## Steps

### 1. Start Minikube

- Start Minikube with increased disk size and unlimited memory:

```shell
minikube start --disk-size 50000mb --memory='no-limit'
```
### 2. Add Tyk and Bitnami Helm Repositories
```shell
helm repo add tyk-helm https://helm.tyk.io/public/helm/charts/
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 3. Generate Tyk Configuration

- Generate Tyk configuration values file:

```shell
helm show values tyk-helm/tyk-oss > values.yaml
```
- and Edit the file to uncomment the Redis addrs and password
```YAML
  redis:
    # The addrs value will allow you to set your Redis addresses.
    #
    # If you are using Redis (e.g. Bitnami Redis at bitnami/redis) then enter single
    # endpoint. If using sentinel connection mode for Redis, please update the port number (typically 26379).
    #
    # If using a Redis Cluster (e.g. bitnami/redis-cluster), you can list
    # the endpoints of the redis instances or use the cluster configuration endpoint.
    #
    # Default value: redis.{{ .Release.Namespace }}.svc:6379
    addrs:
    #   Example using tyk simple redis chart
    #   - redis.tyk.svc:6379
    #   Example using bitnami/redis
    - tyk-redis-master.tyk.svc:6379
    #   Example using bitnami/redis with sentinel
    #   - tyk-redis.tyk.svc:26379
    #   Example using bitnami/redis-cluster
    #   - tyk-redis-redis-cluster.tyk.svc:6379

    # Redis password
    # If you're using Bitnami Redis chart (e.g. bitnami/redis) please input
    # your password in the field below
    pass: ""
```
### 4. Create Kubernetes Namespace for Tyk
```shell
kubectl create namespace tyk
```

### 5. Install Redis for Tyk
- Install Redis using Bitnami Helm chart:
```shell
helm install tyk-redis bitnami/redis -n tyk
```

- Retrieve the Redis password:
```shell
export REDIS_PASSWORD=$(kubectl get secret --namespace tyk tyk-redis -o jsonpath="{.data.redis-password}" | base64 -d)
```

### 6. Install Tyk API Gateway
- Install Tyk API Gateway using Tyk Helm chart:
```shell
helm install tyk-oss tyk-helm/tyk-oss -n tyk -f values.yaml --set "global.redis.pass=$REDIS_PASSWORD"
```

### 7. Forward Gateway Port
- Forward the Tyk Gateway port to localhost:
```shell
kubectl port-forward --namespace tyk service/gateway-svc-tyk-oss-tyk-gateway 8080:8080
```

- You can now access Tyk API Gateway at `http://localhost:8080`.

