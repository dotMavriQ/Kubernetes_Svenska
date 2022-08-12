# The Kubernetes documentation

[![Netlify Status](https://api.netlify.com/api/v1/badges/be93b718-a6df-402a-b4a4-855ba186c97d/deploy-status)](https://app.netlify.com/sites/kubernetes-io-main-staging/deploys) [![GitHub release](https://img.shields.io/github/release/kubernetes/website.svg)](https://github.com/kubernetes/website/releases/latest)

Detta repo innehåller de tillgångar som krävs för att bygga[Kubernetes webbplats och dokumentation](https://kubernetes.io/). Vi är glada att du vill bidra!

- [Contributing to the docs](#contributing-to-the-docs)
- [Localization ReadMes](#localization-readmemds)

## Using this repository

Du kan köra webbplatsen lokalt med Hugo (Extended version), eller så kan du köra den i en containerruntime. Vi rekommenderar starkt att du använder en container, eftersom den ger distributionskonsistens med livewebbplatsen.

## Grundförutsättningar

Du behöver ha följande programvara installerad lokalt för att kunna använda detta repo:

- [npm](https://www.npmjs.com/)
- [Go](https://golang.org/)
- [Hugo (Extended version)](https://gohugo.io/)
- [Docker](https://www.docker.com/) eller liknande mjukvara.

För att använda det här arkivet behöver du följande installerat lokalt:


```bash
git clone https://github.com/kubernetes/website.git
cd website
```

Kubernetes webbplats använder [Docsy Hugo-temat](https://github.com/google/docsy#readme). Även om du planerar att köra webbplatsen i en container, rekommenderar vi starkt att du drar in undermodulen och andra utvecklingsberoenden genom att köra följande:

```bash
# pull in the Docsy submodule
git submodule update --init --recursive --depth 1
```

## Running the website using a container

För att bygga webbplatsen i en container, kör följande:

```bash
# Du kan ställa in $CONTAINER_ENGINE till namnet på vilket Docker-liknande containerverktyg som helst
make container-serve
```

Om du ser fel betyder det förmodligen att hugo-behållaren inte hade tillräckligt med datorresurser tillgängliga. För att lösa det, öka mängden tillåten CPU och minnesanvändning för Docker på din maskin ([MacOSX](https://docs.docker.com/docker-for-mac/#resources) och [Windows](https:/ /docs.docker.com/docker-for-windows/#resources)).

Öppna din webbläsare till <http://localhost:1313> för att visa webbplatsen. När du gör ändringar i källfilerna uppdaterar Hugo webbplatsen och tvingar fram en uppdatering av webbläsaren.

## Kör webbplatsen lokalt med Hugo

Se till att installera Hugo utökade version som specificeras av miljövariabeln `HUGO_VERSION` i filen [`netlify.toml`](netlify.toml#L10).

För att bygga och testa webbplatsen lokalt, kör:

```bash
# install dependencies
npm ci
make serve
```

Detta kommer att starta den lokala Hugo-servern på port 1313. Öppna din webbläsare till <http://localhost:1313> för att se webbplatsen. När du gör ändringar i källfilerna uppdaterar Hugo webbplatsen och tvingar fram en uppdatering av webbläsaren.

## Bygga API-referenssidor

API-referenssidorna som finns i `content/en/docs/reference/kubernetes-api` är byggda från Swagger-specifikationen, med <https://github.com/kubernetes-sigs/reference-docs/tree/master/gen- resourcesdocs>.

Följ dessa steg för att uppdatera referenssidorna för en ny Kubernetes-version:

1. Dra in "api-ref-generator" undermodulen:

   ```bash
   git submodule update --init --recursive --depth 1
   ```

2. Uppdatera Swagger-specifikationen:

   ```bash
   curl 'https://raw.githubusercontent.com/kubernetes/kubernetes/master/api/openapi-spec/swagger.json' > api-ref-assets/api/swagger.json
   ```

3. I `api-ref-assets/config/`, anpassa filerna `toc.yaml` och `fields.yaml` för att återspegla ändringarna i den nya utgåvan.

4. Bygg sedan sidorna:

   ```bash
   make api-reference
   ```

   Du kan testa resultaten lokalt genom att skapa och betjäna webbplatsen från en behållarbild:

   ```bash
   make container-image
   make container-serve
   ```

   I en webbläsare går du till <http://localhost:1313/docs/reference/kubernetes-api/> för att se API-referensen.

5. När alla ändringar av det nya kontraktet återspeglas i konfigurationsfilerna `toc.yaml` och `fields.yaml`, skapa en Pull Request med de nyligen genererade API-referenssidorna.

## Felsökning

### error: failed to transform resource: TOCSS: failed to transform "scss/main.scss" (text/x-scss): this feature is not available in your current Hugo version

Hugo levereras i två uppsättningar binärer av tekniska skäl. Den nuvarande webbplatsen körs endast på **Hugo Extended**-versionen. På [releasesidan](https://github.com/gohugoio/hugo/releases) leta efter arkiv med "extended" i namnet. För att bekräfta, kör "hugo version" och leta efter ordet "extended".

### Felsökning av macOS för för många öppna filer

Om du kör 'make serve' på macOS och får följande felmeddelande:

```bash
ERROR 2020/08/01 19:09:18 Error: listen tcp 127.0.0.1:1313: socket: too many open files
make: *** [serve] Error 1
```

Prova att kontrollera den aktuella gränsen för öppna filer:

`launchctl limit maxfiles`

Kör sedan följande kommandon (anpassade från <https://gist.github.com/tombigel/d503800a282fcadbee14b537735d202c>):

```shell
#!/bin/sh

# These are the original gist links, linking to my gists now.
# curl -O https://gist.githubusercontent.com/a2ikm/761c2ab02b7b3935679e55af5d81786a/raw/ab644cb92f216c019a2f032bbf25e258b01d87f9/limit.maxfiles.plist
# curl -O https://gist.githubusercontent.com/a2ikm/761c2ab02b7b3935679e55af5d81786a/raw/ab644cb92f216c019a2f032bbf25e258b01d87f9/limit.maxproc.plist

curl -O https://gist.githubusercontent.com/tombigel/d503800a282fcadbee14b537735d202c/raw/ed73cacf82906fdde59976a0c8248cce8b44f906/limit.maxfiles.plist
curl -O https://gist.githubusercontent.com/tombigel/d503800a282fcadbee14b537735d202c/raw/ed73cacf82906fdde59976a0c8248cce8b44f906/limit.maxproc.plist

sudo mv limit.maxfiles.plist /Library/LaunchDaemons
sudo mv limit.maxproc.plist /Library/LaunchDaemons

sudo chown root:wheel /Library/LaunchDaemons/limit.maxfiles.plist
sudo chown root:wheel /Library/LaunchDaemons/limit.maxproc.plist

sudo launchctl load -w /Library/LaunchDaemons/limit.maxfiles.plist
```

Detta fungerar för Catalina såväl som Mojave macOS.

## Engagera dig i SIG Docs

Läs mer om SIG Docs Kubernetes community och möten på [community-sidan](https://github.com/kubernetes/community/tree/master/sig-docs#meetings).

Du kan också nå underhållarna av detta projekt på:

- [Slack](https://kubernetes.slack.com/messages/sig-docs)
   - [Få en inbjudan till denna Slack](https://slack.k8s.io/)
- [E-postlista](https://groups.google.com/forum/#!forum/kubernetes-sig-docs)

## Bidra med dokumentation

Du kan klicka på knappen **Fork** i det övre högra området på skärmen för att skapa en kopia av detta förråd i ditt GitHub-konto. Denna kopia kallas en _fork_. Gör alla ändringar du vill ha i din fork, och när du är redo att skicka dessa ändringar till oss, gå till din fork och skapa en ny pull-förfrågan för att meddela oss om det.

När din pull-begäran har skapats kommer en Kubernetes-granskare att ta ansvar för att ge tydlig, handlingsbar feedback. Som ägare av pull-begäran **är det ditt ansvar att ändra din pull-begäran för att adressera feedbacken som har getts till dig av Kubernetes granskare.**

Observera också att du kan få mer än en Kubernetes-granskare som ger dig feedback eller att du kan få feedback från en Kubernetes-granskare som är annorlunda än den som ursprungligen tilldelades att ge dig feedback.

Dessutom, i vissa fall kan en av dina granskare be om en teknisk granskning från en Kubernetes teknisk granskare när det behövs. Granskare kommer att göra sitt bästa för att ge feedback i tid, men svarstiden kan variera beroende på omständigheterna.

För mer information om att bidra till Kubernetes-dokumentationen, se:

- [Bidra till Kubernetes docs](https://kubernetes.io/docs/contribute/)
- [Sidinnehållstyper](https://kubernetes.io/docs/contribute/style/page-content-types/)
- [Dokumentationsstilsguide](https://kubernetes.io/docs/contribute/style/style-guide/)
- [Lokalisera Kubernetes-dokumentation](https://kubernetes.io/docs/contribute/localization/)

### Nya bidragsambassadörer

Om du behöver hjälp när som helst när du bidrar är [New Contributor Ambassadors](https://kubernetes.io/docs/contribute/advanced/#serve-as-a-new-contributor-ambassador) en bra kontaktpunkt . Dessa är SIG Docs-godkännare vars ansvar inkluderar mentorskap för nya bidragsgivare och hjälpa dem genom deras första pull-förfrågningar. Det bästa stället att kontakta New Contributors Ambassadors skulle vara på [Kubernetes Slack](https://slack.k8s.io/). Aktuella nya bidragsgivares ambassadörer för SIG Docs:

| Namn                       | Slack                      | GitHub                     |
| -------------------------- | -------------------------- | -------------------------- |
| Arsh Sharma                | @arsh                      | @RinkiyaKeDad              |

## Lokalisering `README.md`s

| Språk                      | Språk                        |
| -------------------------- | ---------------------------- |
| [Kinesiska](README-zh.md)  | [Koreanska](README-ko.md)    |
| [Franska](README-fr.md)    | [Polska](README-pl.md)       |
| [Tyska](README-de.md)      | [Portugisiska](README-pt.md) |
| [Hindi](README-hi.md)      | [Ryska](README-ru.md)        |
| [Indonesiska](README-id.md)| [Spanska](README-es.md)      |
| [Italienska](README-it.md) | [Ukrainska](README-uk.md)    |
| [Japanska](README-ja.md)   | [Vietnamesiska](README-vi.md)|

## Uppförandekod

Deltagande i Kubernetes-gemenskapen styrs av [CNCF:s uppförandekod](https://github.com/cncf/foundation/blob/main/code-of-conduct.md).

## Tack

Kubernetes trivs med samhällsdeltagande, och vi uppskattar dina bidrag till vår webbplats och vår dokumentation!
