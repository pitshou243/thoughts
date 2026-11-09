# How to Troubleshoot Elemental

## Environment

- SUSE Rancher Prime managing Elemental nodes.
- Elemental Operator and its custom resource definitions installed on the Rancher management cluster.
- Elemental nodes using RKE2 or K3s.

This procedure covers the Elemental Operator and MachineRegistration workflow. Commands must be adapted to the installed release, namespaces, and service names. It does not cover AWS Elemental or separate Cluster API provider implementations.

## Procedure

### 1. Identify the failing stage

Start with the earliest stage that has not completed. A machine appearing in the inventory does not by itself confirm successful OS installation or Kubernetes provisioning.

| Symptom | Start investigating |
| --- | --- |
| Registration resource is not ready | Elemental Operator and MachineRegistration conditions |
| ISO or RAW image is unavailable | SeedImage status and image build pod |
| Booted machine never appears in inventory | Registration service, network, TLS, and authentication |
| Inventory exists, but machine does not finish onboarding | Installation, reboot, and registration after reboot |
| Machine is available but not assigned to a cluster | Inventory labels, selectors, and machine pool quantity |
| Assigned machine does not become a Kubernetes node | Elemental system agent, Rancher system agent, and RKE2/K3s |
| OS upgrade does not start or complete | ManagedOSImage, Fleet delivery, and downstream upgrade jobs |

The official Elemental verification guide describes these lifecycle stages and their diagnostic entry points. [1]

Before changing configuration, record the failure time and timezone, affected machines, last successful operation, and recent changes. Collect logs before rebooting, reinstalling, or deleting resources.

### 2. Confirm the Kubernetes context and component versions

Run the following on the **Rancher management cluster**:

```bash
kubectl config current-context
kubectl get nodes -o wide
helm list -A
kubectl -n cattle-elemental-system get deployments,pods -o wide
kubectl -n cattle-elemental-system get deployments -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.template.spec.containers[*]}{.image}{" "}{end}{"\n"}{end}'
kubectl api-resources --api-group=elemental.cattle.io
```

Record Rancher, Elemental Operator, Elemental Register, OS image, and Kubernetes versions. Compare the combination against the support information for the installed release; Helm chart and application versions may differ.

If the expected API resources are missing, check the Elemental CRD installation. If a command returns `Forbidden`, confirm permissions before concluding that the resource is absent.

### 3. Check operator health and resource conditions

```bash
kubectl -n cattle-elemental-system get pods -o wide
kubectl -n cattle-elemental-system logs deployment/elemental-operator --all-containers=true --since=2h --timestamps=true
kubectl -n cattle-elemental-system get events --sort-by=.metadata.creationTimestamp
kubectl get machineregistrations.elemental.cattle.io -A
kubectl get machineinventories.elemental.cattle.io -A
kubectl get seedimages.elemental.cattle.io -A
```

If the deployment has a different name, select the actual pod or deployment from the listing. For a restarted operator container, collect its previous logs:

```bash
kubectl -n cattle-elemental-system logs <operator-pod> -c <container-name> --previous --timestamps=true
```

Inspect the affected registration:

```bash
kubectl -n fleet-default describe machineregistrations.elemental.cattle.io <registration-name>
kubectl -n fleet-default get machineregistrations.elemental.cattle.io <registration-name> -o yaml
```

Check condition messages and correlate errors with the failure time. Examples include webhook failures, unavailable APIs, permission errors, and reconciliation failures. Resolve the dependency named in the error instead of repeatedly restarting the operator.

A ready MachineRegistration indicates that its registration URL and associated ServiceAccount have been generated; it does not indicate that a machine has finished installation. [2]

### 4. Troubleshoot installation image creation

```bash
kubectl -n fleet-default describe seedimages.elemental.cattle.io <seedimage-name>
kubectl -n fleet-default get seedimages.elemental.cattle.io <seedimage-name> -o yaml
kubectl get pods,jobs -A
```

Identify the related build pod from its name, owner references, and events, then inspect it:

```bash
kubectl -n <build-namespace> describe pod <build-pod>
kubectl -n <build-namespace> logs <build-pod> --all-containers=true --timestamps=true
```

| Evidence | Next check |
| --- | --- |
| No build pod created | SeedImage conditions and operator reconciliation logs |
| Pending pod | Scheduling events, resources, storage, and admission policy |
| ImagePullBackOff | Image reference, registry access, credentials, and registry CA trust |
| Build container exits | Container logs and image build configuration |
| Image built but download fails | Published endpoint, ingress, proxy, and network access |

Confirm that the booted image contains the intended registration configuration. A previously downloaded ISO can retain outdated settings after the management resource changes. Rebuild and use new installation media when its embedded configuration must change.

### 5. Troubleshoot registration from the affected machine

Run these commands on the **affected machine**, including the live installation environment if installation has not completed:

```bash
cat /etc/os-release
date -Is
timedatectl status
ip -br address
ip route
systemctl list-unit-files 'elemental*' 'rancher-system-agent*'
sudo journalctl -u elemental-register-install.service -b --no-pager -n 300 -o short-iso
sudo journalctl -u elemental-register.service -b --no-pager -n 300 -o short-iso
```

Service availability depends on whether the machine is running installation media or its installed OS. A missing unit alone is insufficient to diagnose failure.

Test Rancher hostname resolution and HTTPS access using the actual hostname:

```bash
getent ahosts <rancher-hostname>
curl --connect-timeout 10 --max-time 30 -sS -o /dev/null -w 'HTTP status: %{http_code}\n'  https://<rancher-hostname>/ping
```

Successful access to `/ping` confirms that endpoint is reachable with curl's trust configuration. It does not prove that the registration URL, authentication, or Elemental client's CA configuration is correct. Test the actual registration URL separately in a secure session; its response contains registration configuration and its URL contains a token. [2]

| Error or symptom | Investigate |
| --- | --- |
| Name resolution failure | DNS configuration in the live image and installed OS |
| Connection timeout | Routing, firewall, proxy, and load balancer reachability |
| Connection reset | Load balancer, ingress, proxy, and server logs |
| HTTP 401/403 on registration endpoint | Token, registration resource, and authorization |
| HTTP 404 | Exact registration URL and ingress routing |
| HTTP 5xx | Ingress/backend availability and management cluster logs |
| Certificate validation failure | Certificate chain, hostname, time, and client trust |

Check proxy settings in the service configuration, not only in the interactive shell. Verify `NO_PROXY` entries against the actual topology. Redact credentials from any proxy output shared with support.

### 6. Investigate TLS and CA errors

For errors such as `x509: certificate signed by unknown authority` or `CA Data in provided kubeconfig was empty while strict verify was enabled`, first identify the failing service and the exact endpoint it contacts. Similar messages can originate from different clients and trust configurations.

Inspect the certificate chain presented to the affected machine:

```bash
openssl s_client -connect <rancher-hostname>:443 -servername <rancher-hostname> -showcerts </dev/null
```

Review hostname coverage, validity dates, issuer, intermediate certificates, and verification output. Compare the result from the machine with a working host; differences may indicate TLS interception or different network paths.

Check the registration client's configured `ca-cert`. The MachineRegistration reference documents it as required for a server certificate that is not signed by a public CA. [2]

For a publicly signed certificate, verify that the **boot image and installed OS** trust the required root and that the server supplies the intermediate chain. For a private CA, verify that the intended trust chain reaches the client through the supported configuration.

If curl succeeds but Elemental fails, compare their trust inputs. An explicit CA embedded in registration data can differ from the system trust store. Check the configuration returned by the registration endpoint as well as the MachineRegistration object.

Do not infer a specific product defect from the error text alone. Retain the operator/client versions, generated registration data, presented certificate chain, and failing service logs for comparison with release notes or a confirmed issue. Do not use disabled certificate verification as the resolution.

### 7. Check machine authentication and identity

Inspect the registration authentication settings and hardware:

```bash
ls -l /dev/tpm*
sudo dmidecode -s system-uuid
ip -br link
```

Elemental uses TPM authentication by default. Confirm that TPM 2.0 is enabled and accessible when that method is selected. Alternative identification methods have different security and support limitations; do not switch methods merely to bypass a failure. [3]

For cloned virtual machines, compare identifiers with a working machine. Duplicate SMBIOS UUIDs, MAC addresses, or emulated TPM identities can cause inventory collisions. Verify identity uniqueness before removing inventory records. [3]

### 8. Investigate incomplete OS installation or reboot

If a MachineInventory exists but onboarding remains incomplete:

```bash
kubectl -n fleet-default describe machineinventories.elemental.cattle.io <inventory-name>
kubectl -n fleet-default get machineinventories.elemental.cattle.io <inventory-name> -o yaml
```

On the machine:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINTS
df -h
df -i
findmnt
sudo journalctl -k -b --no-pager -n 200
sudo journalctl --list-boots
```

Compare the installation target with the actual disk layout and firmware mode. Check logs for disk selection errors, unavailable image sources, insufficient space, and I/O failures.

Review `spec.config.elemental.install`, including its disk selection and reboot settings. Also confirm that firmware boots the installed disk instead of returning to installation media. Registration after reboot should be checked separately from the initial installation service. [1][2]

For configuration that did not apply, confirm the intended syntax and when it is evaluated. MachineRegistration `cloud-config` is injected during installation and evaluated after reboot. [2]

Do not rerun an installation or reset command until the target disk and data impact have been verified.

### 9. Troubleshoot cluster assignment

Run on the **management cluster**:

```bash
kubectl -n fleet-default get machineinventories.elemental.cattle.io --show-labels
kubectl -n fleet-default get machineinventoryselectortemplates.elemental.cattle.io -o yaml
kubectl -n fleet-default get machineinventoryselectors.elemental.cattle.io -o yaml
kubectl -n fleet-default get clusters.provisioning.cattle.io <cluster-name> -o yaml
```

Compare the actual inventory labels with the selectors used by each machine pool. Check namespace, requested machine count, existing assignments, and condition messages. Pay attention to full label keys, including prefixes. An inventory's name or visual grouping in the UI is not a substitute for a selector match.

Correct a confirmed selector or configuration mismatch through the supported cluster configuration workflow. Avoid changing owner references or finalizers as an initial troubleshooting step.

### 10. Troubleshoot Kubernetes provisioning

On the **affected machine**:

```bash
sudo journalctl -u elemental-system-agent.service --since '2 hours ago' --no-pager -n 300 -o short-iso
sudo journalctl -u rancher-system-agent.service --since '2 hours ago' --no-pager -n 300 -o short-iso
systemctl status rke2-server rke2-agent k3s k3s-agent --no-pager
```

Inspect only the Kubernetes service installed for that machine's role:

```bash
sudo journalctl -u <kubernetes-service> --since '2 hours ago' --no-pager -n 300 -o short-iso
```

The Elemental system agent initiates the handoff to Rancher provisioning; Rancher system agent executes subsequent provisioning plans. Kubernetes upgrades follow Rancher's provisioning workflow. [1]

Use the failing command or plan in the logs to determine the next check: artifact download, registry trust, API connectivity, certificates, container runtime, or Kubernetes startup. If RKE2/K3s has started, continue with its service logs and node conditions rather than attributing all failures to registration.

On the **downstream cluster**, when its API is available:

```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

### 11. Troubleshoot OS upgrades

OS upgrades and Kubernetes upgrades use different workflows. For an OS upgrade, trace delivery from the management resource to the downstream job.

On the **management cluster**:

```bash
kubectl get managedosimages.elemental.cattle.io -A
kubectl -n fleet-default get managedosimages.elemental.cattle.io <upgrade-name> -o yaml
kubectl -n fleet-default get bundles.fleet.cattle.io
kubectl get bundledeployments.fleet.cattle.io -A
```

Inspect the Bundle associated with the ManagedOSImage, its targets, and BundleDeployment conditions. The documented workflow generates a Bundle with a `mos-` prefix. [4]

On the **downstream cluster**:

```bash
kubectl get plans.upgrade.cattle.io -A
kubectl get deployments -A | rg system-upgrade-controller
kubectl get jobs,pods -A -o wide
kubectl -n <upgrade-namespace> logs deployment/system-upgrade-controller --all-containers=true --since=2h --timestamps=true
kubectl -n <upgrade-namespace> describe plans.upgrade.cattle.io <plan-name>
kubectl -n <upgrade-namespace> describe job <upgrade-job>
kubectl -n <upgrade-namespace> logs job/<upgrade-job> --all-containers=true --timestamps=true
```

If `rg` is unavailable, locate the controller in the unfiltered deployment listing.

| Finding | Next check |
| --- | --- |
| No associated Bundle | ManagedOSImage conditions and operator logs |
| Bundle not delivered | Fleet targets, BundleDeployment status, and cluster connectivity |
| Plan delivered but no job | Controller logs, plan selection, and node eligibility |
| Job cannot pull image | Registry reference, credentials, network, and CA trust |
| Job starts but upgrade fails | Job logs, disk capacity, and target image compatibility |
| Node fails to return after reboot | Console, boot configuration, network, and service logs |

The documented upgrade process includes a reboot and a subsequent verification run. An earlier pod with `Unknown` status alone does not establish failure; correlate it with later job completion and the node's resulting OS version. Failed upgrades may leave nodes cordoned. [4]

Correct or stop the failed upgrade through the ManagedOSImage workflow. Before manually uncordoning a node, verify that it is healthy, its OS version is understood, and no upgrade operation is still active. [4]

### 12. Validate recovery and collect evidence if unresolved

Confirm the outcome relevant to the failed stage:

- The registration resource is ready and the intended image is available.
- The machine completes onboarding with the expected identity and configuration.
- Inventory labels match the intended pool and the machine is assigned correctly.
- The downstream Kubernetes node is Ready and workloads recover.
- An OS upgrade completes verification and the expected OS version is running.

For an unresolved case, provide:

- Failure time and timezone, affected hosts, reproduction steps, and recent changes.
- Component versions and exact source/target image references.
- Relevant MachineRegistration, SeedImage, MachineInventory, selectors, and provisioning Cluster YAML.
- Operator, registration, installation, system agent, and Kubernetes service logs from the affected machine.
- For upgrades: ManagedOSImage, Bundle/BundleDeployment, Plan, job status, and controller/job logs.
- Disk layout, capacity, network checks, and certificate chain evidence when relevant.

Include logs from before and after a reboot if retained. Logs from a healthy machine can provide comparison but cannot establish what failed on a different machine. Redact registration tokens, passwords, private keys, kubeconfig credentials, and other secrets before sharing evidence.

### References

1. [Elemental troubleshooting and verification](https://elemental.docs.rancher.com/troubleshooting-verification/)
2. [MachineRegistration reference](https://elemental.docs.rancher.com/machineregistration-reference/)
3. [Elemental authentication](https://elemental.docs.rancher.com/authentication/)
4. [Elemental OS upgrade troubleshooting](https://elemental.docs.rancher.com/troubleshooting-upgrade/)
