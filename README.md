# My Homelab

Originally based on [Ajay Kumar's CLuster Template](https://github.com/ajaykumar4/cluster-template), but with the Talos stuff stripped out. Cilium and coredns have been replaced with k3s' built in coredns and flannel. kube-vip has been added as the load balancer.

Some inspiration:
- https://github.com/ajaykumar4/home-lab-argocd
- https://github.com/onedr0p/home-ops - flux based
- https://github.com/gruberdev/homelab/
- https://github.com/crkochan/argocd-homelab
- https://github.com/clearlybaffled/homelab
- https://github.com/danmanners/homelab-kube-cluster
- https://github.com/brunnels/talos-cluster/
- https://github.com/bjw-s/helm-charts

### Bootstrap

1) get k3s running

    1) edit `/etc/rancher/k3s/config.yaml`:
        ```yaml
        kube-controller-manager-arg:
        - "bind-address=0.0.0.0"
        kube-proxy-arg:
        - "metrics-bind-address=0.0.0.0"
        kube-scheduler-arg:
        - "bind-address=0.0.0.0"
        etcd-expose-metrics: true
        tls-san:
          - 192.168.1.3

        cluster-init: true
        snapshotter: btrfs
        secrets-encryption: true
        cluster-cidr: 10.42.0.0/16,2001:cafe:42::/56
        service-cidr: 10.43.0.0/16,2001:cafe:43::/112
        flannel-ipv6-masq: true
        disable:
          - traefik
          - servicelb
          - local-storage
        ```

    1) `systemctl enable --now k3s`
    1) `mkdir -p "${HOME}/.kube" && sudo cp /etc/rancher/k3s/k3s.yaml "${HOME}/.kube/config" && sudo chown $(id -u):$(id -g) "${HOME}/.kube/config"`
    1) make sure `open-iscsi` is installed and enabled: `systemctl enable --now iscsid.socket`

1) Install spegel, argo and sync the cluster to the repository state:

    Make sure `helm kubectl helmfile go-task sops` (also might have to do `ln -s /bin/task /bin/go-task` if on arch)

    ```sh
    task bootstrap:apps
    ```

1) Watch the rollout of your cluster happen:

    ```sh
    watch kubectl get pods --all-namespaces
    ```

    If your having issues, temporarily port forward argo so you can see the status:

    ```sh
    kubectl port-forward service/argo-cd-argocd-server -n argo-system 8080:443 --address '0.0.0.0' &
    ```


1) after `kube-vip` comes up, you'll see that the services have IPs assigned now: `kubectl get svc -A`. When this happens you can copy the kube config to another system

    ```sh
    mkdir -p "${HOME}/.kube" && \
    scp <USERNAME>@192.168.1.XXX:/home/<USERNAME>/.kube/config "${HOME}/.kube/config" && \
    sed -i 's/127.0.0.1/192.168.1.9/g' "${HOME}/.kube/config"
    ```

## 📣 Argo w/ Cloudflare post installation

### 🌐 Public DNS

> [!TIP]
> Use the `external` ingress class to make applications public to the internet.

The `external-dns` application created in the `networking` namespace will handle creating public DNS records. By default, `echo-server` and the `argo` are the only subdomains reachable from the public internet. In order to make additional applications public you must set set the correct ingress class name and ingress annotations like in the HelmRelease for `echo-server`.

### 🏠 Home DNS

PiHole forwarding to dns-over-https

### 🪝 Github Webhook

By default Argo will periodically check your git repository for changes. In order to have Argo reconcile on `git push` you must configure Github to send `push` events to Argo.

> [!IMPORTANT]
> This will only work after you have switched over certificates to the Let's Encrypt Production servers.

1. Full URL with the webhook path appended:
    ```text
    https://argo.${cloudflare.domain}/api/webhook
    ```

2. Navigate to the settings of your repository on Github, under "Settings/Webhooks" press the "Add webhook" button. Fill in the webhook URL and your `${github.webhook_token}` secret in `config.yaml`, Content type: `application/json`, Events: Choose Just the push event, and save.

## 🤖 Renovate

[Renovate](https://www.mend.io/renovate) is a tool that automates dependency management. It is designed to scan your repository around the clock and open PRs for out-of-date dependencies it finds. Common dependencies it can discover are Helm charts, container images, GitHub Actions, Ansible roles... even Argo itself! Merging a PR will cause Argo to apply the update to your cluster.

To enable Renovate, click the 'Configure' button over at their [Github app page](https://github.com/apps/renovate) and select your repository. Renovate creates a "Dependency Dashboard" as an issue in your repository, giving an overview of the status of all updates. The dashboard has interactive checkboxes that let you do things like advance scheduling or reattempt update PRs you closed without merging.

The base Renovate configuration in your repository can be viewed at [.github/renovate.json5](./.github/renovate.json5). By default it is scheduled to be active with PRs every weekend, but you can [change the schedule to anything you want](https://docs.renovatebot.com/presets-schedule), or remove it if you want Renovate to open PRs right away.

## 🐛 Debugging

Below is a general guide on trying to debug an issue with an resource or application. For example, if a workload/resource is not showing up or a pod has started but in a `CrashLoopBackOff` or `Pending` state.

1. Start by checking all Argo Applications and verify they are healthy.

    ```sh
      kubectl get applications -n argo-system
    ```

2. Then check the if the pod is present.

    ```sh
    kubectl -n <namespace> get pods -o wide
    ```

3. Then check the logs of the pod if its there.

    ```sh
    kubectl -n <namespace> logs <pod-name> -f
    ```

4. If a resource exists try to describe it to see what problems it might have.

    ```sh
    kubectl -n <namespace> describe <resource> <name>
    ```

5. Check the namespace events

    ```sh
    kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
    ```

Resolving problems that you have could take some tweaking of your YAML manifests in order to get things working, other times it could be a external factor like permissions on NFS. If you are unable to figure out your problem see the help section below.

## Unifi IPv6 ULA setup

For more info: https://techlevelup.net/assign-ipv6-guas-and-ulas-simultaneously-in-your-unifi-home-network/, https://www.appuntidallarete.com/ubiquiti-unifi-usg-advanced-configuration-using-config-gateway-json/, https://community.ui.com/questions/IPv6-Have-USG-assign-Unique-Local-Address-through-Router-Advertisement/e516521b-9f40-4a00-a0b6-60311790276d#comment/5cb60cbc-7bb4-46cd-a284-1d52db8b1173

in your unifi network app's data folder, go to `<unifi folder base>/data/sties/<unifi site name>`. (e.g. on cloudkey it's `/srv/unifi/data/sites/[name]/default`)

Check if it's `br0` on your router, ssh into and run `ip a`, and see which interface currently has `192.168.1.1` assigned to it

Add `config.gateway.json`
```json
{
  "interfaces": {
    "ethernet": {
      "br0": {
        "address": [ "192.168.1.1/24", "fd12:3456:789a:bcde::1/64"],
        "ipv6": {
          "router-advert": {
            "prefix": {
              "fd12:3456:789a:bcde::/64": {
                "autonomous-flag": "true",
                "on-link-flag": "true",
                "preferred-lifetime": "0",
                "valid-lifetime": "86400"
              }
            }
          }
        }
      }
    }
  }
}
```

Run `chown 65534:65534 config.gateway.json`
