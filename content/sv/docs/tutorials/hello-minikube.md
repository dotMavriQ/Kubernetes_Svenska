---
title: Hallå Minikube
content_type: guide
weight: 5
menu:
  main:
    title: "Kom igång"
    weight: 10
    post: >
      <p>Redo att sätta igång? Bygg ett enkelt Kuberneteskluster som kör en test-app.</p>
card:
  name: tutorials
  weight: 10
---

<!-- overview -->

Denna handledning visar hur du kör en provapp
på Kubernetes med Minikube och Katacoda.
Katacoda tillhandahåller en gratis kubernetesmiljö i webbläsaren.

{{< note >}}
Du kan också följa denna handledning om du har installerat Minikube lokalt.
Se [minikube start](https://minikube.sigs.k8s.io/docs/start/) för installationsinstruktioner.
{{< /note >}}

## {{% heading "mål" %}}

* Distribuera en test-app på Minikube.
* Kör appen.
* Kolla applikationens loggar.

## {{% heading "förberedelser" %}}


Denna handledning ger en container-image som använder nginx för att återkalla alla förfrågningar.



<!-- lessoncontent -->

## Skapa ett minikube-kluster

1. Klicka **Launch Terminal**

    {{< kat-button >}}

{{< note >}}
Om du installerade minikube lokalt, kör `minikube start`. Innan du kör `minikube dashboard` så borde du öppna en ny terminal, 
starta `minikube dahboard` där och sedan byta tillbaka till huvudterminalen.
{{< /note >}}

1. Öppna Kubernetes Dashboard i en webbläsare:

    ```shell
    minikube dashboard
    ```

2. Endast för Katacoda: Högst upp i terminalpanelen så klickar du på plustecknet och sedan på **Select port to view on Host 1**.

3. Endast för Katacoda: Skriv `30000` och klicka sedan på **Display Port**.

{{< note >}}
`dashboard` kommandot tillåter add-on för dashboard och öppnar en proxy i webbläsaren.
You can create Kubernetes resources on the dashboard such as Deployment and Service.

Om du kör som root, se [Open Dashboard with URL](#open-dashboard-with-url).

Som standard är instrumentpanelen endast tillgänglig från det interna Kubernetes virtuella nätverket.
`dashboard` kommandot skapar en tillfällig proxy för att göra instrumentpanelen tillgänglig utanför Kubernetes virtuella nätverk.

För att stoppa proxy, kör `Ctrl+C` för att lämna processen.
Efter kommandot avslutats så körs fortfarande kommandot i Kubernetesklustret.
Du kan köra `dashboard` igen för att skapa en annan proxy med vilket du kan nå dashboard.
{{< /note >}}

## Öppna dashboard från en URL

Om du inte vill öppna en webbläsare, kör dashboard med `--url` flag för att dela ut en URL:

```shell
minikube dashboard --url
```

## Skapa ett Deployment

En Kubernetes [*Pod*](/docs/concepts/workloads/pods/) är en grupp av en eller flera Containers,
sammanbundna för administrering eller nätverksfunktionalitet. Podden i denna
guide har bara en Container. Ett Kubernetes
[*Deployment*](/docs/concepts/workloads/controllers/deployment/) kollar hälsotillståndet på din
Pod och startar om poddars containrar om den termineras. Deployments är det rekommenderade sättet att underhålla poddar på.

1. Använd `kubectl create` för att skapa ett Deployment som hanterar en Pod. 
Podden kör en Container baserad på vilken Docker-image den tilldelats.
Pod runs a Container based on the provided Docker image.

    ```shell
    kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4
    ```

2. Se över din Deployment:

    ```shell
    kubectl get deployments
    ```

    Resultatet efterliknar:

    ```
    NAME         READY   UP-TO-DATE   AVAILABLE   AGE
    hello-node   1/1     1            1           1m
    ```

3. Se över din Pod:

    ```shell
    kubectl get pods
    ```

    Resultat är som:

    ```
    NAME                          READY     STATUS    RESTARTS   AGE
    hello-node-5f76cf6ccf-br9b5   1/1       Running   0          1m
    ```

4. Se över kluster-event:

    ```shell
    kubectl get events
    ```

5. Få en överblick på din `kubectl` konfiguration:

    ```shell
    kubectl config view
    ```

{{< note >}}
För mer information om `kubectl` kommandon, se [kubectl overview](/docs/reference/kubectl/).
{{< /note >}}

## Skapa en Service

Som standard är poden endast tillgänglig med sin interna IP-adress i ett
Kuberneteskluster. För att göra containern `Hello-Node` tillgänglig utanför
Kubernetes virtuella nätverk så måste du göra podden till en
Kubernetes [*Service*](/docs/concepts/services-networking/service/).

1. Exponera poden för det offentliga internet med hjälp av `kubectl expose` :

    ```shell
    kubectl expose deployment hello-node --type=LoadBalancer --port=8080
    ```

    Flaggen `--type=LoadBalancer` visar att du vill avslöja din Service
    utanför klustret.
    
    Applikationskoden innuti din image `k8s.gcr.io/echoserver` lyssnar endast på TPC port 8080. Om du använde
    `kubectl expose` för att avslöja en annan port kunde kunder inte ansluta till den andra porten.

2. Se över den Service du skapat:

    ```shell
    kubectl get services
    ```

    En liknande output är:

    ```
    NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    hello-node   LoadBalancer   10.108.144.78   <pending>     8080:30369/TCP   21s
    kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP          23m
    ```

    På molnleverantörer som stöder lastbalanserare,
    En extern IP-adress skulle tillhandahållas för att komma åt tjänsten. På minikube så är
     `LoadBalancer` typen det som gör din Service tillgänglig genom `minikube service`.

3. Kör följande kommando:

    ```shell
    minikube service hello-node
    ```

4. Enbart för Katacoda: Click the plus sign, and then click **Select port to view on Host 1**.

5. Enbart för Katacoda: Observera det 5-siffriga portnumret som visas motsatsen till `8080` i output för Services. Detta portnummer genereras slumpmässigt och det kan vara annorlunda för dig.Skriv ditt nummer i textrutan Portnummer och klicka sedan på Visa port. Med exemplet från tidigare ska du skriva `30369`.

   Detta öppnar upp ett webbläsarfönster som serverar din app och visar appens svar.

## Tillåt addons

Minikube-verktyget innehåller en uppsättning inbyggda{{< glossary_tooltip text="addons" term_id="addons" >}} som kan aktiveras, inaktiveras och öppnas i den lokala Kubernetes-miljön.

1. Lista de för närvarande stödda tillägg:

    ```shell
    minikube addons list
    ```

    Resultatet liknar:

    ```
    addon-manager: enabled
    dashboard: enabled
    default-storageclass: enabled
    efk: disabled
    freshpod: disabled
    gvisor: disabled
    helm-tiller: disabled
    ingress: disabled
    ingress-dns: disabled
    logviewer: disabled
    metrics-server: disabled
    nvidia-driver-installer: disabled
    nvidia-gpu-device-plugin: disabled
    registry: disabled
    registry-creds: disabled
    storage-provisioner: enabled
    storage-provisioner-gluster: disabled
    ```

2. Aktivera till exempel ett tillägg `metrics-server`:

    ```shell
    minikube addons enable metrics-server
    ```

    Resultatet liknar:

    ```
    The 'metrics-server' addon is enabled
    ```

3. Visa poden och tjänsten du skapade:

    ```shell
    kubectl get pod,svc -n kube-system
    ```

    Resultatet liknar:

    ```
    NAME                                        READY     STATUS    RESTARTS   AGE
    pod/coredns-5644d7b6d9-mh9ll                1/1       Running   0          34m
    pod/coredns-5644d7b6d9-pqd2t                1/1       Running   0          34m
    pod/metrics-server-67fb648c5                1/1       Running   0          26s
    pod/etcd-minikube                           1/1       Running   0          34m
    pod/influxdb-grafana-b29w8                  2/2       Running   0          26s
    pod/kube-addon-manager-minikube             1/1       Running   0          34m
    pod/kube-apiserver-minikube                 1/1       Running   0          34m
    pod/kube-controller-manager-minikube        1/1       Running   0          34m
    pod/kube-proxy-rnlps                        1/1       Running   0          34m
    pod/kube-scheduler-minikube                 1/1       Running   0          34m
    pod/storage-provisioner                     1/1       Running   0          34m

    NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
    service/metrics-server         ClusterIP   10.96.241.45    <none>        80/TCP              26s
    service/kube-dns               ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP       34m
    service/monitoring-grafana     NodePort    10.99.24.54     <none>        80:30002/TCP        26s
    service/monitoring-influxdb    ClusterIP   10.111.169.94   <none>        8083/TCP,8086/TCP   26s
    ```

4. Stäng av `metrics-server`:

    ```shell
    minikube addons disable metrics-server
    ```

    Resultatet liknar:

    ```
    metrics-server was successfully disabled
    ```

## Städa upp

Nu kan du städa upp resurserna du skapade i ditt kluster:

```shell
kubectl delete service hello-node
kubectl delete deployment hello-node
```

Eller så kan du stoppa Minikube Virtual Machine (VM):

```shell
minikube stop
```

Eller bara ta bort den:

```shell
minikube delete
```



## {{% heading "whatsnext" %}}


* Learn more about [Deployment objects](/docs/concepts/workloads/controllers/deployment/).
* Learn more about [Deploying applications](/docs/tasks/run-application/run-stateless-application-deployment/).
* Learn more about [Service objects](/docs/concepts/services-networking/service/).

