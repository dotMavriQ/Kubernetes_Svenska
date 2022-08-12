---
reviewers:
- stclair
title: Restrict a Container's Access to Resources with AppArmor
content_type: tutorial
weight: 10
---

<!-- overview -->

{{< feature-state for_k8s_version="v1.4" state="beta" >}}


Apparmor är Linux-kärnans säkerhetsmodul som kompletterar standardåtgärder för Linux användarrättigheter och gruppbaserade
behörigheter för att begränsa program till en begränsad uppsättning resurser.Apparmor kan konfigureras
för att minska dess potentiella attackytor och ge ett större djupgående försvar. Den är
konfigurerad genom inställda profiler för att tillåta åtkomst som behövs av ett specifikt program eller behållare,
som Linux storlekskapacitet, nätverkstillträde, filbehörigheter osv.. Varje profil kan köras i antingen
"enforcing" läge, som blockerar åtkomst till tillåtna resurser, eller "complain"-läge, som bara rapporterar
inskränkningar.

Apparmor kan hjälpa dig att köra en säkrare utplacering genom att begränsa vilka containrar som tillåts
Gör och/eller ge bättre revision genom systemloggar. Det är dock viktigt att komma ihåg
att Apparmor inte är en silverkula och kan bara göra så mycket för att skydda mot exploits i din
applikationskod. Det är viktigt att tillhandahålla bra, restriktiva profiler och härda din
applikationer och kluster från andra vinklar också.



## {{% heading "objectives" %}}


* Se ett exempel på hur man laddar en profil på en nod
* Lär dig hur man upprätthåller profilen på en pod
* Lär dig hur man kontrollerar att profilen är laddad
* Se vad som händer när en profil kränks
* Se vad som händer när en profil inte kan laddas



## {{% heading "prerequisites" %}}


Säkerställ att:

1. Kubernetesversionen är åtminstone v1.4 -- Kubernetes -stöd för Apparmor lades till i
   v1.4. Kubernetes -komponenter äldre än V1.4 är inte medvetna om de nya Apparmor -kommentarerna, och
   kommer **tyst ignorera** alla apparmorinställningar som tillhandahålls. För att säkerställa att dina seedpods är
   Det är viktigt att verifiera Kubelet-versionen av dina noder:

   ```shell
   kubectl get nodes -o=jsonpath=$'{range .items[*]}{@.metadata.name}: {@.status.nodeInfo.kubeletVersion}\n{end}'
   ```
   ```
   gke-test-default-pool-239f5d02-gyn2: v1.4.0
   gke-test-default-pool-239f5d02-x1kf: v1.4.0
   gke-test-default-pool-239f5d02-xwux: v1.4.0
   ```

2. Apparmor -kärnmodulen är aktiverad - för att Linux -kärnan ska kunna upprätthålla en Apparmor -profil,
   Apparmor -kärnmodulen måste installeras och aktiveras.Flera distributioner aktiverar modulen av
   Standard, till exempel Ubuntu och SUSE, och många andra ger valfritt stöd.För att kontrollera om
   modulen är aktiverad, kontrollera `/sys/module/apparmor/parameters/enabled` file:

   ```shell
   cat /sys/module/apparmor/parameters/enabled
   Y
   ```

   Om din Kubelet innehåller Apparmor Support (> = v1.4) kommer det att vägra att köra en pod med Apparmor
   Alternativ om kärnmodulen inte är aktiverad.

  {{< note >}}
Ubuntu bär många Apparmor -fläckar som inte har lagts samman i uppströms Linux
  Kärna, inklusive lappar som lägger till ytterligare krokar och funktioner.Kubernetes har bara varit
  Testat med uppströmsversionen och lovar inte stöd för andra funktioner.
  {{< /note >}}

3. Container Runtime stöder Apparmor-För närvarande alla vanliga Kubernetes-stödda behållare
   RunTimes bör stödja Apparmor, som {{<GLOSSARY_TOOLTIP TEM_ID = "DOCKER">}},
   {{<GLOSSARY_TOOLTIP TEM_ID = "CRI-O">}} eller {{<GLOSSARY_TOOLTIP TEM_ID = "containerd">}}.
   Se motsvarande runtime -documentation och verifiera att klustret uppfyller
   kraven för att använda Apparmor.

4. Profilen laddas - Apparmor appliceras på en pod genom att ange en Apparmor -profil som var och en
   Behållaren ska köras med.Om någon av de angivna profilerna inte redan är laddade i
   Kärnan, Kubelet (> = v1.4) kommer att avvisa poden.Du kan se vilka profiler som är laddade på en
   nod genom att kontrollera `/sys/kernel/security/apparmor/profiles` fil. Till exempel:

   ```shell
   ssh gke-test-default-pool-239f5d02-gyn2 "sudo cat /sys/kernel/security/apparmor/profiles | sort"
   ```
   ```
   apparmor-test-deny-write (enforce)
   apparmor-test-audit-write (enforce)
   docker-default (enforce)
   k8s-nginx (enforce)
   ```

   För mer information om laddningsprofiler på noder, se
   [Uppsättning av noder med profiler](#setting-up-nodes-with-profiles).

Så länge som Kubelet-versionen inkluderar Apparmor Support (> = v1.4) kommer Kubelet att avvisa en pod
med Apparmor-alternativ om någon av förutsättningarna inte uppfylls. Du kan också verifiera Apparmor Support
På noder genom att kontrollera meddelandet Node Ready Condition (även om detta sannolikt kommer att tas bort i en
senare släpp):

```shell
kubectl get nodes -o=jsonpath=$'{range .items[*]}{@.metadata.name}: {.status.conditions[?(@.reason=="KubeletReady")].message}\n{end}'
```
```
gke-test-default-pool-239f5d02-gyn2: kubelet is posting ready status. AppArmor enabled
gke-test-default-pool-239f5d02-x1kf: kubelet is posting ready status. AppArmor enabled
gke-test-default-pool-239f5d02-xwux: kubelet is posting ready status. AppArmor enabled
```



<!-- lessoncontent -->

## Securing a Pod

{{< note >}}
Apparmor är för närvarande i beta, så alternativ anges som kommentarer.En gång stödjer akademiker till
Allmän tillgänglighet, annoteringarna kommer att ersättas med förstklassiga fält (mer information i
[Uppgradera sökvägen till GA](#upgrade-path-to-general-availability)).
{{< /note >}}

Apparmor-profiler anges *per container *. För att ange Apparmor-profilen för att köra en pod
Behållare med, lägg till en kommentar till POD: s metadata:

```yaml
container.apparmor.security.beta.kubernetes.io/<container_name>: <profile_ref>
```

`<container_name>` är namnet på behållaren att tillämpa profilen på och `<profile_ref>`
anger profilen som ska tillämpas. `profile_ref` kan vara en av:

* `runtime/default` för att tillämpa Runtime: s standardprofil
* `localhost/<profile_name>` för att tillämpa profilen laddad på värden med namnet`<profile_name>`
* `unconfined` för att indikera att inga profiler kommer att laddas

Titta på [API Referensen](#api-reference) För fullständiga detaljer om kommentar- och profilnamnformat.

Kubernetes Apparmor Enforcement arbetar genom att först kontrollera att alla förutsättningar har varit
träffades, och sedan vidarebefordra profilvalet till containern runtime för verkställighet.Om
Förutsättningar har inte uppfyllts, poden kommer att avvisas och kommer inte att köras.

För att verifiera att profilen tillämpades kan du leta efter alternativet Apparmor Security som anges i container skapade evenemang:

```shell
kubectl get events | grep Created
```
```
22s        22s         1         hello-apparmor     Pod       spec.containers{hello}   Normal    Created     {kubelet e2e-test-stclair-node-pool-31nt}   Created container with docker id 269a53b202d3; Security:[seccomp=unconfined apparmor=k8s-apparmor-example-deny-write]
```

You can also verify directly that the container's root process is running with the correct profile by checking its proc attr:

```shell
kubectl exec <pod_name> cat /proc/1/attr/current
```
```
k8s-apparmor-example-deny-write (enforce)
```

## Example

*This example assumes you have already set up a cluster with AppArmor support.*

First, we need to load the profile we want to use onto our nodes. This profile denies all file writes:

```shell
#include <tunables/global>

profile k8s-apparmor-example-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
```

Since we don't know where the Pod will be scheduled, we'll need to load the profile on all our
nodes. For this example we'll use SSH to install the profiles, but other approaches are
discussed in [Setting up nodes with profiles](#setting-up-nodes-with-profiles).

```shell
NODES=(
    # The SSH-accessible domain names of your nodes
    gke-test-default-pool-239f5d02-gyn2.us-central1-a.my-k8s
    gke-test-default-pool-239f5d02-x1kf.us-central1-a.my-k8s
    gke-test-default-pool-239f5d02-xwux.us-central1-a.my-k8s)
for NODE in ${NODES[*]}; do ssh $NODE 'sudo apparmor_parser -q <<EOF
#include <tunables/global>

profile k8s-apparmor-example-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
EOF'
done
```

Next, we'll run a simple "Hello AppArmor" pod with the deny-write profile:

{{< codenew file="pods/security/hello-apparmor.yaml" >}}

```shell
kubectl create -f ./hello-apparmor.yaml
```

Om vi tittar på podhändelserna kan vi se att podbehållaren skapades med Apparmor
Profil "K8S-Apparmor-Exempel-DENY-WRITE":

```shell
kubectl get events | grep hello-apparmor
```
```
14s        14s         1         hello-apparmor   Pod                                Normal    Scheduled   {default-scheduler }                           Successfully assigned hello-apparmor to gke-test-default-pool-239f5d02-gyn2
14s        14s         1         hello-apparmor   Pod       spec.containers{hello}   Normal    Pulling     {kubelet gke-test-default-pool-239f5d02-gyn2}   pulling image "busybox"
13s        13s         1         hello-apparmor   Pod       spec.containers{hello}   Normal    Pulled      {kubelet gke-test-default-pool-239f5d02-gyn2}   Successfully pulled image "busybox"
13s        13s         1         hello-apparmor   Pod       spec.containers{hello}   Normal    Created     {kubelet gke-test-default-pool-239f5d02-gyn2}   Created container with docker id 06b6cd1c0989; Security:[seccomp=unconfined apparmor=k8s-apparmor-example-deny-write]
13s        13s         1         hello-apparmor   Pod       spec.containers{hello}   Normal    Started     {kubelet gke-test-default-pool-239f5d02-gyn2}   Started container with docker id 06b6cd1c0989
```

We can verify that the container is actually running with that profile by checking its proc attr:

```shell
kubectl exec hello-apparmor -- cat /proc/1/attr/current
```
```
k8s-apparmor-example-deny-write (enforce)
```

Finally, we can see what happens if we try to violate the profile by writing to a file:

```shell
kubectl exec hello-apparmor -- touch /tmp/test
```
```
touch: /tmp/test: Permission denied
error: error executing remote command: command terminated with non-zero exit code: Error executing in Docker Container: 1
```

To wrap up, let's look at what happens if we try to specify a profile that hasn't been loaded:

```shell
kubectl create -f /dev/stdin <<EOF
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor-2
  annotations:
    container.apparmor.security.beta.kubernetes.io/hello: localhost/k8s-apparmor-example-allow-write
spec:
  containers:
  - name: hello
    image: busybox:1.28
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]
EOF
pod/hello-apparmor-2 created
```

```shell
kubectl describe pod hello-apparmor-2
```
```
Name:          hello-apparmor-2
Namespace:     default
Node:          gke-test-default-pool-239f5d02-x1kf/
Start Time:    Tue, 30 Aug 2016 17:58:56 -0700
Labels:        <none>
Annotations:   container.apparmor.security.beta.kubernetes.io/hello=localhost/k8s-apparmor-example-allow-write
Status:        Pending
Reason:        AppArmor
Message:       Pod Cannot enforce AppArmor: profile "k8s-apparmor-example-allow-write" is not loaded
IP:
Controllers:   <none>
Containers:
  hello:
    Container ID:
    Image:     busybox
    Image ID:
    Port:
    Command:
      sh
      -c
      echo 'Hello AppArmor!' && sleep 1h
    State:              Waiting
      Reason:           Blocked
    Ready:              False
    Restart Count:      0
    Environment:        <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-dnz7v (ro)
Conditions:
  Type          Status
  Initialized   True
  Ready         False
  PodScheduled  True
Volumes:
  default-token-dnz7v:
    Type:    Secret (a volume populated by a Secret)
    SecretName:    default-token-dnz7v
    Optional:   false
QoS Class:      BestEffort
Node-Selectors: <none>
Tolerations:    <none>
Events:
  FirstSeen    LastSeen    Count    From                        SubobjectPath    Type        Reason        Message
  ---------    --------    -----    ----                        -------------    --------    ------        -------
  23s          23s         1        {default-scheduler }                         Normal      Scheduled     Successfully assigned hello-apparmor-2 to e2e-test-stclair-node-pool-t1f5
  23s          23s         1        {kubelet e2e-test-stclair-node-pool-t1f5}             Warning        AppArmor    Cannot enforce AppArmor: profile "k8s-apparmor-example-allow-write" is not loaded
```
Obs! POD -statusen väntar, med ett användbart felmeddelande: `POD kan inte verkställa Apparmor: Profil
"K8S-Apparmor-Exempel-tilläggsskrivning" är inte laddad ".En händelse spelades också in med samma meddelande.
## Administration

### Setting up nodes with profiles

Kubernetes tillhandahåller för närvarande inga infödda mekanismer för att ladda Apparmor -profiler på
knutpunkter.Det finns dock många sätt att ställa in profilerna, till exempel:
* Through a [DaemonSet](/docs/concepts/workloads/controllers/daemonset/) that runs a Pod on each node to
  ensure the correct profiles are loaded. An example implementation can be found
  [here](https://git.k8s.io/kubernetes/test/images/apparmor-loader).
* At node initialization time, using your node initialization scripts (e.g. Salt, Ansible, etc.) or
  image.
* By copying the profiles to each node and loading them through SSH, as demonstrated in the
  [Example](#example).

The scheduler is not aware of which profiles are loaded onto which node, so the full set of profiles
must be loaded onto every node.  An alternative approach is to add a node label for each profile (or
class of profiles) on the node, and use a
[node selector](/docs/concepts/scheduling-eviction/assign-pod-node/) to ensure the Pod is run on a
node with the required profile.

### Restricting profiles with the PodSecurityPolicy

{{< note >}}
PodSecurityPolicy is deprecated in Kubernetes v1.21, and will be removed in v1.25.
See [PodSecurityPolicy](/docs/concepts/security/pod-security-policy/) documentation for more information.
{{< /note >}}

If the PodSecurityPolicy extension is enabled, cluster-wide AppArmor restrictions can be applied. To
enable the PodSecurityPolicy, the following flag must be set on the `apiserver`:

```
--enable-admission-plugins=PodSecurityPolicy[,others...]
```

The AppArmor options can be specified as annotations on the PodSecurityPolicy:

```yaml
apparmor.security.beta.kubernetes.io/defaultProfileName: <profile_ref>
apparmor.security.beta.kubernetes.io/allowedProfileNames: <profile_ref>[,others...]
```

Standardprofilnamnsalternativet Anger profilen som ska tillämpas på containrar som standard när ingen är
specificerad.Alternativet för tillåtna profilnamn anger en lista över profiler som pod containrar är
får köras med.Om båda alternativen tillhandahålls måste standard tillåtas.Profilerna är
specificeras i samma format som på containrar.Se [API-referensen] (#API-referens) för full
Specifikation.

### Disabling AppArmor

If you do not want AppArmor to be available on your cluster, it can be disabled by a command-line flag:

```
--feature-gates=AppArmor=false
```

When disabled, any Pod that includes an AppArmor profile will fail validation with a "Forbidden"
error.

{{<note>}}
Även om Kubernetes -funktionen är inaktiverad kan körtider fortfarande upprätthålla standardprofilen.De
Möjlighet för att inaktivera Apparmor -funktionen kommer att tas bort när Apparmor examen till general
Tillgänglighet (GA).
{{</note>}}

## Authoring Profiles

Getting AppArmor profiles specified correctly can be a tricky business. Fortunately there are some
tools to help with that:

* `aa-genprof` and `aa-logprof` generate profile rules by monitoring an application's activity and
  logs, and admitting the actions it takes. Further instructions are provided by the
  [AppArmor documentation](https://gitlab.com/apparmor/apparmor/wikis/Profiling_with_tools).
* [bane](https://github.com/jfrazelle/bane) is an AppArmor profile generator for Docker that uses a
  simplified profile language.

To debug problems with AppArmor, you can check the system logs to see what, specifically, was
denied. AppArmor logs verbose messages to `dmesg`, and errors can usually be found in the system
logs or through `journalctl`. More information is provided in
[AppArmor failures](https://gitlab.com/apparmor/apparmor/wikis/AppArmor_Failures).


## API Reference

### Pod Annotation

Specifying the profile a container will run with:

- **key**: `container.apparmor.security.beta.kubernetes.io/<container_name>`
  Where `<container_name>` matches the name of a container in the Pod.
  A separate profile can be specified for each container in the Pod.
- **value**: a profile reference, described below

### Profile Reference

- `runtime/default`: Refers to the default runtime profile.
  - Equivalent to not specifying a profile (without a PodSecurityPolicy default), except it still
    requires AppArmor to be enabled.
  - In practice, many container runtimes use the same OCI default profile, defined here:
    https://github.com/containers/common/blob/main/pkg/apparmor/apparmor_linux_template.go
- `localhost/<profile_name>`: Refers to a profile loaded on the node (localhost) by name.
  - The possible profile names are detailed in the
    [core policy reference](https://gitlab.com/apparmor/apparmor/wikis/AppArmor_Core_Policy_Reference#profile-names-and-attachment-specifications).
- `unconfined`: This effectively disables AppArmor on the container.

Any other profile reference format is invalid.

### PodSecurityPolicy Annotations

Specifying the default profile to apply to containers when none is provided:

* **key**: `apparmor.security.beta.kubernetes.io/defaultProfileName`
* **value**: a profile reference, described above

Specifying the list of profiles Pod containers is allowed to specify:

* **key**: `apparmor.security.beta.kubernetes.io/allowedProfileNames`
* **value**: a comma-separated list of profile references (described above)
  - Although an escaped comma is a legal character in a profile name, it cannot be explicitly
    allowed here.



## {{% heading "whatsnext" %}}


Additional resources:

* [Quick guide to the AppArmor profile language](https://gitlab.com/apparmor/apparmor/wikis/QuickProfileLanguage)
* [AppArmor core policy reference](https://gitlab.com/apparmor/apparmor/wikis/Policy_Layout)
