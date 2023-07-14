# [How to use k3d](https://k3d.io/v5.5.1/)

## [Usage Config file](https://k3d.io/v5.5.1/usage/configfile/) ¶

Using a config file is as easy as putting it in a well-known place in your file system and then referencing it via flag:

- All options in config file: `k3d cluster create --config /home/me/my-awesome-config.yaml` (must be `.yaml`/`.yml`)
- With CLI override (name): `k3d cluster create somename --config /home/me/my-awesome-config.yaml`
- With CLI override (extra volume): `k3d cluster create --config /home/me/my-awesome-config.yaml --volume '/some/path:/some:path@server:0'`


<details>
  <summary>Sample Config</summary>
  
```yml
# k3d configuration file, saved as e.g. /home/me/myk3dcluster.yaml
apiVersion: k3d.io/v1alpha5 # this will change in the future as we make everything more stable
kind: Simple # internally, we also have a Cluster config, which is not yet available externally
metadata:
name: mycluster # name that you want to give to your cluster (will still be prefixed with `k3d-`)
servers: 1 # same as `--servers 1`
agents: 2 # same as `--agents 2`
kubeAPI: # same as `--api-port myhost.my.domain:6445` (where the name would resolve to 127.0.0.1)
host: "myhost.my.domain" # important for the `server` setting in the kubeconfig
hostIP: "127.0.0.1" # where the Kubernetes API will be listening on
hostPort: "6445" # where the Kubernetes API listening port will be mapped to on your host system
image: rancher/k3s:v1.20.4-k3s1 # same as `--image rancher/k3s:v1.20.4-k3s1`
network: my-custom-net # same as `--network my-custom-net`
subnet: "172.28.0.0/16" # same as `--subnet 172.28.0.0/16`
token: superSecretToken # same as `--token superSecretToken`
volumes: # repeatable flags are represented as YAML lists
- volume: /my/host/path:/path/in/node # same as `--volume '/my/host/path:/path/in/node@server:0;agent:*'`
    nodeFilters:
    - server:0
    - agent:*
ports:
- port: 8080:80 # same as `--port '8080:80@loadbalancer'`
    nodeFilters:
    - loadbalancer
env:
- envVar: bar=baz # same as `--env 'bar=baz@server:0'`
    nodeFilters:
    - server:0
registries: # define how registries should be created or used
create: # creates a default registry to be used with the cluster; same as `--registry-create registry.localhost`
    name: registry.localhost
    host: "0.0.0.0"
    hostPort: "5000"
    proxy: # omit this to have a "normal" registry, set this to create a registry proxy (pull-through cache)
    remoteURL: https://registry-1.docker.io # mirror the DockerHub registry
    username: "" # unauthenticated
    password: "" # unauthenticated
    volumes:
    - /some/path:/var/lib/registry # persist registry data locally
use:
    - k3d-myotherregistry:5000 # some other k3d-managed registry; same as `--registry-use 'k3d-myotherregistry:5000'`
config: | # define contents of the `registries.yaml` file (or reference a file); same as `--registry-config /path/to/config.yaml`
    mirrors:
    "my.company.registry":
        endpoint:
        - http://my.company.registry:5000
hostAliases: # /etc/hosts style entries to be injected into /etc/hosts in the node containers and in the NodeHosts section in CoreDNS
- ip: 1.2.3.4
    hostnames: 
    - my.host.local
    - that.other.local
- ip: 1.1.1.1
    hostnames:
    - cloud.flare.dns
options:
k3d: # k3d runtime settings
    wait: true # wait for cluster to be usable before returning; same as `--wait` (default: true)
    timeout: "60s" # wait timeout before aborting; same as `--timeout 60s`
    disableLoadbalancer: false # same as `--no-lb`
    disableImageVolume: false # same as `--no-image-volume`
    disableRollback: false # same as `--no-Rollback`
    loadbalancer:
    configOverrides:
        - settings.workerConnections=2048
k3s: # options passed on to K3s itself
    extraArgs: # additional arguments passed to the `k3s server|agent` command; same as `--k3s-arg`
    - arg: "--tls-san=my.host.domain"
        nodeFilters:
        - server:*
    nodeLabels:
    - label: foo=bar # same as `--k3s-node-label 'foo=bar@agent:1'` -> this results in a Kubernetes node label
        nodeFilters:
        - agent:1
kubeconfig:
    updateDefaultKubeconfig: true # add new cluster to your default Kubeconfig; same as `--kubeconfig-update-default` (default: true)
    switchCurrentContext: true # also set current-context to the new cluster's context; same as `--kubeconfig-switch-context` (default: true)
runtime: # runtime (docker) specific options
    gpuRequest: all # same as `--gpus all`
    labels:
    - label: bar=baz # same as `--runtime-label 'bar=baz@agent:1'` -> this results in a runtime (docker) container label
        nodeFilters:
        - agent:1
    ulimits:
    - name: nofile
        soft: 26677
        hard: 26677
```
</details>



## Handling Kubeconfigs¶

Getting the kubeconfig for a newly created cluster¶

1. Create a new kubeconfig file after cluster creation

    * `k3d kubeconfig write mycluster`
        * Note: this will create (or update) the file `$HOME/.k3d/kubeconfig-mycluster.yaml`
        * Tip: Use it: `export KUBECONFIG=$(k3d kubeconfig write mycluster)`
        * Note 2: alternatively you can use `k3d kubeconfig get mycluster > some-file.yaml`

2. Update your default kubeconfig upon cluster creation (DEFAULT)

    * `k3d cluster create mycluster --kubeconfig-update-default`
        * Note: this won’t switch the current-context (append `--kubeconfig-switch-context` to do so)

3. Update your default kubeconfig after cluster creation

    * `k3d kubeconfig merge mycluster --kubeconfig-merge-default`
        * Note: this won’t switch the current-context (append --kubeconfig-switch-context to do so)

4. Update a different kubeconfig after cluster creation

    * `k3d kubeconfig merge mycluster --output some/other/file.yaml`
        * Note: this won’t switch the current-context
    * The file will be created if it doesn’t exist



## Exposing Services¶

### 1. via Ingress (recommended)¶

In this example, we will deploy a simple nginx webserver deployment and make it accessible via ingress.
Therefore, we have to create the cluster in a way, that the internal port 80 (where the traefik ingress controller is listening on) is exposed on the host system.

1. Create a cluster, mapping the ingress port 80 to localhost:8081
    
    `k3d cluster create --api-port 6550 -p "8081:80@loadbalancer" --agents 2`

    ---
    **Good to know**
    * `--api-port 6550` is not required for the example to work.
        It’s used to have k3s‘s API-Server listening on port 6550 with that port mapped to the host system.
    * the port-mapping construct `8081:80@loadbalancer` means:

        “map port `8081` from the host to port `80` on the container which matches the nodefilter `loadbalancer`“
        * the `loadbalancer` nodefilter matches only the `serverlb` that’s deployed in front of a cluster’s server nodes
            * all ports exposed on the `serverlb` will be proxied to the same ports on all server nodes in the cluster
    ---

2. Get the kubeconfig file (redundant, as `k3d cluster create` already merges it into your default kubeconfig file)

    `export KUBECONFIG="$(k3d kubeconfig write k3s-default)"`
3. Create a nginx deployment

    `kubectl create deployment nginx --image=nginx`
4. Create a ClusterIP service for it

    `kubectl create service clusterip nginx --tcp=80:80`
5. Create an ingress object for it by copying the following manifest to a file and applying with `kubectl apply -f thatfile.yaml`

    **Note**: k3s deploys traefik as the default ingress controller
    ```yaml
    # apiVersion: networking.k8s.io/v1beta1 # for k3s < v1.19
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
    name: nginx
    annotations:
        ingress.kubernetes.io/ssl-redirect: "false"
    spec:
    rules:
    - http:
        paths:
        - path: /
            pathType: Prefix
            backend:
            service:
                name: nginx
                port:
                number: 80
    ```

6. Curl it via localhost

    `curl localhost:8081/`


### 2. via NodePort¶

1. Create a cluster, mapping the port `30080` from `agent-0` to `localhost:8082`

    `k3d cluster create mycluster -p "8082:30080@agent:0" --agents 2`

    - **Note 1**: Kubernetes’ default NodePort range is [30000-32767](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport)
    - **Note 2**: You may as well expose the whole NodePort range from the very beginning, e.g. via `k3d cluster create mycluster --agents 3 -p "30000-32767:30000-32767@server:0"` (See [this video from @portainer](https://www.youtube.com/watch?v=5HaU6338lAk))
        - **Warning**: Docker creates iptable entries and a new proxy process per port-mapping, so this may take a very long time or even freeze your system!
2. Create a NodePort service for it by copying the following manifest to a file and applying it with kubectl `apply -f <file>`

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
    labels:
        app: nginx
    name: nginx
    spec:
    ports:
    - name: 80-80
        nodePort: 30080
        port: 80
        protocol: TCP
        targetPort: 80
    selector:
        app: nginx
    type: NodePort
    ```
3. Curl it via localhost

    `curl localhost:8082/`