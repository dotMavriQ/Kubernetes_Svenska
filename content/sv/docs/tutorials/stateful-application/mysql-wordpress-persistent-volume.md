---
Titel: "Exempel: Distribuera WordPress och MySQL med ihållande volymer" "" "
Granskare:
- Ahmetb
content_type: handledning
Vikt: 20
kort:
  Namn: Tutorials
  Vikt: 40
  Titel: "Statligt exempel: WordPress med ihållande volymer"
---

<!-Översikt->
Denna handledning visar hur du distribuerar en WordPress -webbplats och en MySQL -databas med Minikube. Båda applikationerna använder persistentvolumer och persistentvolumeclaims för att lagra data.

A [PersistentVolume] (/Docs/Concepts/Storage/Persistent-Volumes/) (PV) är en lagringsdel i klustret som manuellt har tillhandahållits av en administratör, eller dynamiskt tillhandahålls av Kubernetes som använder en [StorageClass] (/docs /koncept/lagring/lagringsklasser). A [PersistentVolumeClaim] (/docs/concepts/Storage/Persistent-Volumes/#PersistentVolumeClaims) (PVC) är en begäran om lagring av en användare som kan uppfyllas av en PV. Persistentvolumes och PersistentVolumeClaims är oberoende av POD -livscykler och bevarar data genom att starta om, omplanera och till och med ta bort skidor.

{{<warning>}}
Denna distribution är inte lämplig för fall av produktionsanvändning, eftersom den använder enstaka instans WordPress och MySQL Pods. Överväg att använda [WordPress Helm Chart] (https://github.com/bitnami/Charts/Tree/master/bitnami/wordpress) för att distribuera WordPress i produktionen.
{{< /warning>}}

{{<Note>}}
Filerna som tillhandahålls i denna handledning använder GA -distribution API: er och är specifika för Kubernetes version 1.9 och senare. Om du vill använda denna handledning med en tidigare version av Kubernetes, uppdatera API -versionen på lämpligt sätt eller referera tidigare versioner av denna handledning.
{{</note>}}



## {{ % rubrik "objekt" %}}

* Skapa persistentvolumeclaims och persistentvolumes
* Skapa en `kustomization.yaml` med
  * En hemlig generator
  * MySQL Resource Configs
  * WordPress Resource Configs
* Applicera kustomiseringskatalogen med `Kubectl Apply -K./`
* Städa

## {{ % rubrik "Förutsättningar" %}}


{{<inkludera "Task-Tutorial-Prereqs.md">}} {{<version-check>}}
Exemplet som visas på denna sida fungerar med `Kubectl` 1.14 och högre.

Ladda ner följande konfigurationsfiler:

1. [MySQL-DEPLEASMENT.YAML] (/Exempel/Application/WordPress/MySQL-DEPLEAKMENT.YAML)

1. [WordPress-DEployment.YAML] (/Exempel/Application/WordPress/WordPress-DEployment.YAML)



<!-Lessoncontent->

## skapa persistentvolumeclaims och persistentvolumes

MySQL och WordPress kräver vardera en ihållandeVolume för att lagra data. Deras persistentvolumeclaims kommer att skapas vid utplaceringssteget.

Många klustermiljöer har en standard StorageClass installerad. När en StorageClass inte specificeras i PersistentVolumeClaim, används klusterets standard StorageClass istället.

När en persistentvolumeclaim skapas, tillhandahålls en persistentvolym dynamiskt baserat på StorageClass -konfigurationen.

{{<warning>}}
I lokala kluster använder standard StorageClass leveransen "Hostpath". "Hostpath` volymer är bara lämpliga för utveckling och testning. Med `HostPath` -volymer lever dina data i`/tmp` på noden är poden planerad på och rör sig inte mellan noder. Om en pod dör och blir planerad till en annan nod i klustret, eller noden startas om, går data förlorade.
{{< /warning>}}

{{<Note>}}
Om du tar upp ett kluster som måste använda flaggan "HostPath", måste `-Enable-HostPath-Provisioner '-flaggan ställas in i komponenten" Controller-Manager ".
{{</note>}}

{{<Note>}}
Om du har ett Kubernetes-kluster som körs på Google Kubernetes-motor, följ [den här guiden] (https://cloud.google.com/kubernetes-gine/docs/tutorials/persistent-disk).
{{</note>}}
## skapa en kustomization.yaml

### Lägg till en hemlig generator
A [Secret] (/docs/Concepts/Configuration/Secret/) är ett objekt som lagrar en bit känslig data som ett lösenord eller nyckel. Sedan 1.14 stöder `Kubectl` hanteringen av Kubernetes -objekt med en kustomiseringsfil. Du kan skapa en hemlighet av generatorer i `kustomization.yaml`.

Lägg till en hemlig generator i `kustomization.yaml` från följande kommando. Du måste byta ut `your_password 'med lösenordet du vill använda.

`` `Skal
Cat << eof> ./ Kustomization.yaml
SecretGenerator:
- Namn: Mysql-pass
  bokstäver:
  - lösenord = your_password
Eof
`` `

## Lägg till resurskonfigurer för MySQL och WordPress

Följande manifest beskriver en enda instans MySQL-distribution. MySQL -behållaren monterar den persistententvolume at/var/lib/mysql. Miljövariabeln `MySQL_ROOT_PASSWORD 'ställer in databaslösenordet från hemligheten.

{{<codeenew file = "Application/WordPress/mysql-deploy.yaml">}}

Följande manifest beskriver en enda instans WordPress-distribution. WordPress -behållaren monteras
Persistentvolume på `/var/www/html` för webbplatsdatafiler. Miljövariabeluppsättningarna "WordPress_db_Host"
Namnet på MySQL -tjänsten som definieras ovan och WordPress kommer åt databasen efter tjänst. De
`WordPress_DB_Password 'Miljövariabel Ställer in databaslösenordet från den hemliga kustomize genererade.

{{<codeenew file = "Application/WordPress/WordPress-Deproploy.yaml">}}

1. Ladda ner MySQL -distributionskonfigurationsfilen.

      `` `Skal
      curl -lo https://k8s.io/examples/application/wordpress/mysql-deploy.yaml
      `` `
        2. Ladda ner WordPress -konfigurationsfilen.

      `` `Skal
      curl -lo https://k8s.io/examples/application/wordpress/wordpress-deployment.yaml
      `` `
      
3. Lägg till dem i filen "kustomization.yaml`.

`` `Skal
Cat << eof >> ./ Kustomization.yaml
Resurser:
  - mysql-deploy.yaml
  - WordPress-DEployment.yaml
Eof
`` `

## Applicera och verifiera
`Kustomization.yaml` innehåller alla resurser för att distribuera en WordPress -webbplats och en
MySQL -databas. Du kan tillämpa katalogen av
`` `Skal
Kubectl Apply -K ./
`` `

Nu kan du verifiera att alla objekt finns.

1. Kontrollera att hemligheten finns genom att köra följande kommando:

      `` `Skal
      kubectl få hemligheter
      `` `

      Svaret ska vara så här:

      `` `Skal
      Namntyp Dataålder
      MySQL-Pass-C57BB4T7MF Opaque 1 9S
      `` `

2. Kontrollera att en persistentvolume fick dynamiskt tillhandahållen.
 
      `` `Skal
      kubectl get pvc
      `` `
      
      {{<Note>}}
      Det kan ta upp till några minuter innan PV: erna ska tillhandahållas och bundas.
      {{</note>}}

      Svaret ska vara så här:

      `` `Skal
      Namn Status Volym Kapacitet Tillgångslägen
      MySQL-PV-Claim Bound PVC-8CBD7B2E-4044-11E9-B2BB-42010A800002 20GI RWO Standard 77S
      WP-PV-Claim Bound PVC-8CD0DF54-4044-11E9-B2BB-42010A800002 20GI RWO Standard 77S
      `` `

3. Kontrollera att poden körs genom att köra följande kommando:

      `` `Skal
      kubectl get pods
      `` `

      {{<Note>}}
      Det kan ta upp till några minuter för att podens status är "igång".
      {{</note>}}

      Svaret ska vara så här:

      `` `
      Namn redo status startar om ålder
      WordPress-MysQL-1894417608-X5DZT 1/1 kör 0 40-talet
      `` `
4. Kontrollera att tjänsten körs genom att köra följande kommando:

      `` `Skal
      kubectl få tjänster wordpress
      `` `

      Svaret ska vara så här:

      `` `
      Namntyp Cluster-IP Extern-IP-port (er) ålder
      WordPress LoadBalancer 10.0.0.89 <Pending> 80: 32406/TCP 4M
      `` `

      {{<Note>}}
      Minikube kan bara exponera tjänster via `NodePort '. Extern-IP väntar alltid.
      {{</note>}}

5. Kör följande kommando för att få IP -adressen för WordPress -tjänsten:

      `` `Skal
      Minikube Service WordPress --url
      `` `

      Svaret ska vara så här:

      `` `
      http://1.2.3.4:32406
      `` `

6. Kopiera IP -adressen och ladda sidan i din webbläsare för att se din webbplats.

   Du bör se WordPress -inställningssidan som liknar följande skärmdump.

   ! [WordPress-Init] (https://raw.githubusercontent.com/kubernetes/examples/master/mysql-wordpress-pd/wordpress.png)

{{<warning>}}
Lämna inte din WordPress -installation på den här sidan. Om en annan användare hittar det kan de ställa in en webbplats på din instans och använda den för att tjäna skadligt innehåll. <br/> <br/> installerar antingen WordPress genom att skapa ett användarnamn och lösenord eller ta bort din instans.
{{< /warning>}}
## {{ % rubrik "sanering" %}}


1. Kör följande kommando för att ta bort din hemlighet, distributioner, tjänster och persistentvolumeclaims:

      `` `Skal
      kubectl delete -k ./
      `` `



## {{ % rubrik "WhatsNext" %}}


* Lär dig mer om [introspektion och felsökning] (/dokument/uppgifter/felsökning/felsökning/felsökning-pod/)
* Lär dig mer om [jobb] (/docs/koncept/arbetsbelastningar/kontroller/jobb/)
* Lär dig mer om [Port vidarebefordran] (/DOCS/TASKS/Access-Application-Cluster/Port-Forward-Access-Application-Cluster/)
* Lär dig hur man får ett skal till en container] (/docs/uppgifter/felsökning/felsökning/get-shell-running-container/)