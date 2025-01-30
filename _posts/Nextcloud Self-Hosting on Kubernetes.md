
# Nextcloud Self-Hosting on Kubernetes

**Date:** Jan 31, 2025  
**Reading Time:** 12 min read

## Goal

Self-host Nextcloud on Kubernetes and use it as the server-side for the /e operating system.

Nextcloud offers an on-premises content collaboration platform, which is open-source and free. The Nextcloud server is written in PHP and JavaScript.

The /e/ ROM is a fork of Android (more precisely from LineageOS).

See the previous post to see how to install the OS on an LG G3 and my efforts to self-host the /e server beta version.

The /e self-hosting exercise turned out to be quite cumbersome due to the installation script of the beta version, which is one of these do-everything-in-one scripts that are difficult to troubleshoot and make maintenance and updates overly complicated.

However, the /e server-side is in fact just a composition of services and their configuration:

- **Nextcloud**, as file synchronization software
- **Postfix**, to host the own mail server
- **OnlyOffice**, to allow editing of documents

I was actually only interested in photos and files synchronization. Based on this premise, I decided to try to install Nextcloud directly and see if I could use it as the server side for the /e operating system on the phone. The nice guys in charge of /e development told me that this should be possible and advised me to do so.

As a hosting environment, I wanted to use Kubernetes - as usual. This time - instead of deploying directly on either my K8s or my K3s cluster on-premises - I wanted to try the DigitalOcean K8s services. If the experiment was successful, I would move the setup to my home network afterward, to use Nextcloud to sync my files and photos.

**NOTE:** Another reason was that by the time of this writing, I was on vacation with no access to my home network ;)

## Pre-conditions

- A DigitalOcean account to host the K8s cluster
- A client device to test the installation
- A hosted public domain (mydomain.com)
- A DNS provider to register the necessary DNS entries

I had a DigitalOcean account and was still enjoying the free credit I got when signing up for the first time. The client device will be the LG G3 where /e was already running. My public domain was hosted by GoDaddy, and as a DNS provider, I would use Cloudflare as I always do.

## Milestones

1. Set up a K8s cluster on DigitalOcean
2. Identify software components and write K8s manifests
3. Set up DNS
4. Ingress
5. Certmanager and cluster issuer
6. Deployment with Kustomize
7. Register /e client with Nextcloud account

### 1. Set up a K8s Cluster

#### 1.1. Create and Configure the New Cluster

Setting up a K8s cluster on DigitalOcean was easy:

- I selected the latest K8s version, which was the default
- I selected the datacenter closest to my current location
- I selected one single basic node with 2.5 GB RAM usable (4 GB Total)/2 vCPUs
- As the cluster name, I typed `eramonk8s`

**NOTE:** Jumping a little ahead, I have to say this setup was painfully slow. More nodes and a little more RAM would have been more convenient.

#### 1.2. Install and Configure `kubectl` and `doctl`

As I waited for the provisioning of the cluster, I installed version 1.19.3 of `kubectl`:

```bash
eramon@pacharan:~/dev$ curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.19.3/bin/linux/amd64/kubectl
```

**NOTE:** The `kubectl` version must match the K8s version of the cluster.

```bash
eramon@pacharan:~/dev$ chmod +x ./kubectl
eramon@pacharan:~/dev$ sudo mv ./kubectl /usr/local/bin/kubectl
eramon@pacharan:~/dev$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.3", GitCommit:"1e11e4a2108024935ecfcb2912226cedeafd99df", GitTreeState:"clean", BuildDate:"2020-10-14T12:50:19Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}
```

I also downloaded the `doctl` tool as recommended by the DigitalOcean wizard:

```bash
eramon@pacharan:~/dev$ wget https://github.com/digitalocean/doctl/releases/download/v1.54.0/doctl-1.54.0-linux-amd64.tar.gz
eramon@pacharan:~/dev$ tar xvzf doctl-1.54.0-linux-amd64.tar.gz
eramon@pacharan:~/dev$ sudo mv doctl /usr/local/bin/
```

I used `doctl` for automated certificate management to access the K8s API with `kubectl` and download the configuration file. Following the instructions, I first ran:

```bash
eramon@pacharan:~/dev$ doctl auth init
Please authenticate doctl for use with your DigitalOcean account. You can generate a token in the control panel at https://cloud.digitalocean.com/account/api/tokens

Enter your access token: 
Validating token... OK
```

I created the access token on the URL above and gave it the same name as the cluster. I copied it to the clipboard and pasted it on the console as instructed.

Then I used `doctl` to generate the configuration file:

```bash
eramon@pacharan:~/dev$ doctl kubernetes cluster kubeconfig save eramonk8s
Notice: Adding cluster credentials to kubeconfig file found in "/home/eramon/.kube/config"
Notice: Setting current-context to do-fra1-eramonk8s
```

After this, I was able to connect to my cluster via `kubectl`:

```bash
eramon@pacharan:~/dev$ kubectl cluster-info
Kubernetes master is running at https://73c21301-5143-42fa-ab21-1c4c54e9b1f0.k8s.ondigitalocean.com
CoreDNS is running at https://73c21301-5143-42fa-ab21-1c4c54e9b1f0.k8s.ondigitalocean.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

Continuing with the wizard on the browser, I installed the following 1-Click Apps:

- NGINX Ingress Controller
- Kubernetes Monitoring Stack

Over the control panel, I was able to access the Kubernetes Dashboard to monitor the status of the cluster :)

### 2. Software Components, Docker Images, and K8s Manifests

Nextcloud is built upon the following components:

- Persistent storage to save content
- MariaDB for metadata storage
- The Nextcloud web app: PHP + Apache

For persistent storage, I would use the persistent volumes offered as a service by DigitalOcean.

For both MariaDB and Nextcloud, there are official Docker images available on DockerHub.

With all this in mind, I was able to start writing the K8s manifests.

#### 2.1 Secrets

Kubernetes Secrets let you store and manage sensitive information, such as passwords.

I started with the secrets I would need for the MariaDB and Nextcloud deployments later. Usually, I generate the secrets manually, but this time I used a manifest and included it as part of `kustomization.yaml`, just taking care not to commit the YAML file containing the passwords, but an example version of it with placeholders.

Kustomize provides a way to customize application configuration and is built into `kubectl` as `apply -k`.

The following secrets are needed for the MariaDB and Nextcloud containers:

- A `MYSQL_DATABASE` which I would set to `nextcloud`, needed for both MariaDB and Nextcloud.
- A `MYSQL_USER` which I would set to `nextcloud` too, needed for both MariaDB and Nextcloud.
- A `MYSQL_PASSWORD` which I would generate with `pwgen` and encode to have a base-64-encoded string, needed for both MariaDB and Nextcloud.
- A `MYSQL_ROOT_PASSWORD`, which I would set - to simplify - with the same value as the `MYSQL_PASSWORD`.

Install `pwgen`:

```bash
eramon@pacharan:~/dev/kubenextcloud$ sudo apt-get install pwgen
```

Use `pwgen` to generate a random password for MariaDB:

```bash
eramon@pacharan:~/dev/kubenextcloud$ echo -n `pwgen -s -1 16`
```

Note the output.

As mentioned, I used this password for both the `MYSQL_PASSWORD` and the `MYSQL_ROOT_PASSWORD`.

For `MYSQL_DATABASE` and `MYSQL_USER`, I set `nextcloud`, use `openssl` to base64-encode “nextcloud”:

```bash
eramon@pacharan:~/dev/kubenextcloud$ echo -n 'nextcloud' | openssl base64
bmV4dGNsb3Vk
```

Having all the values, write a manifest for the secrets, using the generated passwords:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secrets
type: Opaque
data:
  MYSQL_DATABASE: bmV4dGNsb3Vk 
  MYSQL_USER: bmV4dGNsb3Vk 
  MYSQL_PASSWORD: base64-encoded-pwgen-generated-password 
  MYSQL_ROOT_PASSWORD: base64-encoded-pwgen-generated-password 
```

Create a new `kustomize.yaml` file and included the manifest for the secrets there.

#### 2.2 Persistent Volumes

A PersistentVolumeClaim (PVC) is a request for storage by a user. A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned.

To see the possibilities for the creation of a persistent volume with DigitalOcean, I went to the Volumes section on the Manage menu.

**NOTE:** One persistent volume claim was already created - apparently, it was automatically set for the use of the monitoring tool.

I went to the [How to Add Block Storage Volumes to Kubernetes Clusters](https://www.digitalocean.com/docs/kubernetes/how-to/add-persistent-storage/) tutorial. Taking the code from the example, I created a manifest for a 3GB block storage volume to persist the end-user data managed by Nextcloud:

`nextcloud-pvc.yaml`

I created an additional volume for the storage of the MariaDB metadata with a size of 2GB, to persist the Nextcloud metadata stored on MariaDB:

`mariadb-pvc.yaml`

For managing all manifests and easing the deployment, I would use Kustomize.

Add the PVC manifests to `kustomization.yaml`.

#### 2.3 MariaDB

Create a manifest file for the MariaDB deployment:

`mariadb-deployment.yaml`

A Deployment provides declarative updates for Pods and ReplicaSets.

The MariaDB Docker image allows the creation of a database upon the creation of the container. When you start the MariaDB image, you can adjust the configuration of the MariaDB instance by passing one or more environment variables on the `docker run` command line. Do note that none of the variables below will have any effect if you start the container with a data directory that already contains a database: any pre-existing database will always be left untouched on container startup.

The deployment manifest requires configuring the following environment variables:

- `MYSQL_ROOT_PASSWORD`: specifies the password that will be set for the MariaDB root superuser account (mandatory)
- `MYSQL_DATABASE`: allows you to specify the name of a database to be created on image startup (optional)
- `MYSQL_USER`, `MYSQL_PASSWORD`: used in conjunction to create a new user and to set that user’s password (optional)

**NOTE:** In order for MariaDB to re-create the Nextcloud database, the persistent volume must be deleted. It’s not enough to delete the container. The environment variables only work if the database is started for the first time.

Create a manifest file for the MariaDB service:

`mariadb-service.yaml`

A service is an abstract way to expose an application running on a set of Pods as a network service.

Add the deployment and service manifest files to `kustomization.yaml`.

#### 2.4 Redis

The performance of the Nextcloud server can be improved using memory caching. Redis provides local and distributed caching as well as transactional file locking.

Create the manifest file for the Redis deployment:

`redis-deployment.yaml`

Create the manifest file for the Redis service:

`redis-service.yaml`

For Nextcloud to use Redis, we’ll need to set the environment variable `REDIS_HOST` on the Nextcloud container.

#### 2.5 Nextcloud

Create the manifest file for the Nextcloud deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud 
  labels:
    app: nextcloud 
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nextcloud 
  template:
    metadata:
      labels:
        app: nextcloud
    spec:
      volumes:
      - name: nextcloud-storage
        persistentVolumeClaim: 
          claimName: nextcloud-pvc
      containers:
        - image: nextcloud:apache
          name: nextcloud 
          ports:
            - containerPort: 80
          env:
            - name: REDIS_HOST
              value: redis
            - name: MYSQL_HOST
              value: mariadb 
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  key: MYSQL_DATABASE
                  name: mariadb-secrets
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: MYSQL_PASSWORD
                  name: mariadb-secrets
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  key: MYSQL_USER
                  name: mariadb-secrets
            - name: NEXTCLOUD_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: MYSQL_PASSWORD
                  name: mariadb-secrets 
            - name: NEXTCLOUD_ADMIN_USER
              value: "admin"
            - name: NEXTCLOUD_TRUSTED_DOMAINS 
              value: mydomain.com 
          volumeMounts:
            - mountPath: /var/www/html
              name: nextcloud-storage
```

The Nextcloud image supports auto-configuration via environment variables. You can preconfigure everything that is asked on the install page on first run:

- `MYSQL_DATABASE`: Name of the database using MySQL / MariaDB
- `MYSQL_USER`: Username for the database using MySQL / MariaDB
- `MYSQL_PASSWORD`: Password for the database user using MySQL / MariaDB
- `MYSQL_HOST`: Hostname of the database server using MySQL / MariaDB

With a complete configuration by using all variables for your database type, you can additionally configure your Nextcloud instance by setting admin user and password:

- `NEXTCLOUD_ADMIN_USER`: Name of the Nextcloud admin user. I used `admin`.
- `NEXTCLOUD_ADMIN_PASSWORD`: Password for the Nextcloud admin user. For simplification - since this is just an experimental setup - I configured it to be the same password as the one used for the Nextcloud database.

One or more trusted domains can be set through the environment variable too:

- `NEXTCLOUD_TRUSTED_DOMAINS`: optional space-separated list of IPs or domains. The IP of the load balancer or the domain `mydomain.com` must be configured here.

In order for Nextcloud to see and use the Redis service, we need to include an additional environment variable:

- `REDIS_HOST`: name of the Redis service which in this case is `redis`.

Create a manifest file for the Nextcloud service:

`nextcloud-service.yaml`

Add the Nextcloud deployment and service manifest files to `kustomization.yaml`.

### 3. Ingress

Kubernetes Ingresses allow you to flexibly route traffic from outside your Kubernetes cluster to Services inside of your cluster. This is accomplished using Ingress Resources, which define rules for routing HTTP and HTTPS traffic to Kubernetes Services, and Ingress Controllers, which implement the rules by load balancing traffic and routing it to the appropriate backend Services.

I had already installed the `nginx-ingress-controller` as a 1-click app when setting up the cluster at the beginning. To make sure it was running:

```bash
eramon@pacharan:~/dev/kubenextcloud$ kubectl get pods -n ingress-nginx
NAME                                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-ingress-nginx-controller-7898d5969d-sgph4   1/1     Running   0          110m
```

Create a manifest file for ingress:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nextcloud-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    cert-manager.io/acme-challenge-type: http01
spec:
  tls:
  - hosts: 
    - nextcloud.mydomain.com 
    secretName: nextcloud-tls 
  rules:
  - host: nextcloud.mydomain.com 
    http: 
      paths: 
      - path: /
        backend:
          serviceName: nextcloud
          servicePort: 80
```

Don’t forget to include the `tls` directive and the `certbot` and `letsencrypt` annotations for the SSL server certificate to be automatically generated by `certbot`.

Add the ingress manifest file to `kustomization.yaml`.

### 4. DNS

I have my domain `mydomain.com` registered and hosted by GoDaddy. I use Cloudflare to manage the DNS configuration for my projects.

I listed the services in the namespace `nginx-ingress` to find out the public IP of the load balancer of the cluster:

```bash
eramon@pacharan:~/dev/kubenextcloud$ kubectl get service -n ingress-nginx
NAME                                               TYPE           CLUSTER-
