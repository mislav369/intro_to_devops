# Intro to DevOps — skripta za LO3, LO4 i LO5

> Priprema za praktični ispit iz kolegija *Intro to DevOps* (Algebra Bernays University).
> Gradivo: **DO188** pogl. 1–8 (Podman) i **DO180** pogl. 1–7 (Kubernetes / OpenShift).
> Pokriva **zadatke s prošlih rokova**, sva praktična pitanja za LO3/LO4/LO5,
> oba RHA comprehensive review laba (Beeper i Famous Quotes) te projektni zadatak (tempconverter).

## Podsjetnik o ispitu

| Stavka | Detalj |
|---|---|
| **Oblik** | Praktični ispit na VM-u u Algebra cloudu — sve se radi unutar te konzole |
| **Pristup** | FortiClient VPN, pa `vcsa7.vua.cloud/ui` (vuastudent / St00dent123, IPsec LocalID `4Students`) |
| **Otvorena knjiga** | docs.podman.io, docs.docker.com, kubernetes.io, minikube.sigs.k8s.io, rha.ole.redhat.com, github.com, get.jenkins.io |
| **Zabranjeno** | Korištenje AI-a |
| **Računi (bez 2FA)** | Red Hat Academy, quay.io, hub.docker.com, Infoeduka |
| **Screenshotovi** | Applications > Utilities > Screenshot, unutar VM-a |
| **Predaja** | results.vua.cloud / exam.net |

## Zadaci s prošlih rokova — razrađeni

| Rok | Zadatak | Gdje |
|---|---|---|
| Kolokvij `Lo3_1` | Drupal + PostgreSQL kroz Compose (zamka: `PGPASSWORD`) | [ROKOVI · LO3](#rokovi-lo3-zadaci-s-proslih-ispita) |
| Kolokvij `Lo3_3` | Vlastiti Alpine + nginx image (zamka: HTTP 404 iz `return 404`) | [ROKOVI · LO3](#rokovi-lo3-zadaci-s-proslih-ispita) |
| Kolokvij `Lo3_2` | **Nije zabilježen** — sadržaj nepoznat | — |
| Lipanj | Drupal + MySQL, komunikacija, trajni volumen s dokazom | [ROKOVI · LO3](#rokovi-lo3-zadaci-s-proslih-ispita) |
| Lipanj | MySQL Deployment + ClusterIP Service + klijentski pod | [ROKOVI · LO4](#rokovi-lo4-zadatak-s-lipanjskog-roka) |
| Lipanj | Pokvarena `podman` naredba + pokvaren Containerfile | [ROKOVI · LO5](#rokovi-lo5-greske-s-lipanjskog-roka) |

## Plan učenja — što koji lab pokriva

| Lab | Izravno | Ekvivalentno / na korak | Ne pokriva |
|---|---|---|---|
| **Beeper** — DO188 pogl. 9, 23 LO3 pitanja | **6** — LO3-2, 3, 4, 7, 13, 20 | **6** — LO3-1, 11, 12, 14, 19, 21 | **11** — compose, secret, kube play |
| **Famous Quotes** — DO180 pogl. 8, 53 LO4 pitanja | **8** — LO4-1, 12, 23, 31, 35, 38, 42, 45 | **15** — OpenShift primitive + probes | **30** — workloadi, ConfigMap, strategije |
| **Comprehensive Review 2** — oba laba, 43 LO5 pitanja | **9** — kvarovi iz samog laba | **14** — scale lab, manifesti, alati | **20** — 18 od toga je podman → Beeper |

**Redoslijed koji najviše vrijedi:**

1. Prvo prođi **zadatke s prošlih rokova** gore — to je jedini poznati uzorak stvarnog ispita.
2. Odradi **Beeper**, pa mu dodaj adminer, bind-mountani `nginx.conf` i backup volumena → **12 od 23** LO3 pitanja.
3. Isti stack prepiši u `compose.yaml` → još 7 pitanja na aplikaciji koju već poznaješ.
4. Odradi **Famous Quotes u zadanih 35 minuta**, pa mu dodaj probes i resource limite → još 5 pitanja.
5. Prepiši taj lab u čisti `kubectl` na minikubeu — ImageStream → digest, Route → Ingress, `oc set env --prefix` → `envFrom.prefix`.
6. Na kraju **compreview-scale** za LO4-3 i LO4-48.

**LO5 se ne može pokriti jednim labom** — podman polovica (LO5-1…18) vježba se kroz Beeper, Kubernetes polovica kroz Famous Quotes, i to tako da lab **prvo namjerno pokvariš**.

Detaljne mape s obrazloženjem za **svako** pitanje:
[LO3 ↔ Beeper](#rha-comprehensive-review-beeper-kompletno-rjesenje) · [LO4 ↔ Famous Quotes](#rha-comprehensive-review-famous-quotes-kompletno-rjesenje) · [LO5 ↔ Comprehensive Review 2](#mapa-koja-lo5-pitanja-pokriva-comprehensive-review-2).

---

## Sadržaj

- **[LO3 — Isporuka aplikacija kontejnerima, mrežna arhitektura i sigurnost komponenti](#lo3-isporuka-aplikacija-kontejnerima-mrezna-arhitektura-i-sigurnost-komponenti)**
    - [0. Temeljni koncepti koje moraš znati napamet](#0-temeljni-koncepti-koje-moras-znati-napamet)
    - [LO3-1 · Pod s app + bazom, komunikacija preko localhosta](#lo3-1-pod-s-app-bazom-komunikacija-preko-localhosta)
    - [LO3-2 · User-defined mreža + DNS po imenu kontejnera](#lo3-2-user-defined-mreza-dns-po-imenu-kontejnera)
    - [LO3-3 · Dvoslojni stack koji NIJE Drupal/Joomla/WordPress](#lo3-3-dvoslojni-stack-koji-nije-drupaljoomlawordpress)
    - [LO3-4 · Trajnost podataka u named volumeu (preživi `podman rm`)](#lo3-4-trajnost-podataka-u-named-volumeu-prezivi-podman-rm)
    - [LO3-5 · `podman secret` umjesto `--env`](#lo3-5-podman-secret-umjesto---env)
    - [LO3-6 · compose.yaml za app + bazu](#lo3-6-composeyaml-za-app-bazu)
    - [LO3-7 · Baza na internoj mreži bez publiciranog porta](#lo3-7-baza-na-internoj-mrezi-bez-publiciranog-porta)
    - [LO3-8 · Healthcheck + `depends_on: condition: service_healthy`](#lo3-8-healthcheck-depends_on-condition-service_healthy)
    - [LO3-9 · `env_file:` umjesto inline vrijednosti](#lo3-9-env_file-umjesto-inline-vrijednosti)
    - [LO3-10 · Skaliranje servisa na više replika](#lo3-10-skaliranje-servisa-na-vise-replika)
    - [LO3-11 · Bind mount za konfiguraciju + named volume za podatke](#lo3-11-bind-mount-za-konfiguraciju-named-volume-za-podatke)
    - [LO3-12 · Backup i restore named volumea](#lo3-12-backup-i-restore-named-volumea)
    - [LO3-13 · nginx reverse proxy ispred aplikacije](#lo3-13-nginx-reverse-proxy-ispred-aplikacije)
    - [LO3-14 · DB admin kontejner (adminer / pgadmin)](#lo3-14-db-admin-kontejner-adminer-pgadmin)
    - [LO3-15 · `podman generate kube` — most prema LO4](#lo3-15-podman-generate-kube-most-prema-lo4)
    - [LO3-16 · `podman kube play` i teardown](#lo3-16-podman-kube-play-i-teardown)
    - [LO3-17 · CPU/memory limiti po servisu u composeu](#lo3-17-cpumemory-limiti-po-servisu-u-composeu)
    - [LO3-18 · `podman pod inspect` — koji namespaceovi se dijele](#lo3-18-podman-pod-inspect-koji-namespaceovi-se-dijele)
    - [LO3-19 · DNS pada kad su kontejneri na različitim mrežama → popravak](#lo3-19-dns-pada-kad-su-kontejneri-na-razlicitim-mrezama-popravak)
    - [LO3-20 · Jedan kontejner na dvije mreže (frontend + backend)](#lo3-20-jedan-kontejner-na-dvije-mreze-frontend-backend)
    - [LO3-21 · `podman compose logs` / `ps` za troubleshooting](#lo3-21-podman-compose-logs-ps-za-troubleshooting)
    - [LO3-22 · Dijeljeni named volume između dvije replike + rizik](#lo3-22-dijeljeni-named-volume-između-dvije-replike-rizik)
    - [LO3-23 · `podman compose down` i sudbina volumena](#lo3-23-podman-compose-down-i-sudbina-volumena)
    - [RHA Comprehensive Review · Beeper — kompletno rješenje](#rha-comprehensive-review-beeper-kompletno-rjesenje)
  - [PROJEKT · tempconverter lokalno s podmanom](#projekt-tempconverter-lokalno-s-podmanom)
    - [Containerfile za Flask aplikaciju](#containerfile-za-flask-aplikaciju)
    - [Baza + spajanje kao NON-ROOT korisnik](#baza-spajanje-kao-non-root-korisnik)
    - [Pokretanje aplikacije](#pokretanje-aplikacije)
    - [Ista stvar preko composea](#ista-stvar-preko-composea)
    - [Unit i integration testovi (1 bod)](#unit-i-integration-testovi-1-bod)
    - [Pipeline: testovi → build → push (1 bod)](#pipeline-testovi-build-push-1-bod)
    - [Usporedba potrošnje resursa: kontejner vs. VM (LO1, 2 boda)](#usporedba-potrosnje-resursa-kontejner-vs-vm-lo1-2-boda)
  - [ROKOVI · LO3 zadaci s prošlih ispita](#rokovi-lo3-zadaci-s-proslih-ispita)
    - [Rok · Lo3_1 — Drupal + PostgreSQL kroz Compose](#rok-lo3_1-drupal-postgresql-kroz-compose)
    - [Rok · Lo3_3 — vlastiti Alpine + nginx image](#rok-lo3_3-vlastiti-alpine-nginx-image)
    - [Rok · lipanj — Drupal + MySQL, komunikacija i trajnost](#rok-lipanj-drupal-mysql-komunikacija-i-trajnost)
    - [Obrazac koji se ponavlja na oba roka](#obrazac-koji-se-ponavlja-na-oba-roka)
    - [LO3 · Šalabahter naredbi](#lo3-salabahter-naredbi)
- **[LO4 — Ubrzana isporuka višeslojnih aplikacija pomoću kontejnera (Kubernetes)](#lo4-ubrzana-isporuka-viseslojnih-aplikacija-pomocu-kontejnera-kubernetes)**
    - [0. Preživljavanje na ispitu: imperativno → YAML](#0-prezivljavanje-na-ispitu-imperativno-yaml)
  - [A · DEPLOYMENTI I SKALIRANJE](#a-deploymenti-i-skaliranje)
    - [LO4-1 · Deployment `web`, nginx:1.25, 3 replike + export YAML](#lo4-1-deployment-web-nginx125-3-replike-export-yaml)
    - [LO4-2 · Autentificirano povlačenje s Docker Huba (pull limit)](#lo4-2-autentificirano-povlacenje-s-docker-huba-pull-limit)
    - [LO4-3 · Skaliranje 3 → 5 na dva načina](#lo4-3-skaliranje-3-5-na-dva-nacina)
    - [LO4-11 · Label selector + odnos Deployment → RS → Pod](#lo4-11-label-selector-odnos-deployment-rs-pod)
  - [B · UPDATEOVI, ROLLBACK I STRATEGIJE](#b-updateovi-rollback-i-strategije)
    - [LO4-4 · Rolling update 1.25 → 1.27](#lo4-4-rolling-update-125-127)
    - [LO4-5 · Povijest rolloutova i rollback + CHANGE-CAUSE](#lo4-5-povijest-rolloutova-i-rollback-change-cause)
    - [LO4-6 · `maxSurge: 1`, `maxUnavailable: 0`](#lo4-6-maxsurge-1-maxunavailable-0)
    - [LO4-7 · `Recreate` strategija](#lo4-7-recreate-strategija)
    - [LO4-9 · `revisionHistoryLimit: 3`](#lo4-9-revisionhistorylimit-3)
    - [LO4-10 · Nepostojeći tag → rollout zapne](#lo4-10-nepostojeci-tag-rollout-zapne)
  - [C · OSTALI WORKLOADI](#c-ostali-workloadi)
    - [LO4-15 · Goli Pod vs. Pod pod Deploymentom](#lo4-15-goli-pod-vs-pod-pod-deploymentom)
    - [LO4-14 · Deployment `httpd:2.4`, 2 replike, imenovani port, `nodeSelector`](#lo4-14-deployment-httpd24-2-replike-imenovani-port-nodeselector)
    - [LO4-13 · Sidecar kontejner](#lo4-13-sidecar-kontejner)
    - [LO4-16 · StatefulSet redis:7, 3 replike + headless Service](#lo4-16-statefulset-redis7-3-replike-headless-service)
    - [LO4-21 · StatefulSet stvara/gasi podove REDOM](#lo4-21-statefulset-stvaragasi-podove-redom)
    - [LO4-27 · Skaliranje StatefulSeta → PVC po replici; što kad se obriše](#lo4-27-skaliranje-statefulseta-pvc-po-replici-sto-kad-se-obrise)
    - [LO4-17 · DaemonSet](#lo4-17-daemonset)
    - [LO4-18 · Job](#lo4-18-job)
    - [LO4-19 · CronJob + suspend + popis Jobova](#lo4-19-cronjob-suspend-popis-jobova)
    - [LO4-20 · Ručno pokretanje Joba iz CronJoba](#lo4-20-rucno-pokretanje-joba-iz-cronjoba)
    - [LO4-8 · Requests i limits](#lo4-8-requests-i-limits)
  - [D · STORAGE, CONFIGMAP I SECRET](#d-storage-configmap-i-secret)
    - [LO4-23 · StorageClass, PV, PVC](#lo4-23-storageclass-pv-pvc)
    - [LO4-22 · Multi-container Pod s dijeljenim `emptyDir`](#lo4-22-multi-container-pod-s-dijeljenim-emptydir)
    - [LO4-30 · `sizeLimit` na `emptyDir`](#lo4-30-sizelimit-na-emptydir)
    - [LO4-28 · initContainer puni dijeljeni volume](#lo4-28-initcontainer-puni-dijeljeni-volume)
    - [LO4-29 · `readOnly: true` na mountu](#lo4-29-readonly-true-na-mountu)
    - [LO4-24 · ConfigMap kao volume — svaki ključ postaje datoteka](#lo4-24-configmap-kao-volume-svaki-kljuc-postaje-datoteka)
    - [LO4-25 · `subPath` — jedan ključ na točnu putanju](#lo4-25-subpath-jedan-kljuc-na-tocnu-putanju)
    - [LO4-31 · Secret iz literala; base64 ≠ enkripcija](#lo4-31-secret-iz-literala-base64-enkripcija)
    - [LO4-32 · Secret iz datoteka (`--from-file`)](#lo4-32-secret-iz-datoteka---from-file)
    - [LO4-33 · Typed TLS Secret](#lo4-33-typed-tls-secret)
    - [LO4-26 / LO4-34 · Secret kao volume + trade-offi](#lo4-26-lo4-34-secret-kao-volume-trade-offi)
    - [LO4-35 · `envFrom` + `secretRef` — svi ključevi odjednom](#lo4-35-envfrom-secretref-svi-kljucevi-odjednom)
    - [LO4-37 · Selektivno montiranje jednog ključa](#lo4-37-selektivno-montiranje-jednog-kljuca)
    - [LO4-36 · Image-pull Secret + `imagePullSecrets`](#lo4-36-image-pull-secret-imagepullsecrets)
  - [E · SERVICES I MREŽA](#e-services-i-mreza)
    - [Pregled tipova Servicea](#pregled-tipova-servicea)
    - [LO4-45 · `port` vs `targetPort` vs `nodePort` — KLJUČNO](#lo4-45-port-vs-targetport-vs-nodeport-kljucno)
    - [LO4-12 · `kubectl expose` — koji objekt nastaje i odakle selektor](#lo4-12-kubectl-expose-koji-objekt-nastaje-i-odakle-selektor)
    - [LO4-38 · ClusterIP + DNS razlučivanje iz drugog poda](#lo4-38-clusterip-dns-razlucivanje-iz-drugog-poda)
    - [LO4-39 · NodePort na minikubeu](#lo4-39-nodeport-na-minikubeu)
    - [LO4-40 · LoadBalancer i `<pending>` na minikubeu](#lo4-40-loadbalancer-i-pending-na-minikubeu)
    - [LO4-41 · Headless Service → per-pod A zapisi](#lo4-41-headless-service-per-pod-a-zapisi)
    - [LO4-42 · Endpoints / EndpointSlice](#lo4-42-endpoints-endpointslice)
    - [LO4-43 · Cross-namespace pristup](#lo4-43-cross-namespace-pristup)
    - [LO4-44 · `port-forward` na Service vs. na Pod](#lo4-44-port-forward-na-service-vs-na-pod)
    - [LO4-46 · Provjera konektivnosti iz debug poda](#lo4-46-provjera-konektivnosti-iz-debug-poda)
    - [LO4-47 · Jedan Service balansira preko DVA Deploymenta](#lo4-47-jedan-service-balansira-preko-dva-deploymenta)
    - [LO4-49 · NodePort + provjera](#lo4-49-nodeport-provjera)
    - [LO4-48 · Sistematska dijagnostika "pod ne doseže Service"](#lo4-48-sistematska-dijagnostika-pod-ne-doseze-service)
  - [F · PROBES (health checks)](#f-probes-health-checks)
    - [LO4-50 · HTTP liveness probe](#lo4-50-http-liveness-probe)
    - [LO4-51 · TCP readiness probe za redis](#lo4-51-tcp-readiness-probe-za-redis)
    - [LO4-52 · Startup probe za ~2-minutni boot](#lo4-52-startup-probe-za-2-minutni-boot)
    - [LO4-53 · Default ponašanje bez ijednog probea](#lo4-53-default-ponasanje-bez-ijednog-probea)
  - [G · OPENSHIFT SPECIFIČNO (`oc`)](#g-openshift-specificno-oc)
    - [Što `oc` dodaje iznad `kubectl`](#sto-oc-dodaje-iznad-kubectl)
    - [ImageStream — i zašto postoji](#imagestream-i-zasto-postoji)
    - [Image trigger — automatski redeploy](#image-trigger-automatski-redeploy)
    - [Route — izlaganje van clustera](#route-izlaganje-van-clustera)
    - [`oc set env` s prefiksom — jedan Secret, dvije aplikacije](#oc-set-env-s-prefiksom-jedan-secret-dvije-aplikacije)
    - [`oc set volumes` — PVC u jednoj naredbi](#oc-set-volumes-pvc-u-jednoj-naredbi)
    - [Ostale korisne `oc set` naredbe](#ostale-korisne-oc-set-naredbe)
    - [RHA Comprehensive Review · Famous Quotes — kompletno rješenje](#rha-comprehensive-review-famous-quotes-kompletno-rjesenje)
  - [H · ORKESTRATORI IZ PROJEKTA (Swarm, anti-affinity, Ingress, Helm)](#h-orkestratori-iz-projekta-swarm-anti-affinity-ingress-helm)
    - [Docker Swarm — jednostavan orkestrator](#docker-swarm-jednostavan-orkestrator)
    - [Anti-affinity — „replike ne smiju biti na istom nodeu"](#anti-affinity-replike-ne-smiju-biti-na-istom-nodeu)
    - [Ingress — izlaganje na portu 80 s load balancingom](#ingress-izlaganje-na-portu-80-s-load-balancingom)
    - [Helm — „template" u smislu projektnog zadatka](#helm-template-u-smislu-projektnog-zadatka)
  - [ROKOVI · LO4 zadatak s lipanjskog roka](#rokovi-lo4-zadatak-s-lipanjskog-roka)
    - [Korak 1 · MySQL Deployment (`mysql:latest`)](#korak-1-mysql-deployment-mysqllatest)
    - [Korak 2 · „Internal connectivity" = ClusterIP Service](#korak-2-internal-connectivity-clusterip-service)
    - [Korak 3 · Klijentski pod se spaja na port koji MySQL nudi](#korak-3-klijentski-pod-se-spaja-na-port-koji-mysql-nudi)
    - [Provjera i dijagnostika ako ne radi](#provjera-i-dijagnostika-ako-ne-radi)
    - [Nadogradnje koje zadatak može tražiti povrh bilješke](#nadogradnje-koje-zadatak-moze-traziti-povrh-biljeske)
    - [LO4 · Šalabahter](#lo4-salabahter)
- **[LO5 — Rješavanje problema u isporuci aplikacija kontejnerima](#lo5-rjesavanje-problema-u-isporuci-aplikacija-kontejnerima)**
    - [Univerzalni redoslijed dijagnostike](#univerzalni-redoslijed-dijagnostike)
  - [A · GREŠKE U `podman run` NAREDBAMA](#a-greske-u-podman-run-naredbama)
    - [LO5-1 · Zamijenjeni portovi](#lo5-1-zamijenjeni-portovi)
    - [LO5-2 · `-e` bez vrijednosti](#lo5-2--e-bez-vrijednosti)
    - [LO5-3 · `--network host` + `-p`](#lo5-3---network-host--p)
    - [LO5-4 · Kontejner odmah `Exited (0)`](#lo5-4-kontejner-odmah-exited-0)
    - [LO5-5 · `--rm` + `-d` gotcha](#lo5-5---rm--d-gotcha)
    - [LO5-6 · Premalen memory limit](#lo5-6-premalen-memory-limit)
    - [LO5-7 · Nema DNS-a na default mreži](#lo5-7-nema-dns-a-na-default-mrezi)
    - [LO5-8 · Bind mount bez SELinux oznake](#lo5-8-bind-mount-bez-selinux-oznake)
  - [B · GREŠKE U CONTAINERFILE-OVIMA](#b-greske-u-containerfile-ovima)
    - [LO5-10 · `apt-get install` bez `update`](#lo5-10-apt-get-install-bez-update)
    - [LO5-11 · Shell forma vs. exec forma](#lo5-11-shell-forma-vs-exec-forma)
    - [LO5-12 · Loš redoslijed za layer caching](#lo5-12-los-redoslijed-za-layer-caching)
    - [LO5-13 · Prepisan `PATH`](#lo5-13-prepisan-path)
    - [LO5-14 / LO5-15 · `EXPOSE` ≠ objavljivanje + neusklađen port](#lo5-14-lo5-15-expose-objavljivanje-neusklađen-port)
    - [LO5-16 · Ogroman image nakon `build-essential`](#lo5-16-ogroman-image-nakon-build-essential)
    - [LO5-17 · `USER` prije `COPY`/`RUN`](#lo5-17-user-prije-copyrun)
    - [LO5-18 · Golang image ~1 GB → multi-stage](#lo5-18-golang-image-1-gb-multi-stage)
  - [C · KUBERNETES TROUBLESHOOTING](#c-kubernetes-troubleshooting)
    - [Mapa simptoma → uzrok](#mapa-simptoma-uzrok)
    - [LO5-19 · Pod zaglavljen u `Pending`](#lo5-19-pod-zaglavljen-u-pending)
    - [LO5-20 · Pokvarena indentacija u manifestu](#lo5-20-pokvarena-indentacija-u-manifestu)
    - [LO5-21 · `ImagePullBackOff`](#lo5-21-imagepullbackoff)
    - [LO5-22 · `CrashLoopBackOff`](#lo5-22-crashloopbackoff)
    - [LO5-23 · `OOMKilled`](#lo5-23-oomkilled)
    - [LO5-24 · Podovi nikad ne postanu `Ready`](#lo5-24-podovi-nikad-ne-postanu-ready)
    - [LO5-25 · Service ne vraća ništa → label mismatch](#lo5-25-service-ne-vraca-nista-label-mismatch)
    - [LO5-26 · `selector.matchLabels` ≠ `template.metadata.labels`](#lo5-26-selectormatchlabels-templatemetadatalabels)
    - [LO5-27 · `requests` veći od kapaciteta nodea](#lo5-27-requests-veci-od-kapaciteta-nodea)
    - [LO5-28 · Pod montira nepostojeći PVC](#lo5-28-pod-montira-nepostojeci-pvc)
    - [LO5-29 · `ubuntu` bez long-running naredbe](#lo5-29-ubuntu-bez-long-running-naredbe)
    - [LO5-30 · Triaža preko eventa](#lo5-30-triaza-preko-eventa)
    - [LO5-31 · Debug DNS-a iznutra](#lo5-31-debug-dns-a-iznutra)
    - [LO5-32 · Ephemeral debug kontejner (distroless / bez shella)](#lo5-32-ephemeral-debug-kontejner-distroless-bez-shella)
    - [LO5-33 · Jednokratni pod za test konektivnosti](#lo5-33-jednokratni-pod-za-test-konektivnosti)
    - [LO5-34 · Node `NotReady`](#lo5-34-node-notready)
    - [LO5-35 · Pod zaglavljen u `Terminating`](#lo5-35-pod-zaglavljen-u-terminating)
    - [LO5-36 · Kriv par `apiVersion`/`kind`](#lo5-36-kriv-par-apiversionkind)
    - [LO5-37 · `CreateContainerConfigError` — nedostaje ključ ConfigMapa](#lo5-37-createcontainerconfigerror-nedostaje-kljuc-configmapa)
    - [LO5-38 · Secret iz drugog namespacea](#lo5-38-secret-iz-drugog-namespacea)
    - [LO5-39 · `ProgressDeadlineExceeded`](#lo5-39-progressdeadlineexceeded)
    - [LO5-40 · Uhvati typo PRIJE deploya](#lo5-40-uhvati-typo-prije-deploya)
    - [LO5-41 · "Aplikacija ne može do baze" — provjera po redu](#lo5-41-aplikacija-ne-moze-do-baze-provjera-po-redu)
    - [LO5-42 · `restartCount` i `lastState` preko jsonpatha](#lo5-42-restartcount-i-laststate-preko-jsonpatha)
    - [LO5-43 · OpenShift **Route** vs Kubernetes **Ingress**](#lo5-43-openshift-route-vs-kubernetes-ingress)
  - [D · COMPREHENSIVE REVIEW: TROUBLESHOOT AND SCALE](#d-comprehensive-review-troubleshoot-and-scale)
    - [Rutina od šest koraka za „aplikacija ne radi, popravi je"](#rutina-od-sest-koraka-za-aplikacija-ne-radi-popravi-je)
    - [Tipične podmetnute greške u tom labu](#tipicne-podmetnute-greske-u-tom-labu)
    - [Skaliranje — što se traži](#skaliranje-sto-se-trazi)
    - [Popravak probea `oc`-om, bez ručnog editiranja YAML-a](#popravak-probea-oc-om-bez-rucnog-editiranja-yaml-a)
    - [Zadnji korak koji studenti zaboravljaju](#zadnji-korak-koji-studenti-zaboravljaju)
    - [Mapa: koja LO5 pitanja pokriva Comprehensive Review 2](#mapa-koja-lo5-pitanja-pokriva-comprehensive-review-2)
  - [E · PROJEKT · troubleshooting dnevnik](#e-projekt-troubleshooting-dnevnik)
    - [Format zapisa koji nosi bod](#format-zapisa-koji-nosi-bod)
    - [Problemi koji se na ovom projektu stvarno pojave](#problemi-koji-se-na-ovom-projektu-stvarno-pojave)
    - [Struktura poglavlja u projektnoj dokumentaciji](#struktura-poglavlja-u-projektnoj-dokumentaciji)
    - [Video (2 boda)](#video-2-boda)
  - [ROKOVI · LO5 greške s lipanjskog roka](#rokovi-lo5-greske-s-lipanjskog-roka)
    - [Rok 1 · Pokvarena `podman` naredba](#rok-1-pokvarena-podman-naredba)
    - [Rok 2 · Pokvaren Containerfile](#rok-2-pokvaren-containerfile)
    - [Rok 3 · „Duži popis naredbi"](#rok-3-duzi-popis-naredbi)
    - [Što ovo znači za pripremu](#sto-ovo-znaci-za-pripremu)
    - [LO5 · Šalabahter dijagnostike](#lo5-salabahter-dijagnostike)

---

# LO3 — Isporuka aplikacija kontejnerima, mrežna arhitektura i sigurnost komponenti

> **Što se ocjenjuje:** znaš li povezati više kontejnera u funkcionalnu aplikaciju (app + baza), izolirati ih mrežno, trajno spremiti podatke i sigurno proslijediti tajne. Alat je **podman** (pod, network, volume, secret) i **podman compose**.

---

### 0. Temeljni koncepti koje moraš znati napamet

#### Pod vs. mreža — dva načina da kontejneri komuniciraju

| | **Pod** (`podman pod create`) | **User-defined network** (`podman network create`) |
|---|---|---|
| Dijele | network namespace (isti IP, isti localhost), IPC, UTS | ništa — svaki ima svoj IP |
| Komuniciraju preko | `localhost:<port>` | **imena kontejnera** (ugrađeni DNS) |
| Portovi | publiciraju se **na podu**, ne na kontejneru | publiciraju se po kontejneru |
| Sukob portova | DA — dva kontejnera ne mogu slušati isti port | NE |
| PID namespace | **NE dijele** po defaultu (samo infra pauza) | ne dijele |
| Analogija | Kubernetes Pod | Docker/Compose mreža |

#### Default mreža vs. user-defined mreža — NAJVAŽNIJE PRAVILO

- **`podman` default bridge mreža → NEMA DNS-a.** Kontejneri se **ne mogu** naći po imenu.
- **User-defined mreža (`podman network create`) → IMA DNS (aardvark-dns).** Kontejneri se nađu po `--name`.
- Compose **automatski** stvara user-defined mrežu → zato u composeu DNS uvijek radi.

Ovo je izvor barem 3 pitanja na ispitu (LO3-2, LO3-19, LO5-7).

#### Volume vs. bind mount

| | **Named volume** (`-v mojvol:/path`) | **Bind mount** (`-v ./dir:/path`) |
|---|---|---|
| Gdje živi | podman ga upravlja (`~/.local/share/containers/storage/volumes/`) | bilo gdje na hostu, ti biraš |
| Za što | **podaci** (baze, uploadi) | **konfiguracija**, izvorni kod u razvoju |
| Prijenosivost | da, `podman volume` naredbe | ovisi o putanji hosta |
| SELinux | ne treba oznaka | **treba `:Z` ili `:z`** |
| Backup | preko helper kontejnera | obični `tar` na hostu |

`:Z` = privatna SELinux oznaka (samo taj kontejner). `:z` = dijeljena oznaka (više kontejnera). Bez toga na RHEL-u → **Permission denied**.

---

### LO3-1 · Pod s app + bazom, komunikacija preko localhosta

```bash
# 1. Stvori pod i publiciraj port NA PODU
podman pod create --name mypod -p 8080:80

# 2. Baza ide u pod (ne treba -p, port je već "unutra")
podman run -d --pod mypod --name db \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_USER=app \
  -e POSTGRES_DB=appdb \
  docker.io/library/postgres:16

# 3. Aplikacija ide u isti pod
podman run -d --pod mypod --name web docker.io/library/nginx

# 4. DOKAZ da se vide preko localhosta
podman exec db bash -c "apt-get update -qq && apt-get install -y -qq iputils-ping netcat-openbsd"
podman exec web curl -s http://localhost:80        # nginx sam sebe vidi
podman exec web bash -c "cat < /dev/tcp/localhost/5432"   # app vidi bazu na localhost:5432
```

**Objašnjenje za ispit:** Kontejneri u istom podu dijele *network namespace* koji drži skriveni **infra kontejner** (`k8s.gaus.pause`). Zato imaju isti IP i vide se na `127.0.0.1`. Posljedica: **portovi se ne smiju preklapati** unutar poda, i `-p` se mora zadati kod `podman pod create`, ne kod `podman run --pod`.

```bash
podman pod ps
podman ps -a --pod          # vidi se infra kontejner
```

---

### LO3-2 · User-defined mreža + DNS po imenu kontejnera

```bash
podman network create appnet

podman run -d --network appnet --name postgres-db \
  -e POSTGRES_PASSWORD=secret -e POSTGRES_USER=app -e POSTGRES_DB=appdb \
  docker.io/library/postgres:16

podman run -d --network appnet --name app -p 8080:80 docker.io/library/nginx

# DOKAZ razlučivanja imena
podman exec app getent hosts postgres-db
podman exec -it app bash -c "apt update && apt install -y iputils-ping && ping -c2 postgres-db"
```

**Objašnjenje:** Na user-defined mreži podman pokreće **aardvark-dns** (backend: netavark). Svaki kontejner na toj mreži dobije DNS zapis za svoj `--name` i sve `--network-alias` vrijednosti. Na **default** mreži toga nema → treba `--add-host` ili IP.

```bash
podman network inspect appnet     # subnet, gateway, popis kontejnera
podman network ls
```

---

### LO3-3 · Dvoslojni stack koji NIJE Drupal/Joomla/WordPress

Uzmi **Gitea + PostgreSQL** (najlakši first-run) ili **Ghost + MySQL**.

#### Gitea + PostgreSQL

```bash
podman network create giteanet
podman volume create gitea-db
podman volume create gitea-data

podman run -d --name gitea-db --network giteanet \
  -e POSTGRES_USER=gitea -e POSTGRES_PASSWORD=gitea -e POSTGRES_DB=gitea \
  -v gitea-db:/var/lib/postgresql/data \
  docker.io/library/postgres:16

podman run -d --name gitea --network giteanet -p 3000:3000 \
  -e GITEA__database__DB_TYPE=postgres \
  -e GITEA__database__HOST=gitea-db:5432 \
  -e GITEA__database__NAME=gitea \
  -e GITEA__database__USER=gitea \
  -e GITEA__database__PASSWD=gitea \
  -v gitea-data:/data \
  docker.io/gitea/gitea:latest
```

Otvori `http://localhost:3000` → install stranica → Database Type **PostgreSQL**, Host `gitea-db:5432`, User `gitea`, Password `gitea`, Database Name `gitea` → **Install Gitea** → registriraj admin korisnika. **Screenshot gotove instalacije = bod.**

#### Ghost + MySQL (alternativa)

```bash
podman network create ghostnet
podman run -d --name ghost-db --network ghostnet \
  -e MYSQL_ROOT_PASSWORD=rootpw -e MYSQL_DATABASE=ghost \
  -v ghost-db:/var/lib/mysql docker.io/library/mysql:8

podman run -d --name ghost --network ghostnet -p 2368:2368 \
  -e database__client=mysql \
  -e database__connection__host=ghost-db \
  -e database__connection__user=root \
  -e database__connection__password=rootpw \
  -e database__connection__database=ghost \
  -e NODE_ENV=development \
  -v ghost-content:/var/lib/ghost/content \
  docker.io/library/ghost:5
```

---

### LO3-4 · Trajnost podataka u named volumeu (preživi `podman rm`)

```bash
podman volume create pgdata

podman run -d --name db --network appnet \
  -e POSTGRES_PASSWORD=secret -e POSTGRES_USER=app -e POSTGRES_DB=appdb \
  -v pgdata:/var/lib/postgresql/data \
  docker.io/library/postgres:16

# upiši podatak
podman exec -it db psql -U app -d appdb -c "CREATE TABLE t(id int); INSERT INTO t VALUES (42);"

# UNIŠTI kontejner
podman rm -f db

# ponovno stvori s ISTIM volumeom
podman run -d --name db --network appnet \
  -e POSTGRES_PASSWORD=secret -e POSTGRES_USER=app -e POSTGRES_DB=appdb \
  -v pgdata:/var/lib/postgresql/data docker.io/library/postgres:16

# DOKAZ
podman exec -it db psql -U app -d appdb -c "SELECT * FROM t;"   # → 42
```

**Ključna rečenica:** Container layer (writable layer) je **efemeran** i briše se s kontejnerom. Volume je zaseban objekt sa svojim životnim ciklusom (`podman volume ls/inspect/rm`) i preživljava.

---

### LO3-5 · `podman secret` umjesto `--env`

```bash
printf 'SuperTajna123' | podman secret create db_password -
podman secret ls
podman secret inspect db_password        # NE prikazuje vrijednost

# Način A: secret kao datoteka (default) → /run/secrets/db_password
podman run -d --name db --secret db_password \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  docker.io/library/postgres:16

# Način B: secret kao env varijabla
podman run -d --name db2 --secret db_password,type=env,target=POSTGRES_PASSWORD \
  docker.io/library/postgres:16
```

**Sigurnosna prednost (napiši ovo):**
1. `--env` vrijednost je **trajno vidljiva** u `podman inspect`, u `/proc/<pid>/environ`, u `podman ps --format`, u shell historyju i u image metapodacima ako se commita.
2. Secret se montira u **tmpfs** (`/run/secrets/…`) → nikad ne dotakne disk kontejnera, ne završi u image layeru.
3. Secret je zaseban objekt s vlastitim RBAC/životnim ciklusom, može se rotirati bez mijenjanja naredbe pokretanja.

---

### LO3-6 · compose.yaml za app + bazu

```yaml
# compose.yaml
services:
  db:
    image: docker.io/library/postgres:16
    container_name: postgres-srv
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - dbdata:/var/lib/postgresql/data
    networks:
      - backend

  app:
    image: docker.io/gitea/gitea:latest
    container_name: gitea-srv
    ports:
      - "3000:3000"
    environment:
      GITEA__database__DB_TYPE: postgres
      GITEA__database__HOST: db:5432
      GITEA__database__NAME: appdb
      GITEA__database__USER: app
      GITEA__database__PASSWD: secret
    depends_on:
      - db
    networks:
      - backend
      - frontend

volumes:
  dbdata:

networks:
  frontend:
  backend:
```

```bash
podman compose up -d          # ili: podman-compose up -d
podman compose ps
podman ps
```

**Napomena za ispit:** `podman compose` je wrapper koji poziva `docker-compose` ili `podman-compose`. Ako `podman compose` ne radi, probaj `podman-compose`. Ime servisa (`db`) je ujedno **DNS ime** na compose mreži.

---

### LO3-7 · Baza na internoj mreži bez publiciranog porta

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    networks:
      - backend          # SAMO backend, i NEMA ports:

  app:
    image: docker.io/library/nginx
    ports:
      - "8080:80"        # samo app je izložen hostu
    networks:
      - backend
      - frontend

networks:
  frontend:
  backend:
    internal: true       # ← nema ni izlaza prema internetu
```

**Dokaz da baza nije dostupna s hosta:**
```bash
ss -ltnp | grep 5432          # ništa
nc -zv localhost 5432         # Connection refused
podman port db                # prazno
# ali iz app kontejnera radi:
podman compose exec app bash -c "cat < /dev/tcp/db/5432"
```

**Objašnjenje:** Bez `ports:` nema port-forwardinga (nema `rootlessport`/iptables DNAT pravila), pa je baza dostupna **samo** unutar mreže kontejnera. Ovo je *defense in depth* — smanjuje napadnu površinu. `internal: true` dodatno onemogućuje odlazni promet prema van.

---

### LO3-8 · Healthcheck + `depends_on: condition: service_healthy`

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s

  app:
    image: docker.io/library/nginx
    ports: ["8080:80"]
    depends_on:
      db:
        condition: service_healthy
```

```bash
podman compose up -d
podman ps --format "table {{.Names}}\t{{.Status}}"   # vidi se (healthy)
podman healthcheck run <db-container>
podman inspect --format '{{ .State.Health.Status }}' <db-container>
```

**Objašnjenje:** Obični `depends_on: [db]` čeka samo da se kontejner **pokrene**, ne da bude **spreman**. Baza treba 5–20 s da inicijalizira cluster; app u međuvremenu padne s "connection refused". `condition: service_healthy` blokira start appa dok healthcheck ne prijavi `healthy`. Alternativa bez healthchecka: retry logika u aplikaciji ili `wait-for-it.sh` init skripta.

---

### LO3-9 · `env_file:` umjesto inline vrijednosti

```bash
cat > db.env <<'EOF'
POSTGRES_USER=app
POSTGRES_PASSWORD=secret
POSTGRES_DB=appdb
EOF
echo "db.env" >> .gitignore
```

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    env_file:
      - ./db.env
```

**Zašto:** compose.yaml ide u git, `.env` datoteke ne. Odvaja **konfiguraciju od koda** (12-factor app), omogućuje različite vrijednosti po okolini (dev/stage/prod) bez mijenjanja manifesta.

Razlika: `env_file` = varijable **unutar kontejnera**. `.env` datoteka u istom direktoriju = varijable za **supstituciju `${VAR}` u samom compose.yaml**.

---

### LO3-10 · Skaliranje servisa na više replika

```bash
podman compose up -d --scale app=3
podman compose ps            # app-1, app-2, app-3
```

**Bitno:** servis koji skaliraš **ne smije imati fiksni `ports: "8080:80"`** (sukob portova na hostu) i ne smije imati `container_name:`. Rješenja:
- izbaci `ports:` i stavi **nginx reverse proxy** ispred (LO3-13), ili
- koristi raspon: `ports: ["8080-8082:80"]`.

**Kako se raspoređuju zahtjevi:** compose sam **ne balansira**. Distribucija ide preko DNS round-robina — ime servisa `app` razriješi se u **više A zapisa** (po jedan po replici), a klijent uzima jedan. Za pravi load balancing treba proxy (nginx/HAProxy) ili orkestrator (k8s Service).

---

### LO3-11 · Bind mount za konfiguraciju + named volume za podatke

```bash
mkdir -p ./conf
cat > ./conf/nginx.conf <<'EOF'
server { listen 80; root /usr/share/nginx/html; }
EOF

podman run -d --name web \
  -v ./conf/nginx.conf:/etc/nginx/conf.d/default.conf:ro,Z \
  -v webdata:/usr/share/nginx/html \
  -p 8080:80 docker.io/library/nginx
```

**Zašto tako:**
- **Konfiguracija → bind mount**: čitaš/uređuješ je izravno na hostu, verzionira se u gitu, montiraš `:ro` jer je aplikacija ne smije mijenjati.
- **Podaci → named volume**: podman ih upravlja, ne ovise o putanji hosta, prenosivo, lako backup/restore, ispravne dozvole i SELinux oznake automatski.

---

### LO3-12 · Backup i restore named volumea

```bash
# BACKUP — helper kontejner montira volume + host dir
podman run --rm \
  -v pgdata:/data:ro,Z \
  -v ./backup:/backup:Z \
  docker.io/library/alpine \
  tar czf /backup/pgdata.tar.gz -C /data .

ls -lh ./backup/pgdata.tar.gz

# RESTORE u SVJEŽ volume
podman volume create pgdata-restored
podman run --rm \
  -v pgdata-restored:/data:Z \
  -v ./backup:/backup:ro,Z \
  docker.io/library/alpine \
  tar xzf /backup/pgdata.tar.gz -C /data

podman run --rm -v pgdata-restored:/data alpine ls -la /data
```

**Alternativa (jednostavnija, podman-native):**
```bash
podman volume export pgdata -o pgdata.tar
podman volume import pgdata-restored pgdata.tar
```

**Zašto helper kontejner:** rootless podman drži volumene u `~/.local/share/containers/…` s user-namespace UID mapiranjem — ne možeš ih pouzdano čitati izravno s hosta. Helper kontejner vidi ispravne UID-ove. **Bazu prije backupa zaustavi** (ili koristi `pg_dump`) da izbjegneš nekonzistentan snapshot.

---

### LO3-13 · nginx reverse proxy ispred aplikacije

```bash
podman network create proxynet

podman run -d --name app --network proxynet docker.io/library/httpd   # BEZ -p!

cat > proxy.conf <<'EOF'
server {
    listen 80;
    location / {
        proxy_pass http://app:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
EOF

podman run -d --name proxy --network proxynet -p 8080:80 \
  -v ./proxy.conf:/etc/nginx/conf.d/default.conf:ro,Z \
  docker.io/library/nginx

curl http://localhost:8080     # ide kroz proxy do appa
```

**Objašnjenje:** Samo proxy je izložen hostu; app je dostupan isključivo unutar `proxynet`. Proxy je jedina ulazna točka → tu radiš TLS terminaciju, rate limiting, load balancing (`upstream` blok s više backendova), i skrivaš internu topologiju. Ovo je container-svijet ekvivalent Kubernetes **Ingressa**.

Verzija s upstream load balancingom:
```nginx
upstream backend { server app1:80; server app2:80; }
server { listen 80; location / { proxy_pass http://backend; } }
```

---

### LO3-14 · DB admin kontejner (adminer / pgadmin)

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    networks: [backend]

  adminer:
    image: docker.io/library/adminer
    ports: ["8081:8080"]
    networks: [backend]

networks:
  backend:
```

Otvori `http://localhost:8081` → System **PostgreSQL**, Server **db**, User **app**, Password **secret**, Database **appdb**.

**Za ispit napiši:** adminer je na istoj mreži kao baza pa je doseže po DNS imenu `db`; baza i dalje nema publiciran port prema hostu. Ovo je *bastion* obrazac — pristup bazi samo kroz kontrolirani kontejner. U produkciji admin sučelje ne bi bilo javno izloženo (ili bi bilo iza autentikacije/VPN-a).

---

### LO3-15 · `podman generate kube` — most prema LO4

```bash
podman pod create --name mypod -p 8080:80
podman run -d --pod mypod --name web docker.io/library/nginx
podman run -d --pod mypod --name db -e POSTGRES_PASSWORD=secret docker.io/library/postgres:16

podman generate kube mypod > mypod.yaml
# noviji podman: podman kube generate mypod > mypod.yaml
cat mypod.yaml
```

Izlaz je `apiVersion: v1, kind: Pod` s `spec.containers[]`, portovima i volumenima.

**Kako spaja LO3 i LO4:** podman pod je *namjerno* modeliran po Kubernetes Podu (isti network namespace, infra/pause kontejner). Zato se lokalno razvijeni stack može eksportirati u Kubernetes manifest i deployati na cluster. **Ograničenja:** generira `kind: Pod`, ne Deployment → nema replika, self-healinga ni rolling updatea; treba ručno omotati u Deployment. Ne generira Services, PVC-ove se generira kao `hostPath`/`persistentVolumeClaim` ovisno o verziji.

---

### LO3-16 · `podman kube play` i teardown

```bash
podman kube play mypod.yaml
podman pod ps
podman ps

podman kube play --down mypod.yaml     # ugasi i obriši pod+kontejnere
```

- `--down` uklanja podove i kontejnere, **ali ostavlja volumene**.
- `podman kube play --replace file.yaml` — zamijeni postojeći.
- Podržava `kind: Pod`, `Deployment`, `DaemonSet`, `Job`, `ConfigMap`, `PersistentVolumeClaim`, `Secret`.

**Poanta:** možeš testirati Kubernetes manifest lokalno bez clustera.

---

### LO3-17 · CPU/memory limiti po servisu u composeu

```yaml
services:
  app:
    image: docker.io/library/nginx
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 256M
        reservations:
          cpus: "0.25"
          memory: 128M
```

Ako `deploy:` ignorira tvoja compose implementacija, koristi kratki oblik:
```yaml
    cpus: 0.5
    mem_limit: 256m
```

```bash
podman stats --no-stream
podman inspect <c> --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}'
```

**Objašnjenje:** limiti se provode kroz **cgroups v2** (`memory.max`, `cpu.max`). Prekoračenje memorije → kernel OOM killer ubije proces (**OOMKilled**, exit 137). Prekoračenje CPU-a → throttling, ne ubijanje. Rezervacije su *soft* jamstvo pri rasporedbi.

---

### LO3-18 · `podman pod inspect` — koji namespaceovi se dijele

```bash
podman pod inspect mypod
podman pod inspect mypod --format '{{.SharedNamespaces}}'
```

Tipično: **`net`, `ipc`, `uts`** se dijele. **`pid` i `mnt` se NE dijele** po defaultu.

```bash
# dokaz da PID nije dijeljen
podman exec web ps aux      # vidi samo svoje procese
# uključi dijeljenje PID-a:
podman pod create --name p2 --share net,ipc,uts,pid
```

| Namespace | Dijeli se? | Posljedica |
|---|---|---|
| `net` | ✅ | isti IP, localhost komunikacija, isti portovi |
| `ipc` | ✅ | dijeljena memorija, semafori |
| `uts` | ✅ | isti hostname |
| `pid` | ❌ (opcionalno) | ne vide procese jedan drugom |
| `mnt` | ❌ | svaki ima svoj filesystem |
| `user` | ❌ | zasebno UID mapiranje |

---

### LO3-19 · DNS pada kad su kontejneri na različitim mrežama → popravak

```bash
podman network create net-a
podman network create net-b
podman run -d --name c1 --network net-a docker.io/library/alpine sleep 1d
podman run -d --name c2 --network net-b docker.io/library/alpine sleep 1d

podman exec c1 ping -c1 c2      # ✗ bad address 'c2'
```

**Popravak — spoji ih na zajedničku mrežu:**
```bash
podman network connect net-a c2
podman exec c1 ping -c1 c2      # ✓ radi
```

**Objašnjenje:** aardvark-dns drži **zasebnu DNS zonu po mreži**. Kontejner vidi samo imena kontejnera koji dijele barem jednu mrežu s njim. Ovo je namjerno — **mrežna segmentacija** je sigurnosna značajka, ne bug.

Isto se događa i na **default** mreži, ali iz drugog razloga: default bridge uopće nema DNS (vidi LO5-7).

---

### LO3-20 · Jedan kontejner na dvije mreže (frontend + backend)

```bash
podman network create frontend
podman network create backend

podman run -d --name db --network backend \
  -e POSTGRES_PASSWORD=secret docker.io/library/postgres:16

podman run -d --name app --network frontend -p 8080:80 docker.io/library/nginx
podman network connect backend app       # app je sad na obje

podman inspect app --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
```

**Prednost segmentacije:** app je jedini most između zona. Baza je na `backend` mreži i **fizički nedostupna** s frontenda ili hosta — čak i ako netko kompromitira proxy na frontendu, ne može doseći bazu direktno. To je *least privilege* na mrežnoj razini i smanjuje **lateral movement**. Ekvivalent u Kubernetesu: **NetworkPolicy**.

---

### LO3-21 · `podman compose logs` / `ps` za troubleshooting

```bash
podman compose ps                 # status svih servisa + portovi
podman compose ps -a              # uključi i zaustavljene
podman compose logs               # svi logovi, prefiksirani imenom servisa
podman compose logs -f app        # prati samo app
podman compose logs --tail=50 db
podman compose top
podman compose exec app sh        # shell u servis
podman compose config             # renderirani, validirani compose (rješava ${VARS})
```

**Metodologija:** `ps` → koji servis nije `Up`? → `logs <taj servis>` → zadnji error prije izlaska → `exec` u susjedni servis pa testiraj konektivnost. `podman compose config` je prva provjera kad sumnjaš na YAML/varijable.

---

### LO3-22 · Dijeljeni named volume između dvije replike + rizik

```yaml
services:
  app:
    image: docker.io/library/nginx
    volumes:
      - shared-config:/etc/app/config
volumes:
  shared-config:
```

```bash
podman compose up -d --scale app=2
podman compose exec app sh -c 'echo "hello" > /etc/app/config/test.txt'
# druga replika vidi istu datoteku
```

**Rizik dijeljenog read-write storagea (obavezno napiši):**
1. **Race conditions / korupcija podataka** — dvije replike pišu istovremeno bez zaključavanja; nema koordinacije.
2. **Nema distribuiranog file lockinga** preko mrežnih FS-ova (NFS advisory lockovi su nepouzdani).
3. **Baze podataka NIKAD ne smiju dijeliti data direktorij** — MySQL/Postgres pretpostavljaju ekskluzivan pristup; rezultat je trenutna korupcija.
4. **Postaje bottleneck i single point of failure** — ubija horizontalnu skalabilnost.

**Sigurno:** dijeljeni volume **read-only** (`:ro`) za konfiguraciju/statiku, a stanje drži u bazi ili object storageu. U Kubernetesu: `ReadWriteMany` PVC samo kad aplikacija to eksplicitno podržava; inače `ReadWriteOnce` + StatefulSet s PVC po replici.

---

### LO3-23 · `podman compose down` i sudbina volumena

```bash
podman compose down              # briše kontejnere + mreže. VOLUMENI OSTAJU.
podman volume ls                 # named volumeni su još tu → podaci sačuvani

podman compose down --volumes    # briše i named volumene → PODACI NESTAJU
# ili: podman compose down -v
podman compose down --rmi all    # obriši i imagee
```

**Objašnjenje:** Default je **siguran** — pretpostavka je da su podaci u volumenima vrijedni i da `down` radiš rutinski (npr. prije `up` s novom verzijom). `--volumes` je destruktivan i briše **samo named volumene deklarirane u compose datoteci**; vanjski volumeni (`external: true`) i bind mountovi se **ne diraju**.

---

### RHA Comprehensive Review · Beeper — kompletno rješenje

> **Ovo je najvažniji zadatak za LO3.** Tvoja bilješka s lipanjskog roka kaže doslovno: *„To prepare for the exam complete Chapter 9: Comprehensive Review of the DO188 course."* Zadatak spaja **sve** iz LO3: dvije mreže s DNS-om, bazu s volumenom, dva multi-stage builda i kontejner koji stoji na dvije mreže odjednom.

**Aplikacija Beeper** = React UI (nginx) → Spring Boot API → PostgreSQL baza.

```bash
lab start comprehensive-review
# datoteke: /home/student/DO188/labs/comprehensive-review/{beeper-backend,beeper-ui}
```

#### Korak 1 · Dvije mreže s DNS-om

```bash
podman network create beeper-backend
podman network create beeper-frontend
podman network ls
podman network inspect beeper-backend | grep -i dns
```
User-defined mreže **imaju DNS uključen po defaultu** (aardvark-dns) — zato spec traži baš njih, a ne default mrežu. Ako te pitaju eksplicitno: `--disable-dns` bi ga isključio.

#### Korak 2 · PostgreSQL s named volumeom

```bash
podman volume create beeper-data

podman run -d --name beeper-db \
  --network beeper-backend \
  -v beeper-data:/var/lib/pgsql/data \
  -e POSTGRESQL_USER=beeper \
  -e POSTGRESQL_PASSWORD=beeper123 \
  -e POSTGRESQL_DATABASE=beeper \
  registry.ocp4.example.com:8443/rhel9/postgresql-13:1

podman logs beeper-db          # čekaj "database system is ready to accept connections"
podman volume inspect beeper-data
```
**Pazi:** Red Hat postgres image koristi prefiks **`POSTGRESQL_`** i putanju **`/var/lib/pgsql/data`**. Docker Hub postgres image koristi `POSTGRES_` i `/var/lib/postgresql/data`. Zamjena ta dva je klasična greška — baza se digne, ali s praznom konfiguracijom.
**Nema `-p`** — baza je dostupna samo unutar `beeper-backend`.

#### Korak 3 · Multi-stage Containerfile za API (Java/Maven)

```dockerfile
# ---------- STAGE 1: build ----------
FROM registry.ocp4.example.com:8443/ubi8/openjdk-17:1.12 AS builder
# ovaj image ima WORKDIR /home/jboss
COPY --chown=jboss:root . .
RUN mvn -s settings.xml package

# ---------- STAGE 2: runtime ----------
FROM registry.ocp4.example.com:8443/ubi8/openjdk-17-runtime:1.12
COPY --from=builder /home/jboss/target/beeper-1.0.0.jar .
CMD ["java", "-jar", "beeper-1.0.0.jar"]
```

```bash
cd /home/student/DO188/labs/comprehensive-review/beeper-backend
podman build -t beeper-api:v1 .
podman images | grep beeper-api
```

**Objašnjenja koja nose bodove:**
- `COPY --chown=jboss:root . .` — UBI imageovi rade kao **non-root** korisnik (`jboss`, UID 185). Bez `--chown` datoteke su vlasništvo roota i Maven ne može pisati u `target/` → build padne s *Permission denied*.
- Točka `.` kao odredište koristi **WORKDIR iz base imagea** (`/home/jboss`). Provjeri ga s `podman image inspect <img> --format '{{.Config.WorkingDir}}'`.
- `openjdk-17` (s Mavenom, ~450 MB) služi samo za build; **`openjdk-17-runtime`** (samo JRE, ~230 MB) je finalni image. Compiler, Maven cache i izvorni kod ostaju u odbačenom stageu.
- `COPY --from=builder` je jedina veza između stageova — sve ostalo iz build stagea nestaje.

#### Korak 4 · API kontejner na DVIJE mreže

```bash
podman run -d --name beeper-api \
  --network beeper-backend \
  -e DB_HOST=beeper-db \
  beeper-api:v1

podman network connect beeper-frontend beeper-api
podman inspect beeper-api --format '{{json .NetworkSettings.Networks}}'
```
Ili u jednoj naredbi: `--network beeper-backend,beeper-frontend`.

**Zašto ovako:** `DB_HOST=beeper-db` radi **samo** zato što su oba na `beeper-backend` — DNS ime kontejnera. API je most: govori s bazom preko backenda, a UI ga doseže preko frontenda. **Baza nikad ne vidi frontend mrežu** → i da netko probije UI, ne može direktno na bazu. To je točan odgovor na pitanje LO3-20 o segmentaciji.

#### Korak 5 · Multi-stage Containerfile za UI (Node → nginx)

```dockerfile
# ---------- STAGE 1: build ----------
FROM registry.ocp4.example.com:8443/ubi9/nodejs-22:1 AS builder
USER root
# ovaj image ima WORKDIR /opt/app-root/src
COPY . .
RUN npm install
RUN npm run build

# ---------- STAGE 2: runtime ----------
FROM registry.ocp4.example.com:8443/ubi8/nginx-118:1
COPY nginx.conf /etc/nginx/
COPY --from=builder /opt/app-root/src/dist /usr/share/nginx/html
CMD ["nginx", "-g", "daemon off;"]
```

```bash
cd /home/student/DO188/labs/comprehensive-review/beeper-ui
podman build -t beeper-ui:v1 .
```

**Objašnjenja:**
- `USER root` je nužan jer `npm install` piše u `node_modules` unutar `/opt/app-root/src`, a nodejs UBI image radi kao UID 1001 bez prava na sve poddirektorije.
- Finalni image sadrži **samo statički `dist/`** — nema Nodea, nema `node_modules` (stotine MB), nema izvornog TypeScripta.
- `nginx -g "daemon off;"` — nginx mora ostati u **foregroundu**, inače proces izađe i kontejner umre (isti princip kao LO5-4).
- UBI nginx sluša na **8080**, ne na 80 (rootless kontejneri ne smiju na portove < 1024).

#### Korak 6 · UI kontejner

```bash
podman run -d --name beeper-ui \
  --network beeper-frontend \
  -p 8080:8080 \
  beeper-ui:v1

podman ps
curl -I http://localhost:8080
```

#### Provjera i dokaz trajnosti

```bash
podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}\t{{.Networks}}"
podman logs beeper-api | tail -20

# otvori http://localhost:8080 → "New Beep" → napiši poruku

# DOKAZ PERZISTENCIJE (ovo traže):
podman rm -f beeper-db
podman run -d --name beeper-db --network beeper-backend \
  -v beeper-data:/var/lib/pgsql/data \
  -e POSTGRESQL_USER=beeper -e POSTGRESQL_PASSWORD=beeper123 -e POSTGRESQL_DATABASE=beeper \
  registry.ocp4.example.com:8443/rhel9/postgresql-13:1
podman restart beeper-api
# osvježi stranicu → poruka je i dalje tu
```

```bash
cd ~ && lab finish comprehensive-review
```

#### Mapa: koje LO3 pitanje ovaj lab pokriva

Prolazak kroz svih 23 praktična pitanja LO3 nasuprot specifikaciji ovog laba, poredano po jačini preklapanja.

##### A · Izravno preklapanje — lab te doslovno tjera da to napraviš

| Pitanje | Što u labu odgovara | Zašto je izdvojeno |
|---|---|---|
| **LO3-2** · user-defined mreža + DNS po imenu | *„two Podman networks … **that have DNS enabled**"* + `DB_HOST=beeper-db` | Cijela komunikacija API-ja s bazom stoji na razlučivanju imena `beeper-db` u IP. Lab **eksplicitno** traži DNS u specifikaciji, što znači da se ocjenjuje. Preklapanje je 1:1. |
| **LO3-4** · trajnost u named volumeu | *„Attach a **new volume called beeper-data**"* + *„message that is **persisted after restarting or recreating** the containers"* | Drugi dio nije usputna rečenica nego **kriterij prolaza** — moraš uništiti i ponovno stvoriti kontejner baze i pokazati da je poruka ostala. To je točno dokazni postupak iz LO3-4. |
| **LO3-20** · jedan kontejner na dvije mreže | *„Connect the container [beeper-api] to the **beeper-backend and beeper-frontend** networks"* | Jedini razlog zašto lab uopće ima dvije mreže. API je most, baza ostaje samo na backendu. Gotov primjer koji možeš nacrtati napamet kad traže „objasni korist segmentacije". |
| **LO3-7** · baza bez publiciranog porta | `beeper-db` i `beeper-api` nemaju **nijedan** `-p`; samo UI ima *„Map container port 8080 to host port 8080"* | Identična arhitektura kao u pitanju, samo izvedena s `podman run` umjesto composeom. Dokaz je isti: `podman port beeper-db` → prazno. |
| **LO3-13** · nginx reverse proxy | *„Copy the `beeper-ui/nginx.conf` … into `/etc/nginx/`"*, UI je `nginx-118` na frontend mreži i jedini s objavljenim portom | `beeper-ui` **jest** reverse proxy — poslužuje statički build i prosljeđuje API pozive na `beeper-api`. Otvori tu `nginx.conf` i pogledaj `proxy_pass` blok, to je gotov odgovor na pitanje. |
| **LO3-3** · dvoslojni stack koji nije Drupal/Joomla/WP | PostgreSQL + Spring Boot API + React UI, pa *„Click New Beep"* | Beeper je zapravo **troslojni**, dakle nadskup traženog, i po definiciji nije nijedan zabranjeni CMS. Legitiman odgovor na ispitu — premda je Gitea+PostgreSQL brži jer ne moraš graditi imagee. |

##### B · Isti mehanizam — lab pokazuje ispravnu stranu, pitanje pokvarenu

| Pitanje | Zašto je izdvojeno |
|---|---|
| **LO3-19** · DNS pada između različitih mreža | U Beeperu `beeper-ui` (samo frontend) **ne može** razriješiti `beeper-db` (samo backend) — i to nije bug nego namjera. Lab je živi dokaz da aardvark-dns drži zasebnu zonu po mreži. Pitanje traži da tu situaciju izazoveš i popraviš, lab traži da je od početka izbjegneš. **Isto znanje, obrnut smjer.** |
| **LO3-1** · pod + komunikacija preko localhosta | Izdvojeno kao **kontrast, ne kao pokrivenost**. Beeper namjerno koristi mreže, a ne pod. Kad razumiješ zašto — nginx i Spring Boot bi se potukli oko portova u dijeljenom network namespaceu, a baza ne pripada istoj jedinici skaliranja — znaš odgovoriti na potpitanje „kad pod, a kad mreža". |

##### C · Jedan korak od laba — vrijedi odvježbati nad Beeperom

| Pitanje | Zašto je izdvojeno |
|---|---|
| **LO3-14** · adminer / pgadmin | Lab daje bazu bez ijednog alata za pregled. Dodavanje `adminer` na `beeper-backend` je jedna naredba i odmah dokazuje i LO3-2 (DNS na `beeper-db`) i LO3-7 (baza i dalje bez porta prema hostu). Najjeftinije proširenje laba. |
| **LO3-11** · bind mount za config + volume za podatke | Lab pokriva **pola**: `beeper-data` je named volume za podatke. Drugu polovicu radi **suprotno** od pitanja — `nginx.conf` se peče u image s `COPY`. Prebaci ga na `-v ./nginx.conf:/etc/nginx/conf.d/default.conf:ro,Z` i imaš oba obrasca u istom stacku, plus odgovor zašto (mijenjaš config bez rebuilda). |
| **LO3-12** · backup i restore volumena | `beeper-data` je točno volume kakav bi u praksi backupirao, a lab ga ne dira dalje od stvaranja. Helper kontejner je jedini pouzdan način da dođeš do rootless volumena — Beeper je prirodno mjesto da to isprobaš. |
| **LO3-21** · `compose logs` / `ps` za troubleshooting | Lab nije compose, ali *„Troubleshoot containers"* je **eksplicitno naveden outcome** ovog laba, i realno ćeš `podman logs beeper-api` pokrenuti desetak puta dok Maven build i konekcija na bazu ne sjednu. Ista dijagnostička petlja, samo bez `compose` prefiksa. |

##### D · Što Beeper lab NE pokriva — vježbaj odvojeno

| Pitanje | Zašto ga lab ne dotiče |
|---|---|
| **LO3-5** · `podman secret` | Lab lozinku prosljeđuje golim `-e POSTGRESQL_PASSWORD=beeper123` — radi upravo ono **nesigurno** što pitanje traži da zamijeniš |
| **LO3-6, 8, 9, 10, 17, 22, 23** | Sve compose-bazirano (healthcheck, `env_file`, `--scale`, limiti, `down --volumes`); lab je čisti `podman run` |
| **LO3-15, 16** | `generate kube` / `kube play` — lab ne ide prema Kubernetesu |
| **LO3-18** | `podman pod inspect` i namespaceovi — lab ne koristi nijedan pod |

##### Sažetak i najisplativiji sljedeći korak

Od 23 LO3 pitanja Beeper **izravno pokriva 6** (LO3-2, 3, 4, 7, 13, 20), **konceptualno još 2** (LO3-1, 19) i **stoji na korak od 4** (LO3-11, 12, 14, 21).

- Odradi Beeper i dodaj mu **adminer**, jedan **bind-mountani `nginx.conf`** i jedan **backup volumena** → pokrio si **12 od 23 pitanja** kroz jednu vježbu.
- Zatim isti stack **prepiši u `compose.yaml`** → odjednom dobiješ LO3-6, 8, 9, 10, 17, 21 i 23 na aplikaciji koju već poznaješ, bez učenja novog primjera.
- Ostaje samo troje za zasebnu vježbu: **LO3-5** (secret), **LO3-15/16** (`generate kube` + `kube play`) i **LO3-18** (pod namespaceovi).

---

## PROJEKT · tempconverter lokalno s podmanom

> Projektni zadatak (`Intro to DevOps Project 2025 2026`) nosi **3 boda za LO3**: 1 za lokalni deploy podmanom, 1 za to da aplikacija stvarno komunicira s bazom, 1 za napisane unit i integration testove. Aplikacija: **Flask** app s [github.com/jstanesic/tempconverter](https://github.com/jstanesic/tempconverter), baza **MySQL 8**.

**Varijable okoline koje aplikacija čita:** `DB_USER`, `DB_PASS`, `DB_HOST`, `DB_NAME` (spajanje na bazu) te `STUDENT` i `COLLEGE` (ime i fakultet koji se ispisuju na stranici).

### Containerfile za Flask aplikaciju

```dockerfile
FROM docker.io/library/python:3.12-slim

# (a) ažuriraj SVE pakete kao dio builda
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends gcc default-libmysqlclient-dev && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# (c) ovisnosti PRIJE koda → layer caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd --create-home --uid 1001 appuser && chown -R appuser:appuser /app
USER appuser

# (b) izloži port 5000 TCP
EXPOSE 5000/tcp

# (d) ispravna naredba za pokretanje Flaska
CMD ["python", "app.py"]
```

```bash
podman build -t tempconverter:latest .
podman images | grep tempconverter
```

**Četiri stvari koje se ovdje ocjenjuju, i zašto su tako napisane:**

| Zahtjev | Kako | Zašto baš tako |
|---|---|---|
| (a) ažuriraj pakete | `apt-get update && apt-get upgrade -y` u **istom** RUN-u, pa `rm -rf /var/lib/apt/lists/*` | odvojeni `update` se cachira i sljedeći build instalira iz zastarjelog indeksa; čišćenje u drugom sloju ne smanjuje image (LO5-10, LO5-16) |
| (b) EXPOSE 5000 | `EXPOSE 5000/tcp` | **samo dokumentacija** — ne otvara port. Objavljivanje ide s `-p` (LO5-14/15) |
| (c) requirements | `COPY requirements.txt` prije `COPY . .` | promjena koda ne invalidira sloj s ovisnostima (LO5-12) |
| (d) start naredba | **exec forma** `CMD ["python", "app.py"]` | shell forma stavlja `sh` na PID 1 i ne prosljeđuje SIGTERM (LO5-11) |

**Najčešća greška kod Flaska:** ako `app.py` sadrži `app.run()` bez argumenata, Flask sluša na **`127.0.0.1`** — što unutar kontejnera znači *samo unutar tog kontejnera*. Port mapiranje onda ne radi iako je sve ostalo točno.

```python
app.run(host="0.0.0.0", port=5000)      # ISPRAVNO za kontejner
```
Ili preko naredbe:
```dockerfile
ENV FLASK_APP=app.py
CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]
```
Provjera iznutra: `podman exec tempconv-app ss -tlnp` → mora pisati `0.0.0.0:5000`, ne `127.0.0.1:5000`.

### Baza + spajanje kao NON-ROOT korisnik

Zahtjev iz projekta: *„Application should successfully connect to the database as a non-root user."*

```bash
podman network create tempconv-net
podman volume create tempconv-data

podman run -d --name tempconv-db --network tempconv-net \
  -e MYSQL_ROOT_PASSWORD=RootPw123 \
  -e MYSQL_DATABASE=tempconverter \
  -e MYSQL_USER=tempuser \
  -e MYSQL_PASSWORD=TempPw123 \
  -v tempconv-data:/var/lib/mysql \
  docker.io/library/mysql:8
```

`MYSQL_USER` + `MYSQL_PASSWORD` automatski stvore **non-root korisnika** s punim pravima **samo nad** bazom iz `MYSQL_DATABASE`. To je cijeli trik — ne treba ručni SQL.

**Dokaz (screenshot za dokumentaciju):**
```bash
podman exec -it tempconv-db mysql -utempuser -pTempPw123 \
  -e "SELECT CURRENT_USER(); SHOW GRANTS;"
# → tempuser@%   GRANT ALL PRIVILEGES ON `tempconverter`.* TO `tempuser`@`%`
```

Ručna varijanta ako je baza već stvorena:
```sql
CREATE USER 'tempuser'@'%' IDENTIFIED BY 'TempPw123';
GRANT SELECT, INSERT, UPDATE, DELETE ON tempconverter.* TO 'tempuser'@'%';
FLUSH PRIVILEGES;
```
**Objašnjenje za dokumentaciju:** root ima prava nad **svim** bazama i može mijenjati korisnike i shemu. Aplikacijski korisnik dobiva samo ono što mu treba nad **jednom** bazom — *least privilege*. Ako netko probije aplikaciju (SQL injection), šteta je ograničena na tu bazu i te operacije.

### Pokretanje aplikacije

```bash
podman run -d --name tempconv-app --network tempconv-net -p 8080:5000 \
  -e DB_HOST=tempconv-db \
  -e DB_NAME=tempconverter \
  -e DB_USER=tempuser \
  -e DB_PASS=TempPw123 \
  -e STUDENT="Mislav Uvanović" \
  -e COLLEGE="Algebra Bernays University" \
  tempconverter:latest

podman logs -f tempconv-app
curl http://localhost:8080
```
`DB_HOST=tempconv-db` radi **samo** zato što su oba kontejnera na user-defined mreži `tempconv-net` (DNS po imenu kontejnera — LO3-2). Na default mreži bi palo s *Name or service not known*.

### Ista stvar preko composea

```yaml
# compose.yaml
services:
  db:
    image: docker.io/library/mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: RootPw123
      MYSQL_DATABASE: tempconverter
      MYSQL_USER: tempuser
      MYSQL_PASSWORD: TempPw123
    volumes:
      - dbdata:/var/lib/mysql
    networks: [backend]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-pRootPw123"]
      interval: 5s
      timeout: 3s
      retries: 15
      start_period: 30s

  app:
    image: tempconverter:latest
    ports: ["8080:5000"]
    environment:
      DB_HOST: db
      DB_NAME: tempconverter
      DB_USER: tempuser
      DB_PASS: TempPw123
      STUDENT: "Mislav Uvanović"
      COLLEGE: "Algebra Bernays University"
    depends_on:
      db:
        condition: service_healthy
    networks: [backend]

volumes:
  dbdata:
networks:
  backend:
```
```bash
podman compose up -d
podman compose ps
podman compose logs -f app
```
Baza **nema `ports:`** → nedostupna s hosta, dostupna samo aplikaciji (LO3-7). `condition: service_healthy` rješava to što MySQL treba 20–30 s da inicijalizira data direktorij, a Flask bi u međuvremenu pao (LO3-8).

### Unit i integration testovi (1 bod)

Struktura koja se očekuje:
```
tests/
  test_unit.py           # čista logika, bez baze
  test_integration.py    # stvarna baza u kontejneru
```

```python
# tests/test_unit.py — UNIT: samo konverzija, nikakva vanjska ovisnost
import pytest
from converter import c_to_f, f_to_c

@pytest.mark.parametrize("c,f", [(0, 32), (100, 212), (-40, -40), (37, 98.6)])
def test_celsius_to_fahrenheit(c, f):
    assert c_to_f(c) == pytest.approx(f)

def test_roundtrip():
    assert f_to_c(c_to_f(21.5)) == pytest.approx(21.5)

def test_invalid_input():
    with pytest.raises((ValueError, TypeError)):
        c_to_f("nije broj")
```

```python
# tests/test_integration.py — INTEGRATION: app + stvarna baza
import os, pytest
from app import app as flask_app

@pytest.fixture
def client():
    flask_app.config["TESTING"] = True
    with flask_app.test_client() as c:
        yield c

def test_home_returns_200_and_student(client):
    r = client.get("/")
    assert r.status_code == 200
    assert os.environ["STUDENT"].encode() in r.data

def test_db_connection():
    import MySQLdb
    conn = MySQLdb.connect(
        host=os.environ["DB_HOST"], user=os.environ["DB_USER"],
        passwd=os.environ["DB_PASS"], db=os.environ["DB_NAME"])
    cur = conn.cursor()
    cur.execute("SELECT CURRENT_USER()")
    user = cur.fetchone()[0]
    assert not user.startswith("root@")      # DOKAZ non-root pristupa
    conn.close()
```

**Razlika koju moraš napisati u dokumentaciji:**

| | **Unit test** | **Integration test** |
|---|---|---|
| Što testira | jednu funkciju u izolaciji | suradnju komponenti (app ↔ baza ↔ HTTP) |
| Vanjske ovisnosti | nema (ili mockane) | stvarna baza, stvarni kontejneri |
| Brzina | milisekunde | sekunde do minute |
| Kad pada | logika je kriva | konfiguracija, mreža, shema, dozvole su krive |
| Gdje u pipelineu | prvi korak, na svaki commit | nakon podizanja stacka |

### Pipeline: testovi → build → push (1 bod)

Najjednostavnija verzija koja zadovoljava „create a simple pipeline which will run unit and integration tests and build an image":

```bash
#!/usr/bin/env bash
# pipeline.sh — pokreni s: ./pipeline.sh
set -euo pipefail

echo "=== 1/4 UNIT TESTOVI ==="
python -m pytest tests/test_unit.py -v

echo "=== 2/4 PODIZANJE TEST OKOLINE ==="
podman compose -f compose.test.yaml up -d --wait

echo "=== 3/4 INTEGRATION TESTOVI ==="
DB_HOST=127.0.0.1 DB_NAME=tempconverter DB_USER=tempuser DB_PASS=TempPw123 \
  python -m pytest tests/test_integration.py -v
podman compose -f compose.test.yaml down -v

echo "=== 4/4 BUILD I PUSH ==="
podman build -t tempconverter:latest .
podman tag tempconverter:latest quay.io/<tvoj-user>/tempconverter:latest
podman login quay.io
podman push quay.io/<tvoj-user>/tempconverter:latest

echo "PIPELINE OK"
```
`set -euo pipefail` je bitan: bez `-e` skripta nastavi dalje nakon pada testa i svejedno pushne pokvaren image — što je upravo ono što pipeline treba spriječiti.

#### GitHub Actions verzija

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test-and-build:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8
        env:
          MYSQL_ROOT_PASSWORD: RootPw123
          MYSQL_DATABASE: tempconverter
          MYSQL_USER: tempuser
          MYSQL_PASSWORD: TempPw123
        ports: ["3306:3306"]
        options: >-
          --health-cmd="mysqladmin ping -h localhost -uroot -pRootPw123"
          --health-interval=5s --health-timeout=3s --health-retries=20

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.12"}

      - name: Install deps
        run: pip install -r requirements.txt pytest

      - name: Unit tests
        run: pytest tests/test_unit.py -v

      - name: Integration tests
        env:
          DB_HOST: 127.0.0.1
          DB_NAME: tempconverter
          DB_USER: tempuser
          DB_PASS: TempPw123
          STUDENT: "Mislav Uvanović"
        run: pytest tests/test_integration.py -v

      - name: Build image
        run: podman build -t tempconverter:${{ github.sha }} .

      - name: Push to Quay
        run: |
          podman login -u "${{ secrets.QUAY_USER }}" -p "${{ secrets.QUAY_TOKEN }}" quay.io
          podman tag tempconverter:${{ github.sha }} quay.io/${{ secrets.QUAY_USER }}/tempconverter:latest
          podman push quay.io/${{ secrets.QUAY_USER }}/tempconverter:latest
```

#### Jenkinsfile (ako radiš s Jenkinsom)

> Dozvoljene domene za ispit uključuju **`get.jenkins.io`** — dobar razlog da znaš barem kostur.

```groovy
pipeline {
  agent any
  environment {
    IMAGE = "quay.io/mislav/tempconverter"
    REG   = credentials('quay-creds')
  }
  stages {
    stage('Unit')        { steps { sh 'pytest tests/test_unit.py -v --junitxml=unit.xml' } }
    stage('Up')          { steps { sh 'podman compose -f compose.test.yaml up -d --wait' } }
    stage('Integration') { steps { sh 'pytest tests/test_integration.py -v --junitxml=int.xml' } }
    stage('Build')       { steps { sh "podman build -t ${IMAGE}:${BUILD_NUMBER} ." } }
    stage('Push') {
      steps {
        sh "podman login -u $REG_USR -p $REG_PSW quay.io"
        sh "podman push ${IMAGE}:${BUILD_NUMBER}"
        sh "podman tag ${IMAGE}:${BUILD_NUMBER} ${IMAGE}:latest && podman push ${IMAGE}:latest"
      }
    }
  }
  post {
    always  { junit '*.xml'; sh 'podman compose -f compose.test.yaml down -v || true' }
    failure { echo 'Pipeline pao — image NIJE pushan.' }
  }
}
```

**Rečenica za dokumentaciju:** pipeline provodi *fail fast* — jeftini unit testovi idu prvi, pa skuplji integration testovi, i **tek ako sve prođe** gradi se i objavljuje image. Tako pokvaren artefakt nikad ne dođe do registryja, a povratna informacija stiže u sekundama umjesto nakon deploya.

### Usporedba potrošnje resursa: kontejner vs. VM (LO1, 2 boda)

Nosi bodove **samo ako si stvarno mjerio**, ne ako si prepisao teoriju.

```bash
# KONTEJNER
podman stats --no-stream
podman inspect tempconv-app --format '{{.State.Pid}}'
ps -o pid,rss,vsz,comm -p <pid>
podman exec tempconv-app cat /sys/fs/cgroup/memory.current
du -sh ~/.local/share/containers/storage        # storage footprint
time podman run --rm tempconverter:latest python -c "print('up')"   # vrijeme starta

# VM (za usporedbu)
free -m ; nproc ; df -h /
systemd-analyze                                  # vrijeme boota
qemu-img info /var/lib/libvirt/images/vm.qcow2   # veličina diska
```

| Mjera | Kontejner | VM |
|---|---|---|
| Vrijeme pokretanja | ~0,1–1 s | 20–60 s (boot cijelog OS-a) |
| Memorija u praznom hodu | 5–50 MB (samo proces) | 512 MB – 2 GB (kernel + OS servisi) |
| Veličina na disku | 50–200 MB (dijeljeni slojevi) | 2–20 GB (cijeli root filesystem) |
| Kernel | **dijeli hostov** | vlastiti, preko hipervizora |
| Izolacija | namespaces + cgroups + SELinux/seccomp | hardverska, jača |
| Gustoća po hostu | stotine | desetci |

Ključna rečenica: kontejner nije mala VM — **nema vlastiti kernel ni boot**, nego je izolirani proces na hostu. Zato je razlika u startu i memoriji reda veličine, a ne postotaka. Cijena je slabija izolacija: bijeg iz kontejnera vodi na host kernel.

---

## ROKOVI · LO3 zadaci s prošlih ispita

> Rekonstruirano iz `exam rok kolokvij.docx` (7 zadataka + 7 snimki rezultata) i `exam rok lipanj.docx`. To su **bilješke i pokušaji rješenja**, ne službeni tekstovi zadataka — ali obrazac se jasno vidi i ponavlja.

**Što je zabilježeno na kolokviju:** `Lo1_1`, `Lo1_2`, `Lo1_3`, `Lo2_1`, `Lo2_2`, `Lo3_3`, `Lo3_1`.
**`Lo3_2` nije zabilježen** — njegov sadržaj ne znamo. Sudeći po ostatku, najvjerojatniji kandidati su pod (LO3-1), trajnost (LO3-4) ili `podman secret` (LO3-5).

---

### Rok · Lo3_1 — Drupal + PostgreSQL kroz Compose

#### Kako je zapisano na ispitu

```yaml
services:
  db:
    image: docker.io/library/postgres:latest
    container_name: postgres-srv
    environment:
      POSTGRES_PASSWORD: my-secret-pw
      POSTGRES_DB: drupal
      POSTGRES_USER: drupal_user
      PGPASSWORD: drupal_password
  drupal:
    image: docker.io/library/drupal:latest
    container_name: drupal-srv
    ports:
      - "8080:80"
    depends_on:
      - db
```
```bash
podman-compose up -d
```

#### Tri stvari koje ovdje treba primijetiti

**1. `PGPASSWORD` nije lozinka baze.** Ovo je zamka i najvjerojatnije mjesto gubitka boda.

| Varijabla | Što radi |
|---|---|
| `POSTGRES_PASSWORD` | postavlja lozinku korisnika **pri inicijalizaciji** imagea |
| `PGPASSWORD` | **klijentska** varijabla za `psql` i libpq — ne stvara ni ne mijenja nijednog korisnika u bazi |

Lozinka korisnika `drupal_user` je dakle **`my-secret-pw`**, ne `drupal_password`. Ako u Drupalovoj instalaciji upišeš `drupal_password`, prijava na bazu pada. `PGPASSWORD` u ovoj konfiguraciji nema nikakvu funkciju i može se izbaciti.

**2. Nema volumena** → podaci nestaju s kontejnerom. Ako zadatak traži trajnost, treba ih dodati (dolje).

**3. Nema `networks:` bloka — i to je u redu.** Compose sam stvara zadanu mrežu projekta s DNS-om, pa Drupal bazu nalazi po **imenu servisa `db`** (ne po `container_name` i ne po `localhost`).

#### Ispravljena, potpuna verzija

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    container_name: postgres-srv
    environment:
      POSTGRES_DB: drupal
      POSTGRES_USER: drupal_user
      POSTGRES_PASSWORD: my-secret-pw
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U drupal_user -d drupal"]
      interval: 5s
      timeout: 3s
      retries: 15
      start_period: 20s

  drupal:
    image: docker.io/library/drupal:10-apache
    container_name: drupal-srv
    ports:
      - "8080:80"
    volumes:
      - drupal-sites:/var/www/html/sites
    depends_on:
      db:
        condition: service_healthy

volumes:
  postgres-data:
  drupal-sites:
```

```bash
podman compose -f compose.yaml config      # validacija prije pokretanja
podman compose -f compose.yaml up -d
podman compose -f compose.yaml ps
podman compose -f compose.yaml logs --tail 30 db   # čekaj: ready to accept connections
```

#### Dovršetak instalacije (traži se „full install")

Otvori `http://localhost:8080`, jezik → profil **Standard** → korak baze:

| Polje | Vrijednost |
|---|---|
| Database type | **PostgreSQL** |
| Database name | `drupal` |
| Database username | `drupal_user` |
| Database password | `my-secret-pw` |
| Advanced options → **Host** | `db` |
| Advanced options → **Port** | `5432` |

Zatim **Configure site**: naziv stranice, e-mail, i **Drupalov admin račun** — koji je zaseban od `drupal_user`. `drupal_user` je račun **baze**; `admin` je račun **web-aplikacije**.

**Zamka:** default host u Drupalovoj instalaciji je `localhost`. Unutar Drupal kontejnera `localhost` označava **sam Drupal**, ne bazu. Mora se upisati `db`.

Ako zadatak traži i dokaz trajnosti: stvori stranicu (Content → Add content → Basic page), pa `podman compose down` i `up -d`, pa Ctrl+F5 — stranica mora ostati.

---

### Rok · Lo3_3 — vlastiti Alpine + nginx image

#### Kako je zapisano na ispitu

```dockerfile
FROM alpine:latest
ENV STUDENT="Mislav Uvanovic"
RUN apk add --no-cache nginx
RUN mkdir -p /run/nginx
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
```bash
podman build -t alpine-nginx .
podman run -d --name my-alpine-web -p 8082:80 alpine-nginx
```

**Snimka rezultata uz ovaj zadatak prikazuje `HTTP/1.1 404 Not Found` sa zaglavljem `Server: nginx`.**

#### Zašto 404 — i zašto to NIJE kvar

404 znači da je **nginx primio zahtjev i odgovorio**. To nije odbijena veza (`Connection refused`) ni ugašen kontejner. Uzrok:

```bash
podman exec my-alpine-web nginx -T      # veliko T = ispiši SVE učitane konfiguracije
```

Alpineov paket nginx dolazi s `/etc/nginx/http.d/default.conf` koji sadrži:
```nginx
location / { return 404; }
```

**Direktiva je eksplicitna** — dodavanje `index.html` ne bi ništa promijenilo, jer nginx nikad ne dođe do posluživanja datoteka.

#### Druga zamka: `ENV STUDENT` ne ispisuje ime na stranici

`ENV` postavlja varijablu okruženja. Provjerava se ovako:
```bash
podman exec my-alpine-web printenv STUDENT     # → Mislav Uvanovic
```
Da bi se ime **vidjelo u browseru**, mora se zasebno upisati u konfiguraciju ili HTML.

#### Potpuno rješenje

`default.conf` u istom direktoriju kao Containerfile:
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    location / {
        default_type text/html;
        return 200 "<h1>Mislav Uvanovic - nginx radi!</h1>\n";
    }
}
```

```dockerfile
FROM docker.io/library/alpine:latest
ENV STUDENT="Mislav Uvanovic"
RUN apk add --no-cache nginx
RUN mkdir -p /run/nginx
COPY default.conf /etc/nginx/http.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```bash
podman build -t alpine-nginx .
podman stop my-alpine-web && podman rm my-alpine-web     # OBAVEZNO
podman run -d --name my-alpine-web -p 8082:80 localhost/alpine-nginx:latest
curl -i http://localhost:8082      # → HTTP/1.1 200 OK + HTML
```

**Ključno:** `podman restart` **ne bi** prikazao novi sadržaj — postojeći kontejner i dalje koristi stari image. Novi build zahtijeva `rm` + `run`.

**Alternativa s HTML datotekom** (ako zadatak traži datoteku, a ne `return 200`):
```dockerfile
COPY index.html /usr/share/nginx/html/index.html
COPY default.conf /etc/nginx/http.d/default.conf     # i dalje treba — bez toga ostaje return 404
```

#### Putanja konfiguracije ovisi o imageu — čest gubitak boda

| Image | Putanja server bloka |
|---|---|
| **Alpine + `apk add nginx`** | `/etc/nginx/http.d/default.conf` |
| **Službeni `nginx` image** | `/etc/nginx/conf.d/default.conf` |
| **UBI `nginx-118`** | `/etc/nginx/nginx.conf` + sluša na **8080** |

Provjeri s `nginx -T` umjesto da pogađaš.

---

### Rok · lipanj — Drupal + MySQL, komunikacija i trajnost

> Bilješka: *„drupal i mysql containeri valjda komuniciraju i napraviti full install appa"* i *„napraviti persistent volument za drupal i nek bude nest u drupal data"*

Isti zadatak kao `Lo3_1`, ali s **MySQL-om** i **bez Composea** (zasebne `podman run` naredbe), uz izričit zahtjev za trajnošću.

```bash
podman network create drupal-net
podman volume create mysql-data
podman volume create drupal-sites

podman run -d --name db --network drupal-net \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -e MYSQL_DATABASE=drupal \
  -e MYSQL_USER=drupaluser \
  -e MYSQL_PASSWORD=drupal123 \
  -v mysql-data:/var/lib/mysql \
  docker.io/library/mysql:8.0

podman logs --tail 30 db        # čekaj: ready for connections ... port: 3306

podman run -d --name web --network drupal-net -p 8080:80 \
  -v drupal-sites:/var/www/html/sites \
  docker.io/library/drupal:10-apache
```

Instalacija na `http://localhost:8080`: type **MySQL/MariaDB**, name `drupal`, user `drupaluser`, password `drupal123`, **Host `db`**, Port `3306`.

**Baza namjerno nema `-p`** — Drupal joj pristupa preko mreže. Ako je objaviš, to nije greška, ali je nepotrebno izlaganje.

#### Dokaz trajnosti — postupak koji nosi bod

Zahtjev *„nek bude nest u drupal data"* znači: **stvori sadržaj pa dokaži da preživi**.

```bash
# 1. U Drupalu: Content → Add content → Basic page
#    Title: "Provjera trajne pohrane", Body: "Mislav LO3" → Save → zabilježi /node/1

# 2. Datoteka u Drupalovom volumenu (dokaz za drupal-sites)
podman exec web sh -c "echo Mislav-LO3 > /var/www/html/sites/default/files/provjera.txt"

# 3. UNIŠTI oba kontejnera (ne volumene)
podman stop web db && podman rm web db
podman volume ls                      # mysql-data i drupal-sites su tu

# 4. Ponovno stvori istim naredbama, pa provjeri
podman exec web cat /var/www/html/sites/default/files/provjera.txt    # → Mislav-LO3
# i u browseru Ctrl+F5 na /node/1 → naslov i tekst su tu
```

**Zašto `restart` nije dokaz:** zapisivi sloj kontejnera preživljava restart istog kontejnera. Dokaz trajnosti traži **`rm` + ponovno stvaranje** s istim volumenom.

**Dva volumena, dva različita dokaza:** tekst stranice je u **MySQL** volumenu, a datoteke i postavke u **Drupal** volumenu. Ako pokažeš samo tekst stranice, nisi dokazao `drupal-sites`.

---

### Obrazac koji se ponavlja na oba roka

| Element | Lo3_1 | Lo3_3 | Lipanj |
|---|---|---|---|
| Aplikacija + baza koje **stvarno komuniciraju** | ✅ | — | ✅ |
| Dovršena instalacija kroz browser | ✅ | — | ✅ |
| Vlastiti image iz Containerfilea | — | ✅ | — |
| Trajna pohrana s dokazom | — | — | ✅ |
| Objavljivanje porta `-p` | ✅ | ✅ | ✅ |

**Zaključak za pripremu:** na LO3 gotovo sigurno dolazi **app + baza koje moraju raditi zajedno**, s instalacijom dovršenom u browseru. Vježbaj obje varijante — `podman run` na zajedničkoj mreži **i** compose — jer su se pojavile obje. Uz to pripremi jedan **vlastiti Containerfile** koji poslužuje tvoje ime.

---

### LO3 · Šalabahter naredbi

```bash
# PODOVI
podman pod create --name p -p 8080:80 [--share net,ipc,uts,pid]
podman pod ps | podman pod inspect p | podman pod stop/start/rm p
podman run -d --pod p --name c IMAGE

# MREŽE
podman network create [--internal] [--subnet 10.89.0.0/24] net
podman network ls | inspect net | connect net c | disconnect net c | rm net
podman run --network net --name c --network-alias db IMAGE

# VOLUMENI
podman volume create v | ls | inspect v | rm v | prune
podman volume export v -o v.tar | podman volume import v2 v.tar
-v v:/path              # named volume
-v ./dir:/path:ro,Z     # bind mount, read-only, SELinux private

# SECRETS
printf 'pw' | podman secret create name -
podman run --secret name[,type=env,target=VAR] IMAGE

# COMPOSE
podman compose up -d [--scale app=3] | down [-v] | ps | logs -f svc | exec svc sh | config

# KUBE MOST
podman generate kube pod > f.yaml     (ili: podman kube generate)
podman kube play f.yaml | podman kube play --down f.yaml
```


---

# LO4 — Ubrzana isporuka višeslojnih aplikacija pomoću kontejnera (Kubernetes)

> **Najveći dio ispita — 53 pitanja.** Sve se svodi na 5 skupina: **workloadi** (Deployment, StatefulSet, DaemonSet, Job, CronJob), **updateovi i rollback**, **storage i konfiguracija** (Volume, ConfigMap, Secret, PVC), **mreža** (Service, DNS, port-forward) i **probes**.

---

### 0. Preživljavanje na ispitu: imperativno → YAML

Nikad ne piši veliki manifest od nule. Generiraj ga:

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3 \
  --dry-run=client -o yaml > web.yaml
```

`--dry-run=client -o yaml` je **najvažnija naredba na cijelom ispitu**. Radi za: `create deployment/job/cronjob/configmap/secret/service`, `expose`, `run`.

Ubrzaj se:
```bash
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"
k create deploy web --image=nginx $do > web.yaml
```

Traži pomoć u samom clusteru (dozvoljeno, brže od dokumentacije):
```bash
kubectl explain deployment.spec.strategy --recursive
kubectl api-resources          # popis svih vrsta + kratice + apiVersion
kubectl get deploy web -o yaml
```

#### Hijerarhija koju moraš znati
```
Deployment  →  ReplicaSet  →  Pod  →  Container
   (update       (broj         (jedinica     (proces)
    strategija)   replika)      rasporedbe)
```
Deployment upravlja ReplicaSetovima; **svaka promjena pod templatea stvara NOVI ReplicaSet**. Stari se čuvaju (za rollback) do `revisionHistoryLimit`.

---

## A · DEPLOYMENTI I SKALIRANJE

### LO4-1 · Deployment `web`, nginx:1.25, 3 replike + export YAML

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deploy,rs,pods -l app=web

# export generiranog YAML-a
kubectl get deployment web -o yaml > web.yaml
# čisti YAML bez status/metadata smeća:
kubectl get deployment web -o yaml --show-managed-fields=false > web.yaml
```

Ručni manifest (nauči napamet ovaj kostur):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web            # MORA odgovarati template labelama
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### LO4-2 · Autentificirano povlačenje s Docker Huba (pull limit)

**Odgovor: treba `docker-registry` Secret + `imagePullSecrets`.**

```bash
kubectl create secret docker-registry dockerhub-cred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<user> \
  --docker-password=<token> \
  --docker-email=<mail>
```

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
      - name: dockerhub-cred
      containers:
      - name: app
        image: docker.io/myuser/app:1.0
```

Alternativa za sve podove u namespaceu — zakači na ServiceAccount:
```bash
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets":[{"name":"dockerhub-cred"}]}'
```

**Objašnjenje:** Docker Hub ograničava anonimne pullove po IP-u (rate limit). Autentificirani korisnik ima veću kvotu. Tip secreta je `kubernetes.io/dockerconfigjson`, sadržaj je `.dockerconfigjson` ključ.

### LO4-3 · Skaliranje 3 → 5 na dva načina

```bash
# NAČIN 1 — imperativno
kubectl scale deployment web --replicas=5
kubectl get rs                     # ISTI ReplicaSet, DESIRED 5

# NAČIN 2 — deklarativno (uredi manifest)
kubectl edit deployment web        # promijeni replicas: 5
# ili u datoteci:
sed -i 's/replicas: 3/replicas: 5/' web.yaml
kubectl apply -f web.yaml
```

**Ključno:** skaliranje **NE stvara novi ReplicaSet** — mijenja se samo `spec.replicas` postojećeg RS-a, jer se pod template nije promijenio. Novi RS nastaje **samo** kad se promijeni `spec.template` (image, env, resources…).

```bash
kubectl get rs -l app=web -o wide   # DESIRED/CURRENT/READY
```

### LO4-11 · Label selector + odnos Deployment → RS → Pod

```bash
kubectl get pods -l app=web
kubectl get pods --show-labels
kubectl get pods -l 'app in (web,api)'
kubectl get pods -l app=web,tier=frontend      # AND
kubectl get pods -l 'app!=web'
```

**Odnos selektora:**
1. `Deployment.spec.selector.matchLabels` bira **koje ReplicaSetove/podove** posjeduje.
2. Deployment doda ReplicaSetu **dodatnu labelu `pod-template-hash`** (npr. `web-6f8d9c`) → tako razlikuje svoju "generaciju" podova od tuđih.
3. ReplicaSet svojim selektorom (`app=web` + `pod-template-hash=…`) broji svoje podove.

Zato dva RS-a istog Deploymenta ne "kradu" podove jedan drugom.
**`spec.selector` je immutable** nakon stvaranja — mijenjaš li ga, moraš obrisati i ponovno stvoriti Deployment.

---

## B · UPDATEOVI, ROLLBACK I STRATEGIJE

### LO4-4 · Rolling update 1.25 → 1.27

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web       # prati napredak
kubectl get rs -l app=web                   # STARI RS → 0, NOVI RS → 3
kubectl describe deployment web | grep -A5 Events
```

`kubectl set image deployment/<deploy> <container-name>=<new-image>` — pazi da je `<container-name>` ime **kontejnera**, ne deploymenta.

### LO4-5 · Povijest rolloutova i rollback + CHANGE-CAUSE

```bash
kubectl rollout history deployment/web
kubectl rollout history deployment/web --revision=2     # detalji revizije
kubectl rollout undo deployment/web                     # na prethodnu
kubectl rollout undo deployment/web --to-revision=1     # na točnu
```

**CHANGE-CAUSE** dolazi iz anotacije `kubernetes.io/change-cause`. Prazan je ako je ne postaviš:
```bash
kubectl annotate deployment/web kubernetes.io/change-cause="update to nginx 1.27" --overwrite
# ili u manifestu:
metadata:
  annotations:
    kubernetes.io/change-cause: "update to nginx 1.27"
```
(Stari `--record` flag je deprecated/uklonjen.)

### LO4-6 · `maxSurge: 1`, `maxUnavailable: 0`

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

**Učinak:** nikad manje od `replicas` **spremnih** podova → **nula prekida usluge**. Update ide korak po korak: podigne 1 dodatni pod (ukupno N+1), čeka da bude Ready, tek onda gasi 1 stari. Cijena: potreban je **višak kapaciteta** za +1 pod i update je **sporiji**.

Default je `maxSurge: 25%, maxUnavailable: 25%` — brže, ali privremeno smanjuje kapacitet.

| Postavka | Značenje |
|---|---|
| `maxSurge` | koliko podova **iznad** `replicas` smije postojati tijekom updatea |
| `maxUnavailable` | koliko podova smije **nedostajati** (ne biti Ready) tijekom updatea |
| oba 0 | **nedozvoljeno** — rollout bi zapeo |

### LO4-7 · `Recreate` strategija

```yaml
spec:
  strategy:
    type: Recreate
```

Ubije **sve** stare podove, pa tek onda stvori nove → **postoji downtime**.

**Kad je nužno:**
- Aplikacija koristi **ReadWriteOnce** PVC — dvije verzije ne mogu istovremeno montirati isti volume.
- **Migracija sheme baze** koja je nekompatibilna unatrag — stara i nova verzija ne smiju istovremeno pisati.
- Aplikacija drži **ekskluzivni lock** (leader-only legacy app, licencni lock).
- Singleton procesi koji ne podnose dvije instance.

### LO4-9 · `revisionHistoryLimit: 3`

```yaml
spec:
  revisionHistoryLimit: 3
```

Čuva **3 stara ReplicaSeta** (skalirana na 0) → možeš se vratiti najviše 3 revizije unatrag. Default je 10. `0` = nema rollbacka uopće. Veći broj troši etcd i zatrpava `kubectl get rs`.

### LO4-10 · Nepostojeći tag → rollout zapne

```bash
kubectl set image deployment/web nginx=nginx:doesnotexist
kubectl rollout status deployment/web       # visi
kubectl get pods                            # novi pod: ImagePullBackOff / ErrImagePull
kubectl get rs                              # stari RS i dalje 3/3 READY
curl <service>                              # aplikacija I DALJE RADI
```

**Zašto stari podovi i dalje služe promet:** rolling update **ne gasi stari pod dok novi ne postane `Ready`**. Novi pod nikad ne dođe do Running → `maxUnavailable` prag se ne prekorači → controller stane i čeka. To je ugrađena zaštita. Nakon `progressDeadlineSeconds` (default 600 s) Deployment dobije uvjet `ProgressDeadlineExceeded`, ali stari podovi ostaju.

**Popravak:** `kubectl rollout undo deployment/web` ili `kubectl set image` s ispravnim tagom.

---

## C · OSTALI WORKLOADI

### LO4-15 · Goli Pod vs. Pod pod Deploymentom

```bash
kubectl run mypod --image=busybox --command -- sleep 3600
kubectl delete pod mypod        # NESTAO ZAUVIJEK

kubectl delete pod <pod-iz-deploymenta>    # ReplicaSet ga ODMAH ponovno stvori
```

**Objašnjenje:** goli Pod nema kontroler koji nadgleda željeno stanje. Ako pod padne, node umre ili ga netko obriše — nema ga. Deployment→ReplicaSet je **reconciliation loop**: stalno uspoređuje *desired* i *actual* i popravlja razliku → self-healing, rescheduling na drugi node, rolling update, rollback. Goli podovi se koriste samo za debug/jednokratne zadatke.

### LO4-14 · Deployment `httpd:2.4`, 2 replike, imenovani port, `nodeSelector`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
spec:
  replicas: 2
  selector:
    matchLabels: {app: httpd}
  template:
    metadata:
      labels: {app: httpd}
    spec:
      nodeSelector:
        disktype: ssd
      containers:
      - name: httpd
        image: httpd:2.4
        ports:
        - name: http          # ← imenovani port
          containerPort: 80
```

**Zašto ostaje Pending:** `nodeSelector` je **hard** zahtjev. Scheduler filtrira nodove po labelama; ako nijedan node nema `disktype=ssd`, nema kandidata → pod ostaje `Pending` s eventom `0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector`.

```bash
kubectl get nodes --show-labels
kubectl label node minikube disktype=ssd      # popravak
kubectl describe pod <pod> | tail -20
```

**Imenovani port** se onda referencira u Serviceu kao `targetPort: http` — otporno na promjenu broja porta.

### LO4-13 · Sidecar kontejner

```yaml
    spec:
      volumes:
      - name: shared-logs
        emptyDir: {}
      containers:
      - name: app
        image: nginx:1.25
        volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
      - name: log-shipper            # ← sidecar
        image: busybox
        command: ["sh","-c","tail -F /logs/access.log"]
        volumeMounts:
        - name: shared-logs
          mountPath: /logs
```

```bash
kubectl logs <pod> -c log-shipper
kubectl exec -it <pod> -c app -- sh
```

**Kako dijele:** oba kontejnera su u **istom Podu** → dijele **network namespace** (isti IP, vide se na `localhost`, ne smiju koristiti isti port) i mogu montirati **iste volumene** Poda. Ne dijele filesystem po defaultu (svaki ima svoj image), samo eksplicitno montirane volumene. Tipični sidecari: log shipper, proxy (service mesh), config reloader, metrics exporter.

### LO4-16 · StatefulSet redis:7, 3 replike + headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  clusterIP: None            # ← HEADLESS
  selector: {app: redis}
  ports:
  - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis         # ← mora pokazivati na headless Service
  replicas: 3
  selector:
    matchLabels: {app: redis}
  template:
    metadata:
      labels: {app: redis}
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:      # ← PVC PO REPLICI
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

```bash
kubectl get pods -l app=redis      # redis-0, redis-1, redis-2 (STABILNA IMENA)
kubectl get pvc                    # data-redis-0, data-redis-1, data-redis-2

# per-pod DNS zapisi
kubectl run tmp --rm -it --image=busybox --restart=Never -- \
  nslookup redis-0.redis.default.svc.cluster.local
```

**Format DNS-a:** `<pod-name>.<service-name>.<namespace>.svc.cluster.local`

### LO4-21 · StatefulSet stvara/gasi podove REDOM

```bash
kubectl scale statefulset redis --replicas=3
kubectl get pods -w        # redis-0 Ready → tek onda redis-1 → tek onda redis-2

kubectl scale statefulset redis --replicas=0
kubectl get pods -w        # OBRNUTO: redis-2 → redis-1 → redis-0
```

| | **Deployment** | **StatefulSet** |
|---|---|---|
| Imena podova | slučajna (`web-6f8d-x9k2`) | ordinalna, stabilna (`redis-0`) |
| Redoslijed starta | **paralelno**, bez reda | **serijski, 0→N**, čeka Ready |
| Redoslijed gašenja | paralelno | **obrnuto, N→0** |
| Storage | dijeljen ili bez | **PVC po podu** (`volumeClaimTemplates`) |
| DNS | jedan VIP servisa | per-pod A zapis (headless) |
| Za što | stateless web/API | baze, Kafka, ZooKeeper, quorum sustavi |

Zašto: quorum sustavi trebaju predvidljiv identitet i redoslijed (seed node mora biti prvi gore, zadnji dolje).

### LO4-27 · Skaliranje StatefulSeta → PVC po replici; što kad se obriše

```bash
kubectl scale statefulset redis --replicas=5
kubectl get pvc          # data-redis-3, data-redis-4 automatski stvoreni

kubectl delete statefulset redis
kubectl get pvc          # PVC-ovi SU I DALJE TU
```

**Objašnjenje:** PVC-ovi iz `volumeClaimTemplates` se **namjerno NE brišu** brisanjem StatefulSeta ni skaliranjem prema dolje — podaci su vrijedni, i ponovno stvaranje StatefulSeta ponovno zakači iste PVC-ove na iste ordinale. Moraš ih obrisati ručno:
```bash
kubectl delete pvc -l app=redis
```
Novije verzije (v1.27+) imaju `spec.persistentVolumeClaimRetentionPolicy: {whenDeleted: Delete, whenScaled: Delete}`.

### LO4-17 · DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: agent
spec:
  selector:
    matchLabels: {app: agent}
  template:
    metadata:
      labels: {app: agent}
    spec:
      containers:
      - name: agent
        image: busybox
        command: ["sh","-c","while true; do echo agent alive; sleep 30; done"]
```

```bash
kubectl get daemonset
kubectl get pods -o wide -l app=agent    # točno 1 po nodeu
```

**Zašto točno jedan po nodeu:** DaemonSet controller ne prati `replicas` nego **popis nodova**. Za svaki node koji odgovara selektoru/tolerancijama stvori točno jedan pod; kad se doda novi node, automatski dobije pod; kad se node ukloni, pod nestane. **Nema `replicas` polja.** Za što: log agenti (Fluentd), monitoring (node-exporter), CNI/mrežni pluginovi, storage daemoni — sve što mora biti *na svakom stroju*.

### LO4-18 · Job

```bash
kubectl create job pi --image=perl:5.34 -- \
  perl -Mbignum=bpi -wle 'print bpi(2000)'

kubectl get jobs           # COMPLETIONS 1/1
kubectl logs job/pi
kubectl describe job pi
```

```yaml
apiVersion: batch/v1
kind: Job
metadata: {name: pi}
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 4
  template:
    spec:
      restartPolicy: Never        # OBAVEZNO Never ili OnFailure
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl","-Mbignum=bpi","-wle","print bpi(2000)"]
```

Job pokreće pod **do uspješnog završetka**; pod ostaje u stanju `Completed` da mu možeš pročitati logove.

### LO4-19 · CronJob + suspend + popis Jobova

```bash
kubectl create cronjob datecron --image=busybox \
  --schedule="*/1 * * * *" -- /bin/sh -c "date"

kubectl get cronjob
kubectl get jobs --watch                       # novi Job svake minute
kubectl logs job/<datecron-28...>

# SUSPEND
kubectl patch cronjob datecron -p '{"spec":{"suspend":true}}'
# ili: kubectl patch cronjob datecron --type=merge -p '{"spec":{"suspend":true}}'
kubectl get cronjob datecron       # SUSPEND: True
```

Ostala polja: `concurrencyPolicy: Allow|Forbid|Replace`, `successfulJobsHistoryLimit`, `failedJobsHistoryLimit`, `startingDeadlineSeconds`.

### LO4-20 · Ručno pokretanje Joba iz CronJoba

```bash
kubectl create job manual-run --from=cronjob/datecron
kubectl get jobs
kubectl logs job/manual-run
```
Kopira `jobTemplate` iz CronJoba → testiraš bez čekanja rasporeda i bez diranja rasporeda.

### LO4-8 · Requests i limits

```yaml
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

```bash
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].resources}' | python3 -m json.tool
kubectl describe pod <pod> | grep -A6 Limits
kubectl top pod        # stvarna potrošnja (treba metrics-server)
```

**Razlika (obavezno):**
- **requests** = koliko scheduler **rezervira** pri odabiru nodea. Utječe na rasporedbu i na QoS klasu.
- **limits** = tvrdi strop koji provodi cgroup. CPU preko limita → **throttling**. Memorija preko limita → **OOMKilled** (exit 137).
- QoS: `requests == limits` za sve → **Guaranteed**; postavljeni ali različiti → **Burstable**; ništa → **BestEffort** (prvi se izbacuje pod pritiskom).

`100m` = 0,1 CPU jezgre (millicores). `Mi` = mebibajt (1024²), `M` = megabajt (1000²).

---

## D · STORAGE, CONFIGMAP I SECRET

### LO4-23 · StorageClass, PV, PVC

```bash
kubectl get storageclass         # kratica: sc
kubectl get pv
kubectl get pvc
kubectl get pvc -A
kubectl describe pvc <name>
```

Na minikubeu default SC je `standard` (provisioner `k8s.io/minikube-hostpath`) → **dynamic provisioning**: čim stvoriš PVC, automatski nastane PV.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: mypvc}
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
  # storageClassName: standard
```

| Pojam | Što je |
|---|---|
| **StorageClass** | "recept" za dinamičko stvaranje storagea (koji provisioner, koji parametri, reclaim policy) |
| **PV** | stvarni komad storagea u clusteru (cluster-scoped) |
| **PVC** | zahtjev poda za storageom (namespaced); veže se na PV |
| **accessModes** | `ReadWriteOnce` (1 node), `ReadOnlyMany`, `ReadWriteMany` (više nodova), `ReadWriteOncePod` |
| **reclaimPolicy** | `Delete` (obriši PV s PVC-om) ili `Retain` (sačuvaj podatke) |

### LO4-22 · Multi-container Pod s dijeljenim `emptyDir`

```yaml
apiVersion: v1
kind: Pod
metadata: {name: shared}
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: writer
    image: busybox
    command: ["sh","-c","while true; do date >> /data/log.txt; sleep 5; done"]
    volumeMounts: [{name: shared-data, mountPath: /data}]
  - name: reader
    image: busybox
    command: ["sh","-c","sleep 3600"]
    volumeMounts: [{name: shared-data, mountPath: /data}]
```

```bash
kubectl exec shared -c reader -- cat /data/log.txt      # DOKAZ dijeljenja
```

`emptyDir` nastaje prazan kad se pod rasporedi na node i **briše se kad pod nestane** (ne kad se kontejner restarta). Postoji `emptyDir: {medium: Memory}` → tmpfs u RAM-u.

### LO4-30 · `sizeLimit` na `emptyDir`

```yaml
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 100Mi
```
Kad se prekorači, **kubelet evictira cijeli Pod** (`Evicted`, razlog `Usage of EmptyDir volume "cache" exceeds the limit`). Nije "disk full" greška u kontejneru nego izbacivanje poda. Za `medium: Memory` limit se broji u memory limit poda.

### LO4-28 · initContainer puni dijeljeni volume

```yaml
spec:
  volumes:
  - name: data
    emptyDir: {}
  initContainers:
  - name: init-data
    image: busybox
    command: ["sh","-c","echo 'pripremljeni podaci' > /data/seed.txt"]
    volumeMounts: [{name: data, mountPath: /data}]
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","cat /data/seed.txt && sleep 3600"]
    volumeMounts: [{name: data, mountPath: /data}]
```

initContaineri se izvode **serijski, prije** glavnih kontejnera i moraju **uspješno završiti** (exit 0). Za: preuzimanje konfiguracije, čekanje na bazu, migracije sheme, postavljanje dozvola.

### LO4-29 · `readOnly: true` na mountu

```yaml
        volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
```
```bash
kubectl exec <pod> -- sh -c "echo x > /etc/config/test"
# → sh: can't create /etc/config/test: Read-only file system
```
Sprječava da kompromitirana ili bugovita aplikacija promijeni konfiguraciju/podatke. Dio *least privilege* pristupa, uz `securityContext.readOnlyRootFilesystem: true`.

### LO4-24 · ConfigMap kao volume — svaki ključ postaje datoteka

```bash
kubectl create configmap appconfig \
  --from-literal=APP_MODE=production \
  --from-literal=LOG_LEVEL=debug
kubectl get cm appconfig -o yaml
```

```yaml
  volumes:
  - name: cfg
    configMap:
      name: appconfig
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","sleep 3600"]
    volumeMounts:
    - name: cfg
      mountPath: /etc/appconfig
```

```bash
kubectl exec <pod> -- ls /etc/appconfig       # APP_MODE  LOG_LEVEL
kubectl exec <pod> -- cat /etc/appconfig/APP_MODE   # production
```

**Prednost volumena nad env varijablama:** ConfigMap montiran kao volume se **automatski osvježava** (unutar ~1 min kubelet sync perioda) kad promijeniš ConfigMap. Env varijable se **ne** osvježavaju — treba restart poda.

### LO4-25 · `subPath` — jedan ključ na točnu putanju

```yaml
    volumeMounts:
    - name: cfg
      mountPath: /etc/nginx/nginx.conf     # PUNA putanja do DATOTEKE
      subPath: nginx.conf                  # ključ iz ConfigMapa
```

**Kad treba:** kad moraš ubaciti **jednu datoteku u direktorij koji već ima druge datoteke**. Bez `subPath` mount **prekrije cijeli direktorij** i obriše sve ostalo iz njega (npr. montiranjem na `/etc/nginx` izgubio bi `mime.types`).

**Caveat (bitno za bod):** mount sa `subPath` se **NE osvježava automatski** kad se ConfigMap promijeni. Treba restart poda (`kubectl rollout restart deployment/…`).

### LO4-31 · Secret iz literala; base64 ≠ enkripcija

```bash
kubectl create secret generic db-cred \
  --from-literal=username=appuser \
  --from-literal=password=S3cr3tPw

kubectl get secret db-cred -o yaml
#   username: YXBwdXNlcg==
#   password: UzNjcjN0UHc=

echo 'UzNjcjN0UHc=' | base64 -d          # → S3cr3tPw
kubectl get secret db-cred -o jsonpath='{.data.password}' | base64 -d
```

**Obavezna rečenica:** base64 je **kodiranje, ne enkripcija** — svatko s `get secrets` pravom ili pristupom etcd-u čita vrijednost u sekundi. Prava zaštita traži: **RBAC** (ograniči tko smije čitati secrete), **encryption at rest** u etcd-u (`EncryptionConfiguration`), i/ili vanjski store (HashiCorp Vault, Sealed Secrets, External Secrets Operator). Nikad ne commitaj Secret YAML u git.

### LO4-32 · Secret iz datoteka (`--from-file`)

```bash
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=example.com" -days 365

kubectl create secret generic certs --from-file=tls.crt --from-file=tls.key
kubectl get secret certs -o jsonpath='{.data}' | python3 -m json.tool
```
**Ključevi su imena datoteka:** `tls.crt` i `tls.key`. Preimenovanje: `--from-file=mycert=tls.crt`.

### LO4-33 · Typed TLS Secret

```bash
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key
kubectl get secret my-tls -o yaml      # type: kubernetes.io/tls
```
Ključevi su **fiksno** `tls.crt` i `tls.key`. **Gdje se koristi:** `Ingress.spec.tls[].secretName` (TLS terminacija na Ingress controlleru), OpenShift `Route`, mTLS u service meshu, webhook serveri. Tipizacija omogućuje validaciju i to da kontroleri znaju gdje tražiti cert/key.

### LO4-26 / LO4-34 · Secret kao volume + trade-offi

```yaml
  volumes:
  - name: sec
    secret:
      secretName: db-cred
      defaultMode: 0400        # samo owner čita
  containers:
  - name: app
    volumeMounts:
    - name: sec
      mountPath: /etc/secrets
      readOnly: true
```

```bash
kubectl exec <pod> -- ls -l /etc/secrets
kubectl exec <pod> -- cat /etc/secrets/password    # DEKODIRANA vrijednost
```

Datoteke sadrže **dekodirane** vrijednosti; kubelet montira **tmpfs (RAM)**, ne dira disk nodea.

| | **Volume** | **Env varijable** |
|---|---|---|
| Curi u `describe pod` / inspect | ne | **da** |
| Vidljivo u `/proc/<pid>/environ` | ne | **da** — svaki proces u kontejneru |
| Curi u crash dumpove i logove | rjeđe | **često** (mnoge biblioteke logiraju env) |
| Naslijeđeno u child procese | ne | **da** |
| Rotacija bez restarta | **da** (auto-osvježavanje) | ne |
| Dozvole datoteka | `defaultMode` | n/a |
| Jednostavnost | treba čitati datoteku | trivijalno |

**Zaključak: volume je sigurniji, env je praktičniji.** Za produkciju → volume.

### LO4-35 · `envFrom` + `secretRef` — svi ključevi odjednom

```yaml
    envFrom:
    - secretRef:
        name: db-cred
    - configMapRef:
        name: appconfig
```
```bash
kubectl exec <pod> -- env | grep -E 'username|password'
```
Ključevi postaju imena env varijabli **1:1** → moraju biti valjana imena env varijabli (`[A-Za-z_][A-Za-z0-9_]*`); nevaljani se tiho preskaču. Za pojedinačni ključ s preimenovanjem koristi `valueFrom`.

### LO4-37 · Selektivno montiranje jednog ključa

```yaml
  volumes:
  - name: sec
    secret:
      secretName: multi-secret
      items:
      - key: password          # samo ovaj ključ
        path: db_password      # pod ovim imenom datoteke
        mode: 0400
```
Rezultat: samo `/etc/secrets/db_password`, ostali ključevi se **ne** montiraju. *Least privilege* — kontejner vidi samo ono što mu treba.

Ista logika za pojedinačnu env varijablu:
```yaml
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-cred
          key: password
```

### LO4-36 · Image-pull Secret + `imagePullSecrets`
Vidi **LO4-2**. Potreban kad je image u **privatnom repozitoriju** (privatni Docker Hub repo, Quay privatni, korporativni Harbor/Nexus) ili kad želiš izbjeći anonimni rate limit. Bez njega: `ImagePullBackOff` s porukom `unauthorized: authentication required`.

---

## E · SERVICES I MREŽA

### Pregled tipova Servicea

| Tip | Dostupan odakle | Kako radi |
|---|---|---|
| **ClusterIP** (default) | samo unutar clustera | virtualni IP + DNS ime |
| **NodePort** | izvana, preko `<nodeIP>:30000-32767` | otvori isti port na **svakom** nodeu; sadrži ClusterIP |
| **LoadBalancer** | izvana, preko cloud LB-a | traži od clouda vanjski IP; sadrži NodePort + ClusterIP |
| **ExternalName** | — | CNAME na vanjski DNS, bez proxyja |
| **Headless** (`clusterIP: None`) | unutar clustera | **bez** VIP-a → DNS vraća IP-eve pojedinih podova |

### LO4-45 · `port` vs `targetPort` vs `nodePort` — KLJUČNO

```yaml
apiVersion: v1
kind: Service
metadata: {name: web}
spec:
  type: NodePort
  selector: {app: web}
  ports:
  - port: 80            # port SERVICEA (ClusterIP:80) — kako ga zovu drugi podovi
    targetPort: 8080    # port KONTEJNERA (containerPort) — kamo se promet šalje
    nodePort: 30080     # port NA NODEU (30000–32767) — vanjski pristup
```

Tok prometa: `klijent → nodeIP:30080 → ClusterIP:80 → podIP:8080`

Pamti: **`port` = ulaz u Service, `targetPort` = izlaz prema podu, `nodePort` = ulaz izvana.**
`targetPort` može biti **ime** porta iz pod specifikacije (`targetPort: http`).

### LO4-12 · `kubectl expose` — koji objekt nastaje i odakle selektor

```bash
kubectl expose deployment web --port=80 --target-port=80 --name=web-svc
kubectl get svc web-svc -o yaml
```

Nastaje **Service** tipa **ClusterIP** (default). **Selektor se kopira iz `deployment.spec.selector.matchLabels`** (`app: web`) → Service automatski pogađa podove tog Deploymenta.

```bash
kubectl expose deployment web --type=NodePort --port=80
kubectl expose deployment web --type=LoadBalancer --port=80
```

### LO4-38 · ClusterIP + DNS razlučivanje iz drugog poda

```bash
kubectl expose deployment web --port=80 --name=web-svc
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh
  nslookup web-svc
  nslookup web-svc.default.svc.cluster.local
  wget -qO- http://web-svc
```

**Puni FQDN:** `<service>.<namespace>.svc.cluster.local`
Kratko ime radi zbog `search` domena u `/etc/resolv.conf` poda:
```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10          # CoreDNS ClusterIP
```

### LO4-39 · NodePort na minikubeu

```bash
kubectl expose deployment web --type=NodePort --port=80
kubectl get svc web            # 80:31234/TCP

minikube service web --url     # → http://192.168.49.2:31234
curl $(minikube service web --url)

# ručno
echo "$(minikube ip):$(kubectl get svc web -o jsonpath='{.spec.ports[0].nodePort}')"
```

### LO4-40 · LoadBalancer i `<pending>` na minikubeu

```bash
kubectl expose deployment web --type=LoadBalancer --port=80
kubectl get svc web            # EXTERNAL-IP: <pending>

# u DRUGOM terminalu:
minikube tunnel                # traži sudo
kubectl get svc web            # EXTERNAL-IP: 10.96.x.x
curl http://<external-ip>
```

**Zašto `<pending>`:** tip LoadBalancer traži da **cloud controller manager** (AWS/GCP/Azure) provisionira stvarni vanjski load balancer i vrati IP. Minikube je lokalni cluster **bez cloud providera** → nitko ne popuni `status.loadBalancer.ingress` → ostaje `<pending>` zauvijek. `minikube tunnel` simulira LB: stvara mrežnu rutu na hostu i dodjeljuje IP. Alternativa na baremetalu: **MetalLB**.

### LO4-41 · Headless Service → per-pod A zapisi

```yaml
spec:
  clusterIP: None
  selector: {app: redis}
  ports: [{port: 6379}]
```
```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup redis
# vrati 3 IP-a (po jedan po podu), NE jedan VIP
nslookup redis-0.redis.default.svc.cluster.local
```
**Zašto:** klijent mora znati **koji točno** pod je koji (replikacija, sharding, leader election). Standardni Service bi sve sakrio iza jednog VIP-a i nasumično balansirao — beskorisno za bazu s masterom i replikama.

### LO4-42 · Endpoints / EndpointSlice

```bash
kubectl get endpoints web-svc
kubectl get endpointslices -l kubernetes.io/service-name=web-svc
kubectl describe endpoints web-svc
```

**Kako se popunjavaju:** *endpoints controller* prati Service i traži podove čije labele odgovaraju `service.spec.selector`, **u istom namespaceu**. U Endpoints upisuje samo podove koji su **Ready** (prošli readiness probe). `kube-proxy` (iptables/IPVS) iz toga generira pravila za balansiranje.

**Prazan `ENDPOINTS` = najčešći uzrok "Service ne radi"** → ili selektor ne odgovara labelama, ili nijedan pod nije Ready.

EndpointSlice je moderna zamjena za Endpoints — dijeli listu na komade od 100 (skalabilnost u velikim clusterima).

### LO4-43 · Cross-namespace pristup

```bash
kubectl create namespace prod
kubectl -n prod create deployment web --image=nginx
kubectl -n prod expose deployment web --port=80

kubectl run tmp --rm -it --image=busybox --restart=Never -- sh
  wget -qO- http://web                                  # ✗ ne radi (default ns)
  wget -qO- http://web.prod.svc.cluster.local           # ✓ radi
  wget -qO- http://web.prod                             # ✓ radi (search domene)
```
**Zašto kratko ime pada:** `search` lista poda počinje s **njegovim vlastitim** namespaceom. Servisi su **namespace-scoped**; kratko ime se razrješava lokalno. Iz drugog namespacea treba `<svc>.<ns>` ili puni FQDN.

### LO4-44 · `port-forward` na Service vs. na Pod

```bash
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward svc/web-svc 8080:80
kubectl port-forward deployment/web 8080:80
```

| | **na Pod** | **na Service** |
|---|---|---|
| Kamo ide | točno taj pod | kubectl **odabere jedan** pod iza servisa |
| Balansira | ne | **NE** — bira jedan i drži tunel na njemu |
| Preživi restart poda | ne | ne (tunel puca) |
| Za što | debug **konkretne** instance, čitanje stanja jednog poda | brzi pristup aplikaciji bez brige koji je pod |

**Bitno:** ni jedan ni drugi nisu load balancing — `port-forward` je **debug alat**, ne način izlaganja aplikacije. Tunel ide kroz API server.

### LO4-46 · Provjera konektivnosti iz debug poda

```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh
  nslookup web-svc
  wget -qO- http://web-svc:80
  nc -zv web-svc 80
  telnet web-svc 80
```
Bolji image s alatima: `nicolaka/netshoot`.
```bash
kubectl run netshoot --rm -it --image=nicolaka/netshoot --restart=Never -- bash
```

### LO4-47 · Jedan Service balansira preko DVA Deploymenta

```bash
kubectl create deployment v1 --image=nginx:1.25
kubectl create deployment v2 --image=nginx:1.27
# dodaj im ZAJEDNIČKU labelu
kubectl label deployment v1 tier=web
kubectl label deployment v2 tier=web
kubectl patch deployment v1 -p '{"spec":{"template":{"metadata":{"labels":{"tier":"web"}}}}}'
kubectl patch deployment v2 -p '{"spec":{"template":{"metadata":{"labels":{"tier":"web"}}}}}'
```
```yaml
apiVersion: v1
kind: Service
metadata: {name: web-all}
spec:
  selector:
    tier: web           # ← pogađa PODOVE OBA deploymenta
  ports: [{port: 80, targetPort: 80}]
```
```bash
kubectl get endpoints web-all      # IP-evi iz oba deploymenta
kubectl run tmp --rm -it --image=busybox --restart=Never -- \
  sh -c 'for i in $(seq 10); do wget -qO- http://web-all | grep -o "nginx/1..*"; done'
```
**Poanta:** Service ne zna ništa o Deploymentima — vidi **samo podove i njihove labele**. Ovo je mehanizam iza **blue/green** i **canary** deploymenta bez service mesha.

### LO4-49 · NodePort + provjera
```bash
kubectl expose deployment web --type=NodePort --port=80 --name=web-np
kubectl get svc web-np -o wide
curl $(minikube ip):$(kubectl get svc web-np -o jsonpath='{.spec.ports[0].nodePort}')
```

### LO4-48 · Sistematska dijagnostika "pod ne doseže Service"
Redoslijed (nauči napamet, ovo je i LO5-41):
```bash
# 1. DNS
kubectl run tmp --rm -it --image=busybox --restart=Never -- nslookup my-svc
kubectl -n kube-system get pods -l k8s-app=kube-dns          # radi li CoreDNS?
# 2. ENDPOINTS  ← najčešći krivac
kubectl get endpoints my-svc                                  # prazno?
# 3. SELEKTOR vs LABELE
kubectl get svc my-svc -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels
# 4. READINESS
kubectl get pods                       # READY 0/1 → pod nije u endpointsima
kubectl describe pod <pod> | grep -A10 Readiness
# 5. PORT
kubectl get svc my-svc -o yaml | grep -A3 ports    # targetPort == containerPort?
kubectl exec <pod> -- netstat -tlnp                # sluša li app stvarno taj port?
```

---

## F · PROBES (health checks)

| Probe | Pitanje na koje odgovara | Što se dogodi kad padne |
|---|---|---|
| **liveness** | Je li aplikacija **živa**? | kubelet **restarta kontejner** |
| **readiness** | Smije li primati **promet**? | pod se **izbaci iz Service Endpointsa** (ne restarta se) |
| **startup** | Je li se **podigla**? | kubelet ubije kontejner; **dok traje, liveness i readiness su onemogućeni** |

### LO4-50 · HTTP liveness probe

```yaml
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 1
          failureThreshold: 3
```
```bash
kubectl describe pod <pod> | grep -i liveness
# Liveness: http-get http://:80/ delay=5s timeout=1s period=10s #success=1 #failure=3
kubectl get pods       # RESTARTS raste ako probe pada
```
Uspjeh = HTTP status **200–399**.

### LO4-51 · TCP readiness probe za redis

```yaml
        readinessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 5
          periodSeconds: 10
```
```bash
kubectl describe pod <pod> | grep -i readiness
kubectl get pods                  # READY 1/1 tek kad probe prođe
kubectl get endpoints redis       # pod se pojavi tek kad je Ready
```
Treći tip: `exec: {command: ["redis-cli","ping"]}` — najfleksibilniji, ali najskuplji.

### LO4-52 · Startup probe za ~2-minutni boot

```yaml
        startupProbe:
          httpGet: {path: /healthz, port: 8080}
          failureThreshold: 30
          periodSeconds: 5
        livenessProbe:
          httpGet: {path: /healthz, port: 8080}
          periodSeconds: 10
          failureThreshold: 3
```

**Obrazloženje brojeva:** `failureThreshold × periodSeconds = 30 × 5 = 150 s` proračuna za start. To je ~2 min boota + 25 % rezerve za spor node ili hladan cache. `periodSeconds: 5` daje dovoljno gustu provjeru da aplikacija krene služiti promet čim je spremna (ne čeka se puni budžet).

**Zašto ne samo veliki `initialDelaySeconds` na livenessu:** tada bi liveness probe **uvijek** čekao 150 s prije prve provjere — i nakon što se app već odavno digla, i pri svakom restartu. Startup probe je "budžet samo za prvo podizanje": čim jednom prođe, prestaje i predaje kontrolu livenessu s kratkim, agresivnim intervalom. Bez njega spora aplikacija upada u **CrashLoopBackOff** jer je liveness ubije prije nego se digne.

### LO4-53 · Default ponašanje bez ijednog probea

- **Liveness**: default je da je kontejner "živ" **dok god glavni proces (PID 1) radi**. Kubernetes provjerava samo je li proces živ. Aplikacija koja je **deadlockana, zaglavljena ili vraća 500** ostaje "zdrava" zauvijek → **nikad se ne restarta**.
- **Readiness**: pod postaje `Ready` **čim su svi kontejneri Running** → odmah ulazi u Service Endpoints i **prima promet prije nego je aplikacija spremna** → prvi zahtjevi padaju s 502/connection refused.
- **Startup**: nema ga; liveness (ako postoji) kreće odmah nakon `initialDelaySeconds`.

**Zaključna rečenica za ispit:** bez probea Kubernetes vidi samo *proces*, ne *aplikaciju* — dobiva se lažan osjećaj zdravlja i prekidi usluge tijekom deploya.

---

## G · OPENSHIFT SPECIFIČNO (`oc`)

> DO180 se izvodi na OpenShiftu, a comprehensive review lab traži **ImageStream, Route i `oc set` naredbe** kojih u čistom Kubernetesu nema. Ovo je najčešća rupa u znanju na ispitu.

### Što `oc` dodaje iznad `kubectl`

`oc` je nadskup `kubectl`-a — svaka `kubectl` naredba radi i kao `oc`. Razlike:

| Pojam | Kubernetes | OpenShift |
|---|---|---|
| Izolacija | Namespace | **Project** (= namespace + anotacije + RBAC) |
| Vanjski pristup | Ingress + controller | **Route** (ugrađeni HAProxy router) |
| Reference na image | direktan string `repo/img:tag` | **ImageStream** — indirekcija |
| Build iz koda | vanjski CI | **BuildConfig / S2I** |
| Stariji workload | — | **DeploymentConfig** (naslijeđe; danas Deployment) |
| Sigurnost | opcionalna | **SCC** — non-root i random UID **po defaultu** |

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
oc new-project review
oc project                       # trenutni projekt
oc projects                      # svi
oc status                        # pregled projekta — što je deployano i kako je izloženo
oc get all
oc whoami ; oc whoami --show-console
```

### ImageStream — i zašto postoji

**ImageStream** je imenovani pokazivač na image u registryju. Workloadi referenciraju `mysql8:1`, a ne `registry.ocp4.example.com:8443/rhel9/mysql-80:1-228`.

```bash
oc create is mysql8
oc import-image mysql8:1 \
  --from=registry.ocp4.example.com:8443/rhel9/mysql-80:1-228 \
  --confirm

oc get is
oc describe is mysql8
oc get istag mysql8:1
```

**Tri razloga zašto (ovo je pitanje iz lab spec-a: „you need to isolate the workload from that change"):**
1. **Izolacija od promjene registryja** — ako se registry ili ime imagea promijeni, mijenja se **samo ImageStream**, nijedan manifest.
2. **Image triggeri** — deployment se može automatski redeployati kad ImageStream pokaže na novi image.
3. **Nepromjenjive reference** — OpenShift interno razrješava tag u **digest (`sha256:…`)**, pa svi podovi jedne revizije stvarno trče isti bit-po-bit image, čak i ako netko u međuvremenu prepiše tag u vanjskom registryju.

### Image trigger — automatski redeploy

```bash
# ime kontejnera unutar deploymenta:
oc get deploy quotesdb -o jsonpath='{.spec.template.spec.containers[0].name}'

oc set triggers deployment/quotesdb --from-image=mysql8:1 -c mysql-80
oc set triggers deployment/quotesdb            # ispis svih trigera
```
Kad `oc import-image` (ili automatski scheduled import) povuče novi image u `mysql8:1`, trigger prepiše `image:` polje u deploymentu → nastane nova revizija → rolling update. Bez trigera bi trebao ručni `oc set image`.

### Route — izlaganje van clustera

```bash
oc expose deployment frontend --port=8000      # 1) stvori SERVICE
oc expose service frontend \
  --hostname=frontend-review.apps.ocp4.example.com   # 2) stvori ROUTE

oc get route
oc get route frontend -o jsonpath='{.spec.host}'
curl http://frontend-review.apps.ocp4.example.com
```
Bez `--hostname` OpenShift sam generira `<route>-<project>.apps.<cluster-domain>`.
Za HTTPS: `oc create route edge --service=frontend` (vidi LO5-43 za usporedbu s Ingressom).

**Pazi na dvostruki `oc expose`:** prvi (`expose deployment`) stvara **Service**, drugi (`expose service`) stvara **Route**. Preskakanje prvog je česta greška.

### `oc set env` s prefiksom — jedan Secret, dvije aplikacije

```bash
oc create secret generic dbparams \
  --from-literal=user=operator1 \
  --from-literal=password=redhat123 \
  --from-literal=database=quotesdb

# baza očekuje MYSQL_USER, MYSQL_PASSWORD, MYSQL_DATABASE
oc set env deployment/quotesdb --from=secret/dbparams --prefix=MYSQL_

# frontend očekuje QUOTES_USER, QUOTES_PASSWORD, QUOTES_DATABASE
oc set env deployment/frontend --from=secret/dbparams --prefix=QUOTES_

oc set env deployment/frontend --list      # provjera
```
**Poanta:** `--prefix` uzme ključeve Secreta (`user`, `password`, `database`) i pred svaki zalijepi prefiks. Tako **jedan** Secret hrani dvije aplikacije koje očekuju različita imena varijabli — bez dupliciranja tajni. U čistom Kubernetesu to je `envFrom.prefix`:
```yaml
        envFrom:
        - secretRef: {name: dbparams}
          prefix: MYSQL_
```

### `oc set volumes` — PVC u jednoj naredbi

```bash
oc set volumes deployment/quotesdb --add \
  --name=quotesdb-storage \
  --type=persistentVolumeClaim \
  --claim-class=lvms-vg1 \
  --claim-size=2Gi \
  --claim-name=quotesdb-pvc \
  --mount-path=/var/lib/mysql

oc set volumes deployment/quotesdb           # popis
oc get pvc
```
Stvara PVC **i** montira ga u jednom koraku. U `kubectl`-u bi trebao zasebni PVC manifest + `volumes:` + `volumeMounts:` u deploymentu.

### Ostale korisne `oc set` naredbe

```bash
oc set image deployment/web nginx=nginx:1.27
oc set resources deployment/web --requests=cpu=100m,memory=128Mi --limits=cpu=500m,memory=256Mi
oc set probe deployment/web --readiness --get-url=http://:8080/healthz
oc set probe deployment/web --liveness  --get-url=http://:8080/healthz --initial-delay-seconds=10
oc set serviceaccount deployment/web myapp-sa
oc rollout status|history|undo deployment/web
oc scale deployment/web --replicas=3
oc logs -f deployment/web
oc rsh <pod>                     # = kubectl exec -it <pod> -- sh
oc port-forward svc/web 8080:8000
oc debug deployment/web          # kopija poda s shellom, bez probea
```

---

### RHA Comprehensive Review · Famous Quotes — kompletno rješenje

> DO180 Chapter 8, lab **`compreview-deploy`**. Predviđeno vrijeme: **35 minuta.** Vježbaj dok ne uđeš u to vrijeme.

**Aplikacija:** HTML frontend + MySQL 8 baza citata.

```bash
lab start compreview-deploy
cat /home/student/DO0018L/labs/compreview-deploy/resources.txt   # URL-ovi i imena imageova
oc login -u developer -p developer
```

#### 1 · Projekt
```bash
oc new-project review
```

#### 2 · ImageStream za bazu
```bash
oc create is mysql8
oc import-image mysql8:1 \
  --from=registry.ocp4.example.com:8443/rhel9/mysql-80:1-228 --confirm
oc get istag mysql8:1
```

#### 3 · Secret s parametrima baze
```bash
oc create secret generic dbparams \
  --from-literal=user=operator1 \
  --from-literal=password=redhat123 \
  --from-literal=database=quotesdb

oc get secret dbparams -o jsonpath='{.data}' | python3 -m json.tool
```

#### 4 · Deployment baze
```bash
oc create deployment quotesdb \
  --image=registry.ocp4.example.com:8443/rhel9/mysql-80:1-228

oc set env deployment/quotesdb --from=secret/dbparams --prefix=MYSQL_

oc set volumes deployment/quotesdb --add \
  --name=quotesdb-storage --type=persistentVolumeClaim \
  --claim-class=lvms-vg1 --claim-size=2Gi \
  --claim-name=quotesdb-pvc --mount-path=/var/lib/mysql

# automatski redeploy kad se promijeni mysql8:1
CNAME=$(oc get deploy quotesdb -o jsonpath='{.spec.template.spec.containers[0].name}')
oc set triggers deployment/quotesdb --from-image=mysql8:1 -c $CNAME

oc get pods -w
oc logs deployment/quotesdb
```

#### 5 · Service baze
```bash
oc expose deployment quotesdb --port=3306
oc get svc quotesdb
oc get endpoints quotesdb          # mora imati IP
```
Ime servisa **`quotesdb`** je ujedno DNS ime koje frontend dobiva kroz `QUOTES_HOSTNAME`.

#### 6 · Frontend
```bash
oc create deployment frontend \
  --image=registry.ocp4.example.com:8443/redhattraining/famous-quotes:2-42

oc set env deployment/frontend --from=secret/dbparams --prefix=QUOTES_
oc set env deployment/frontend QUOTES_HOSTNAME=quotesdb
oc set env deployment/frontend --list
```
Rezultat u podu: `QUOTES_USER=operator1`, `QUOTES_PASSWORD=redhat123`, `QUOTES_DATABASE=quotesdb`, `QUOTES_HOSTNAME=quotesdb`.

#### 7 · Service + Route
```bash
oc expose deployment frontend --port=8000
oc expose service frontend --hostname=frontend-review.apps.ocp4.example.com

oc get route
curl http://frontend-review.apps.ocp4.example.com
```

#### 8 · Test trigera (traži se u zadatku)
```bash
oc import-image mysql8:1 \
  --from=registry.ocp4.example.com:8443/rhel9/mysql-80:1-237 --confirm
oc rollout status deployment/quotesdb     # novi rollout kreće SAM
oc rollout history deployment/quotesdb
```

#### 9 · Ocjenjivanje
```bash
oc get all
oc status
lab grade compreview-deploy
lab finish compreview-deploy
```

#### Ono što lab NAMJERNO izostavlja (i što ispitivač voli pitati)
Lab sam upozorava: *„Before the resulting configuration is ready for production, you must configure probes and resource limits."* Ako te pitaju „što još nedostaje da ovo bude produkcijski spremno", odgovor je:

```bash
oc set probe deployment/frontend --readiness --get-url=http://:8000/ --initial-delay-seconds=5
oc set probe deployment/frontend --liveness  --get-url=http://:8000/ --period-seconds=10
oc set probe deployment/quotesdb --readiness --open-tcp=3306
oc set resources deployment/frontend --requests=cpu=100m,memory=128Mi --limits=cpu=500m,memory=512Mi
oc set resources deployment/quotesdb --requests=cpu=200m,memory=512Mi --limits=cpu=1,memory=1Gi
oc scale deployment/frontend --replicas=3
```
Plus: `revisionHistoryLimit`, `maxSurge/maxUnavailable`, NetworkPolicy da samo frontend smije na bazu, i **StatefulSet umjesto Deploymenta** za bazu ako ide više replika.

#### Mapa: koja LO4 pitanja ovaj lab pokriva

Prolazak kroz svih 53 praktična pitanja LO4 nasuprot poglavlju *Comprehensive Review* iz DO180. **Preklapanje je drukčije raspoređeno nego kod Beepera** — lab je OpenShift-specifičan, pa dio pitanja ne pokriva doslovno nego **ekvivalentno**: isti problem, druga primitiva. Baš to razlikovanje nosi bodove.

##### A · Identično — isti resurs, samo `oc` umjesto `kubectl`

| Pitanje | Što u labu odgovara | Zašto je izdvojeno |
|---|---|---|
| **LO4-31** · Secret iz literala, base64 nije enkripcija | *„MySQL database parameters that are stored in a secret named **dbparams**"* (user=operator1, password=redhat123, database=quotesdb) | `oc create secret generic` je bajt-za-bajt isti resurs kao `kubectl create secret generic`. Lab stvara točno onaj Secret koji pitanje traži, s tri ključa. Provjera base64 kodiranja (`oc get secret dbparams -o yaml`) je isti korak. |
| **LO4-35** · `envFrom` + `secretRef` — svi ključevi odjednom | *„…with the **MYSQL_ prefix** for each variable name"* → `oc set env --from=secret/dbparams --prefix=MYSQL_` | `--prefix` **jest** polje `envFrom[].prefix` iz Kubernetes API-ja — `oc` ga samo izlaže kao flag. Lab k tome radi ono što pitanje ne traži, a ispitivač voli: **isti Secret puni dvije aplikacije** kroz dva prefiksa (`MYSQL_` i `QUOTES_`). |
| **LO4-23** · StorageClass, PV i PVC | *„…no more than **2 GiB** … using `/var/lib/mysql` … and the **lvms-vg1** storage class"* | Ne možeš riješiti zadatak bez `oc get sc` da nađeš `lvms-vg1`, ni bez `oc get pvc/pv` da potvrdiš `Bound`. To je doslovno ono što pitanje traži da izlistaš, samo s konkretnim vrijednostima. |
| **LO4-12** · `kubectl expose` — koji objekt nastaje i odakle selektor | `oc expose deployment quotesdb --port=3306` i `oc expose deployment frontend --port=8000` | Identična naredba i identično ponašanje: nastaje **Service** tipa ClusterIP, a selektor se kopira iz `deployment.spec.selector.matchLabels`. Lab te tjera da to napraviš dvaput. |
| **LO4-38** · ClusterIP + razlučivanje po DNS-u iz drugog poda | `QUOTES_HOSTNAME=quotesdb` | Cijela konekcija frontenda na bazu ide preko **imena Servicea** razriješenog cluster DNS-om. To je razlog postojanja te varijable. Ako Service ne postoji ili se zove drukčije, aplikacija pada — pa je pitanje ugrađeno u kriterij prolaza. |
| **LO4-45** · `port` vs `targetPort` vs `nodePort` | *„The application service must listen on **port 8000**."* | Zahtjev fiksira **`port` Servicea** na 8000, a ti moraš provjeriti na kojem portu `famous-quotes:2-42` stvarno sluša i uskladiti `targetPort`. Ako pogodiš port napamet umjesto da ga provjeriš (`oc get pod -o yaml`, `oc logs`), Route vraća 504. Točno zamka iz pitanja. |
| **LO4-42** · Endpoints / EndpointSlice iz selektora | implicitno — bez popunjenih endpointa frontend ne dohvaća bazu, a Route vraća 503 | Lab to ne traži naredbom, ali je **prva provjera** kad ne radi. `oc get endpoints quotesdb` s praznim ispisom znači da selektor ne pogađa podove ili da nijedan pod nije Ready. Bez ovog koraka ne možeš debugirati lab. |
| **LO4-1** · imperativno stvaranje Deploymenta + izvoz YAML-a | `oc create deployment quotesdb --image=…` i `oc create deployment frontend --image=…` | Lab se rješava imperativno jer je **ograničen na 35 minuta** — pisanje manifesta od nule tu ne stane. Isti obrazac kao u pitanju: stvori naredbom, pa `oc get deploy -o yaml` ako trebaš manifest. |

##### B · Ekvivalentno — lab rješava isti problem OpenShift primitivom

Ovo je dio koji moraš znati **prevoditi u oba smjera**. Pitanje pita `kubectl` rješenje, lab traži `oc` rješenje istog problema.

| Pitanje | Lab to rješava ovako | Zašto je ekvivalentno |
|---|---|---|
| **LO4-2** · autentificirano povlačenje s Docker Huba (pull limit) · **LO4-36** · `docker-registry` Secret + `imagePullSecrets` | **ImageStream** `mysql8:1` — *„The database image name and its source registry might change in the future, **you need to isolate the workload from that change**."* | Oba odgovaraju na isto pitanje: **kako kontrolirati i stabilizirati odakle image dolazi**. Pitanje to rješava vjerodajnicama i pinanjem, lab indirekcijom kroz ImageStream (koji uz to interno razriješi tag u `sha256:` digest). Na ispitu moraš znati reći kad koje: pull secret kad je registry **privatan**, ImageStream kad se **ime ili lokacija imagea mijenja**. |
| **LO4-4** · rolling update · **LO4-5** · `rollout history` / `undo` / CHANGE-CAUSE | **image trigger** — *„The database must be **automatically deployed** whenever the source container in the `mysql8:1` resource changes"*, testirano s `mysql-80:1-237` | Trigger **ne zamjenjuje** rolling update nego ga **pokreće**. Ispod haube je isti mehanizam: novi pod template → novi ReplicaSet → postupna zamjena. Provjera je doslovno ista naredba — `oc rollout status/history deployment/quotesdb`. Razlika je samo tko je povukao okidač: ti (`set image`) ili ImageStream. |
| **LO4-39** · NodePort · **LO4-40** · LoadBalancer + `minikube tunnel` | **Route** — *„accessible from outside the cluster by using the `http://frontend-review.apps.ocp4.example.com` route"* | Objektiv laba glasi *„Expose applications to **clients outside the cluster**"* — identičan cilj kao oba pitanja. Route to daje jednom naredbom jer OpenShift ima ugrađen HAProxy router; na minikubeu isti rezultat postižeš NodePortom, LoadBalancerom uz `tunnel`, ili Ingressom. **Ovo je najveći prijevod koji moraš znati** i izravno se veže na LO5-43 (Route vs Ingress). |
| **LO4-26** · Secret kao volume · **LO4-34** · trade-offi volume vs env varijable | lab injektira secret **isključivo kao env varijable**, u oba workloada | Lab namjerno radi **manje sigurnu** od dvije opcije — vrijednosti završe u `oc describe`, u `/proc/<pid>/environ` i naslijede ih child procesi. Zato je savršen povod za pitanje: imaš gotovu konfiguraciju kojoj možeš nabrojati slabosti i pokazati kako bi je prepisao na montirani volume s `defaultMode: 0400`. |

##### C · Ono što lab izrijekom ostavlja nedovršenim

Lab sam upozorava: *„Before the resulting configuration is ready for production, you must configure **probes and resource limits**. Another comprehensive review exercise covers these subjects."* To upozorenje je praktički popis pitanja.

| Pitanje | Zašto je izdvojeno |
|---|---|
| **LO4-8** · requests i limits | Doslovno imenovano u upozorenju laba. Dodaješ s `oc set resources deployment/quotesdb --requests=… --limits=…` i provjeravaš u pod specu — isti korak kao u pitanju. |
| **LO4-50** · HTTP liveness probe | Frontend služi HTTP na portu iz zahtjeva → `oc set probe deployment/frontend --liveness --get-url=http://:8000/`. Ista struktura kao pitanje (nginx na 80), samo drugi port. |
| **LO4-51** · tcpSocket readiness probe | Pitanje traži redis na 6379; lab ima **MySQL na 3306** — identičan obrazac, jer za bazu nema smislenog HTTP endpointa pa je `tcpSocket` (ili `exec` s `mysqladmin ping`) jedini izbor. |
| **LO4-52** · startup probe s proračunom vremena | MySQL inicijalizacija data direktorija na praznom PVC-u traje 20–40 s. To je konkretan slučaj u kojem obrazlažeš `failureThreshold × periodSeconds` — s pravim brojevima iz laba, ne izmišljenima. |
| **LO4-53** · zadano ponašanje bez ijednog probea | Lab **jest** stanje „bez probea". Možeš uživo pokazati posljedicu: frontend postane `Ready` prije nego je MySQL spreman i prvi zahtjevi padnu. Najbolji mogući dokaz za to pitanje. |
| **LO4-3** · skaliranje na dva načina · **LO4-48** · sistematska dijagnostika „pod ne doseže Service" | Pokriva ih **drugi lab istog poglavlja** — *Troubleshoot and Scale Applications* (`compreview-scale`). U dokumentu je samo imenovan, bez specifikacije, ali mu je cijeli sadržaj upravo to. Detaljna rutina je u LO5 dijelu skripte. |

##### D · Što lab NE pokriva — vježbaj odvojeno

| Skupina | Pitanja | Zašto |
|---|---|---|
| Ostale vrste workloada | **LO4-15, 16, 17, 18, 19, 20, 21, 27, 41** | Lab koristi **samo Deploymente**. Nema golog Poda, StatefulSeta, DaemonSeta, Joba, CronJoba ni headless Servicea. |
| Više kontejnera po podu | **LO4-13, 22, 28** | Svaki pod u labu ima **jedan** kontejner — nema sidecara, dijeljenog `emptyDir`-a ni initContainera. |
| ConfigMap | **LO4-24, 25** | Lab koristi isključivo Secret. ConfigMap se ne pojavljuje, pa ni montiranje po ključevima ni `subPath`. |
| Secreti izvan literala | **LO4-32, 33, 37** | `dbparams` je stvoren samo `--from-literal`. Nema `--from-file`, nema `kubernetes.io/tls`, nema selektivnog montiranja jednog ključa. |
| Strategije updatea | **LO4-6, 7, 9, 10** | Lab koristi **zadanu** RollingUpdate strategiju i nikad je ne podešava — nema `maxSurge`/`maxUnavailable`, `Recreate`, `revisionHistoryLimit` ni zaglavljenog rollouta. |
| Mreža — ostalo | **LO4-43, 44, 46, 47, 49** | Sve je u jednom projektu `review` (nema cross-namespace FQDN-a), izlaganje ide Routeom (nema NodePorta ni `port-forward`), i nema dva Deploymenta iza jednog Servicea. |
| Ostalo | **LO4-11, 14** | Odnos selektora Deployment→RS→Pod je prisutan implicitno ali se ne vježba; `nodeSelector` se ne pojavljuje. |

##### Sažetak i najisplativiji sljedeći korak

Od 53 LO4 pitanja lab **izravno pokriva 8** (LO4-1, 12, 23, 31, 35, 38, 42, 45), **ekvivalentno rješava 8** (LO4-2, 4, 5, 26, 34, 36, 39, 40) i **imenuje kao nedovršeno 7** (LO4-3, 8, 48, 50, 51, 52, 53). Preostalih 30 se moraju vježbati zasebno — uglavnom su to **druge vrste workloada, ConfigMapi i strategije updatea**.

Redoslijed koji ti najviše vrijedi:

1. **Odradi `compreview-deploy` u zadanih 35 minuta.** Vrijeme je dio zadatka — ako ne staneš, gradivo ti još nije u prstima.
2. **Zatim mu dodaj ono na što sam lab upozorava** — probes i resource limite na oba workloada. Time u istoj vježbi pokriješ još pet pitanja (LO4-8, 50, 51, 52, 53).
3. **Onda cijeli lab prepiši u čisti `kubectl`** na minikubeu: ImageStream → pinani image po digestu, Route → Ingress ili NodePort, `oc set env --prefix` → `envFrom.prefix`, `oc set volumes` → zaseban PVC manifest. Ovaj korak je najvrjedniji jer te tjera da **prevedeš svaku OpenShift primitivu u Kubernetes ekvivalent** — a to je točno oblik u kojem su pitanja iz kategorije B postavljena.
4. **Na kraju odradi `compreview-scale`** za LO4-3 i LO4-48.

---

## H · ORKESTRATORI IZ PROJEKTA (Swarm, anti-affinity, Ingress, Helm)

> Projekt traži deploy na **dva** orkestratora — jedan jednostavan (Docker Swarm / ECS / Nomad / Cloud Run / Container Apps) i jedan složen (Kubernetes / OpenShift / EKS / AKS / GKE / Rancher) — s tri zahtjeva koja se ponavljaju u oba: **izloženo prema van preko load balancinga na portu 80**, **1 instanca baze i 2 instance aplikacije koje NISU na istom nodeu**, i **dokumentirano skaliranje na 3 replike**. Zadnja dva nisu pokrivena osnovnim gradivom, pa su ovdje.

---

### Docker Swarm — jednostavan orkestrator

Najbrži izbor za projekt: dolazi u samom Dockeru, nema instalacije, a compose datoteka se gotovo ne mijenja.

```bash
docker swarm init                                  # na manager nodeu
docker swarm join-token worker                     # ispiše naredbu za worker nodove
docker node ls                                     # popis nodova + uloge
docker node update --label-add zone=a <node-id>
```

#### `stack.yml`

```yaml
version: "3.9"

services:
  db:
    image: docker.io/library/mysql:8
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root
      MYSQL_DATABASE: tempconverter
      MYSQL_USER: tempuser
      MYSQL_PASSWORD_FILE: /run/secrets/db_pass
    volumes:
      - dbdata:/var/lib/mysql
    networks: [backend]
    secrets: [db_root, db_pass]
    deploy:
      replicas: 1                                  # ← JEDNA instanca baze
      placement:
        constraints: ["node.role == manager"]      # baza uvijek na istom nodeu (RWO volume)
      restart_policy:
        condition: on-failure

  app:
    image: quay.io/mislav/tempconverter:latest
    environment:
      DB_HOST: db
      DB_NAME: tempconverter
      DB_USER: tempuser
      DB_PASS_FILE: /run/secrets/db_pass
      STUDENT: "Mislav Uvanović"
      COLLEGE: "Algebra Bernays University"
    networks: [backend, frontend]
    secrets: [db_pass]
    deploy:
      replicas: 2                                  # ← DVIJE instance aplikacije
      placement:
        max_replicas_per_node: 1                   # ← NE na istom nodeu
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      restart_policy:
        condition: on-failure

  proxy:
    image: docker.io/library/nginx:alpine
    ports:
      - target: 80
        published: 80                              # ← izloženo na portu 80
        mode: ingress                              # routing mesh
    configs:
      - source: nginx_conf
        target: /etc/nginx/conf.d/default.conf
    networks: [frontend]
    depends_on: [app]
    deploy:
      replicas: 1

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    internal: true                                 # baza nema izlaz prema van

volumes:
  dbdata:

secrets:
  db_root:
    external: true
  db_pass:
    external: true

configs:
  nginx_conf:
    file: ./proxy.conf
```

```nginx
# proxy.conf
upstream tempconv { server app:5000; }
server {
    listen 80;
    location / {
        proxy_pass http://tempconv;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

#### Deploy i rad sa stackom

```bash
printf 'RootPw123' | docker secret create db_root -
printf 'TempPw123' | docker secret create db_pass -

docker stack deploy -c stack.yml tempconv
docker stack services tempconv
docker stack ps tempconv                    # koja replika je na kojem nodeu
docker service ps tempconv_app --no-trunc
docker service logs -f tempconv_app
```

#### Skaliranje na 3 replike (dokumentirati oba načina)

```bash
# imperativno
docker service scale tempconv_app=3

# deklarativno — promijeni replicas: 3 u stack.yml, pa
docker stack deploy -c stack.yml tempconv     # idempotentno, radi rolling update

docker service ps tempconv_app                # provjeri raspored po nodovima
```
**Oprez:** uz `max_replicas_per_node: 1` skaliranje na 3 traži **najmanje 3 nodea**. S manje nodova treće replika ostaje u stanju `Pending` s porukom *no suitable node*. Ako imaš samo 2 nodea, ili podigni `max_replicas_per_node: 2`, ili zamijeni tvrdo ograničenje mekim:
```yaml
      placement:
        preferences:
          - spread: node.id       # ravnomjerno raspoređuj, ali dopusti gomilanje
```

#### Zašto Swarm zadovoljava „load balancing na portu 80"

Swarm ima **ugrađen routing mesh**: port objavljen s `mode: ingress` otvara se na **svakom** nodeu u clusteru, a ulazni promet se preko IPVS-a L4 balansira na sve zdrave replike servisa — bez obzira na kojem nodeu su. Dodatno, `app` je i interno dostupan po imenu servisa `app` preko **VIP-a** koji balansira između replika. Zato u ovom stacku imaš dvije razine balansiranja: routing mesh na ulazu i VIP prema aplikaciji.

#### Swarm ↔ Kubernetes rječnik

| Swarm | Kubernetes |
|---|---|
| `service` | Deployment + Service |
| `task` | Pod |
| `stack` | skup manifesta / Helm chart |
| `docker service scale` | `kubectl scale` |
| routing mesh | NodePort + kube-proxy |
| overlay network | pod network (CNI) |
| `docker config` / `docker secret` | ConfigMap / Secret |
| `placement.constraints` | `nodeSelector` / `nodeAffinity` |
| `max_replicas_per_node` | `podAntiAffinity` / `topologySpreadConstraints` |
| `update_config` | `strategy.rollingUpdate` |

**Kad Swarm, kad Kubernetes:** Swarm za mali tim, jedan ili nekoliko hostova, gdje je compose datoteka već napisana i operativni trošak mora biti blizu nule. Kubernetes kad trebaš autoscaling, bogat ekosustav (operatori, service mesh, CSI storage), više timova na istom clusteru i podršku cloud providera. Swarm je u praksi u održavanju, ne u aktivnom razvoju — što je argument protiv njega za nove projekte.

---

### Anti-affinity — „replike ne smiju biti na istom nodeu"

Ovo je jedini dio projektnog zahtjeva za Kubernetes koji nije pokriven osnovnim gradivom.

#### Tvrdo pravilo (`required`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tempconverter
spec:
  replicas: 2
  selector:
    matchLabels: {app: tempconverter}
  template:
    metadata:
      labels: {app: tempconverter}
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels: {app: tempconverter}
              topologyKey: kubernetes.io/hostname     # ← "isti node" = ista vrijednost ove labele
      containers:
        - name: app
          image: quay.io/mislav/tempconverter:latest
          ports: [{containerPort: 5000}]
```

```bash
kubectl get pods -o wide -l app=tempconverter    # stupac NODE mora biti različit
kubectl describe pod <p> | grep -A5 Events
```

**Kako radi:** scheduler prije rasporeda provjeri postoji li već pod s labelom `app=tempconverter` na kandidatskom nodeu (uspoređujući vrijednost labele `kubernetes.io/hostname`). Ako postoji — node se odbacuje.

**Posljedica koju moraš znati:** `required` je tvrdo. Skaliranje na više replika nego što ima nodova ostavlja višak u **`Pending`** s eventom `didn't match pod anti-affinity rules`. Na **minikubeu s jednim nodeom** čak ni druga replika neće krenuti.

#### Meko pravilo (`preferred`) — za minikube

```yaml
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels: {app: tempconverter}
                topologyKey: kubernetes.io/hostname
```
Scheduler *pokušava* razdvojiti replike, ali ako ne može — svejedno ih rasporedi. Nema `Pending` podova.

#### Moderna alternativa: `topologySpreadConstraints`

```yaml
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule      # ili ScheduleAnyway (meko)
          labelSelector:
            matchLabels: {app: tempconverter}
```
`maxSkew: 1` = razlika u broju podova između najpunijeg i najpraznijeg nodea smije biti najviše 1 → ravnomjerno raspoređivanje. Čitljivije je od anti-affinityja i jeftinije za scheduler pri velikom broju podova.

**Zašto se ovo uopće traži:** dvije replike na istom nodeu ne štite ni od čega — pad tog jednog nodea obori obje. Razdvajanje je osnova **visoke dostupnosti**. `topologyKey` može biti i `topology.kubernetes.io/zone` → replike u različitim zonama dostupnosti, otporne i na pad cijelog data centra.

Za **minikube s više nodova**:
```bash
minikube start --nodes=3
kubectl get nodes
```

---

### Ingress — izlaganje na portu 80 s load balancingom

```bash
minikube addons enable ingress
kubectl -n ingress-nginx get pods
```

```yaml
apiVersion: v1
kind: Service
metadata: {name: tempconverter}
spec:
  selector: {app: tempconverter}
  ports:
    - port: 80
      targetPort: 5000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tempconverter
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: tempconverter.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: tempconverter
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
echo "$(minikube ip) tempconverter.local" | sudo tee -a /etc/hosts
curl http://tempconverter.local
```

**Dvije razine balansiranja, obje spomeni:** Ingress controller (nginx) balansira **HTTP** promet izvana prema Serviceu; Service preko `kube-proxy`/iptables balansira **TCP** promet na pojedine podove. Ingress radi na sloju 7 pa može routati po hostu i putanji, terminirati TLS i raditi sticky sessions — Service to ne može.

Na OpenShiftu isto postižeš s **Routeom** (`oc expose service tempconverter`) bez instaliranja ijednog controllera — vidi LO5-43.

---

### Helm — „template" u smislu projektnog zadatka

Projekt dopušta da „template" bude bash skripta, YAML ili **Helm chart**. Helm je najbliži onome što se u praksi zove template.

```bash
helm create tempconverter          # generira kostur charta
tree tempconverter
# Chart.yaml  values.yaml  templates/{deployment,service,ingress}.yaml  templates/_helpers.tpl
```

```yaml
# values.yaml
replicaCount: 2
image:
  repository: quay.io/mislav/tempconverter
  tag: latest
service:
  port: 80
  targetPort: 5000
ingress:
  enabled: true
  host: tempconverter.local
db:
  host: tempconverter-mysql
  name: tempconverter
  user: tempuser
```

```yaml
# templates/deployment.yaml (izvadak)
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: {{ .Chart.Name }}
              topologyKey: kubernetes.io/hostname
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          env:
            - name: DB_HOST
              value: {{ .Values.db.host | quote }}
            - name: DB_PASS
              valueFrom:
                secretKeyRef: {name: dbparams, key: password}
```

```bash
helm template ./tempconverter                  # renderiraj bez instalacije (za dokumentaciju)
helm install tempconv ./tempconverter
helm upgrade tempconv ./tempconverter --set replicaCount=3     # ← skaliranje na 3
helm list
helm history tempconv
helm rollback tempconv 1
helm uninstall tempconv
```

**Zašto Helm, a ne goli YAML:** jedan chart, više okolina — `-f values-dev.yaml` / `-f values-prod.yaml` mijenja replike, imagee i resurse bez diranja manifesta. Helm k tome vodi **povijest izdanja** s `rollback`, i pakira sve resurse aplikacije u jedan imenovani release koji se instalira i uklanja u komadu.

---

## ROKOVI · LO4 zadatak s lipanjskog roka

> Bilješka s roka, doslovno:
> *„podici neki deployment mysql deployment koristeci se sa mysql latest"*
> *„omoguciti neki internal connectivity(nesta unutar clustera)"*
> *„onda povezati neki pod sa portom kojeg mysql pruza"*
>
> U `exam rok kolokvij.docx` **nema** zabilježenih LO4 zadataka — kolokvij je pokrivao LO1, LO2 i LO3.

Tri rečenice = tri koraka: **Deployment → Service → klijentski pod koji stvarno izvrši upit.**

---

### Korak 1 · MySQL Deployment (`mysql:latest`)

```bash
kubectl create deployment mysql --image=docker.io/library/mysql:latest \
  --dry-run=client -o yaml > mysql-deployment.yaml
```
Pa dopuni `env` (generator ih ne zna dodati):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: docker.io/library/mysql:latest
          env:
            - {name: MYSQL_ROOT_PASSWORD, value: "root123"}
            - {name: MYSQL_DATABASE,      value: "ispit"}
            - {name: MYSQL_USER,          value: "student"}
            - {name: MYSQL_PASSWORD,      value: "student123"}
          ports:
            - containerPort: 3306
```

```bash
kubectl apply -f mysql-deployment.yaml
kubectl get pods -l app=mysql
kubectl logs deployment/mysql --tail=30 -f
```

**Čekaj u logu:** `ready for connections` uz **`port: 3306`**.

| Zamka | Objašnjenje |
|---|---|
| `1/1 Running` **nije** dokaz da baza prima upite | Bez readiness probe Kubernetes gleda samo je li proces živ. Inicijalizacija MySQL-a traje 20–40 s |
| Privremeni init server javlja spremnost na **`port: 0`** | To nije konačni server. Čekaj redak s 3306 |
| `root@localhost is created with an empty password` u logu | Poruka iz **faze inicijalizacije**; entrypoint zatim postavlja `MYSQL_ROOT_PASSWORD`. Nije dokaz da je root bez lozinke |
| `selector.matchLabels` mora odgovarati `template.metadata.labels` | Inače `apply` odbija manifest (vidi LO5-26) |
| `latest` je promjenjiv tag | Bilješka ga traži pa ga koristi — ali znaj da nije isto što i `mysql:8.0` |
| `containerPort: 3306` ne stvara Service | To je samo dokumentacija; pristup daje Service |

---

### Korak 2 · „Internal connectivity" = ClusterIP Service

Bilješka kaže *„nesta unutar clustera"* — to je **ClusterIP**, zadani tip Servicea: dostupan **samo unutar** clustera, bez izlaganja prema hostu.

```bash
kubectl expose deployment mysql --port=3306 --target-port=3306 --name=mysql
```
Ili deklarativno:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  type: ClusterIP
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
      protocol: TCP
```

```bash
kubectl apply -f mysql-service.yaml
kubectl get service mysql                      # CLUSTER-IP dodijeljen, EXTERNAL-IP <none>
kubectl get endpointslices -l kubernetes.io/service-name=mysql
```

**`endpointslices` je ključna provjera** — mora sadržavati IP MySQL poda i port 3306. Prazno = Service ne pogađa nijedan pod (selektor ili readiness), pa klijent nikad neće proći.

**Odgovori koje ispitivač očekuje:**
- Service bira podove po **labeli** (`app: mysql`), **ne** po imenu Deploymenta.
- Deployment i Service smiju imati isto ime — različite su vrste objekata.
- `port` = port Servicea (na njega se klijent spaja) · `targetPort` = port u podu · brojevi mogu biti različiti.
- ClusterIP **ne objavljuje** port na hostu — to bi bio NodePort ili LoadBalancer.

---

### Korak 3 · Klijentski pod se spaja na port koji MySQL nudi

```bash
kubectl run mysql-client --image=docker.io/library/mysql:latest \
  --restart=Never --rm -it --command -- \
  mysql -h mysql -P 3306 -u student -p ispit
```
Na `Enter password:` upiši `student123` (znakovi se ne prikazuju).

```sql
SELECT DATABASE();        -- → ispit
SELECT CURRENT_USER();    -- → student@%
SELECT 1 AS veza_radi;    -- → 1
exit;
```

| Dio naredbe | Značenje |
|---|---|
| `kubectl run` | stvara **goli Pod**, ne Deployment |
| `--restart=Never` | pod se ne restarta nakon završetka procesa |
| `--rm` | uklanja pod nakon izlaska iz sesije |
| `-it` | interaktivni terminal (za unos lozinke) |
| `--command --` | sve iza je naredba koja se pokreće u kontejneru |
| `-h mysql` | **DNS ime Servicea** — ne ime poda, ne `localhost` |
| `-P 3306` | **veliko P** = port |
| `-p` | **malo p** = traži lozinku interaktivno |
| `ispit` | ime baze |

**Najveća zamka:** unutar klijentskog poda `localhost` označava **taj klijentski pod**, ne bazu. Adresa mora biti ime Servicea.

Isti image sadrži i server i klijent — zato `mysql:latest` služi za oba poda i ne treba instalirati ništa na host.

**Ako zadatak traži da klijentski pod ostane kao resurs:** izostavi `--rm`. Uz `--restart=Never` pod nakon izlaska prelazi u `Completed`, ne ostaje `Running`.

---

### Provjera i dijagnostika ako ne radi

```bash
# redoslijed provjere (isti kao LO5-41)
kubectl get pods -l app=mysql                    # Running? READY 1/1?
kubectl logs deployment/mysql --tail=30          # ready for connections, port 3306?
kubectl get svc mysql
kubectl get endpoints mysql                      # PRAZNO = selektor/readiness problem
kubectl get svc mysql -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels                   # poklapaju li se?

# test iz jednokratnog poda, bez MySQL klijenta
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh
  nslookup mysql
  nc -zv mysql 3306
```

| Simptom | Uzrok |
|---|---|
| `Unknown MySQL server host 'mysql'` | Service ne postoji ili si u krivom namespaceu |
| `Can't connect ... (111) Connection refused` | Service postoji, ali baza još inicijalizira — čekaj `port: 3306` |
| `Access denied for user 'student'@'%'` | kriva lozinka, ili si promijenio env nad **postojećim** data direktorijem |
| `kubectl get endpoints` prazno | selektor Servicea ne odgovara labelama poda |

---

### Nadogradnje koje zadatak može tražiti povrh bilješke

Bilješka ne spominje trajnost ni probes, ali su prirodno sljedeće:

```bash
# trajna pohrana
kubectl set env deployment/mysql --list
kubectl explain deployment.spec.template.spec.volumes

# readiness probe za bazu (rješava „Running ali ne prima upite")
kubectl set probe deployment/mysql --readiness --open-tcp=3306 \
  --initial-delay-seconds=15 --period-seconds=10        # oc; u kubectl uredi manifest

# skaliranje i rollout
kubectl scale deployment mysql --replicas=1     # baza ostaje na 1 — RWO volume
kubectl rollout status deployment/mysql
```

**Ne skaliraj bazu na više replika** ako dijeli jedan `ReadWriteOnce` PVC — dvije instance nad istim data direktorijem znače korupciju. Za više replika treba **StatefulSet** s `volumeClaimTemplates` (vidi LO4-16).

---

### LO4 · Šalabahter

```bash
# GENERIRANJE
k create deploy web --image=nginx:1.25 --replicas=3 $do > web.yaml
k create job pi --image=perl -- perl -e 'print 1'  $do
k create cronjob c --image=busybox --schedule="*/1 * * * *" -- date  $do
k create cm  app --from-literal=K=V --from-file=f.conf  $do
k create secret generic s --from-literal=pw=x  $do
k create secret docker-registry r --docker-server=... --docker-username=... --docker-password=...
k create secret tls t --cert=tls.crt --key=tls.key
k expose deploy web --port=80 --target-port=8080 --type=NodePort  $do
k run tmp --rm -it --image=busybox --restart=Never -- sh

# UPDATE / ROLLBACK
k set image deploy/web nginx=nginx:1.27
k rollout status|history|undo deploy/web  [--to-revision=N]
k rollout restart deploy/web
k annotate deploy/web kubernetes.io/change-cause="..." --overwrite
k scale deploy web --replicas=5

# INSPEKCIJA
k get deploy,rs,po,svc,ep,pvc,cm,secret -o wide
k get po --show-labels ; k get po -l app=web
k describe po|deploy|svc <name>
k logs <po> [-c cnt] [--previous] [-f] [--tail=50]
k exec -it <po> [-c cnt] -- sh
k get po <po> -o jsonpath='{.status.containerStatuses[0].lastState}'
k explain deploy.spec.strategy --recursive
k api-resources

# MREŽA
k get ep <svc> ; k get endpointslices
k port-forward svc/web 8080:80
minikube service <svc> --url ; minikube tunnel ; minikube ip
# FQDN: <svc>.<ns>.svc.cluster.local ; pod StatefulSeta: <pod>.<svc>.<ns>.svc.cluster.local
```


---

# LO5 — Rješavanje problema u isporuci aplikacija kontejnerima

> **Format zadatka:** dobiješ pokvarenu naredbu, Containerfile ili pod u lošem stanju. Za svaki: **(1) identificiraj grešku, (2) popravi je, (3) objasni zašto je bila kriva.** Sva tri dijela nose bodove — nikad ne piši samo ispravnu naredbu.

---

### Univerzalni redoslijed dijagnostike

**Podman:**
```
podman ps -a          → u kojem je stanju? (Exited 0/1/125/137, Created)
podman logs <c>       → što je zadnje rekao prije smrti
podman inspect <c>    → .State.ExitCode, .State.OOMKilled, .Config.Cmd, portovi, mountovi
podman events         → lifecycle događaji uživo
```

**Kubernetes:**
```
kubectl get pods                      → STATUS + RESTARTS
kubectl describe pod <p>              → Events na dnu = 90 % odgovora
kubectl logs <p> [--previous]         → zašto je aplikacija pala
kubectl get events --sort-by=.lastTimestamp
```

#### Exit kodovi koje moraš prepoznati
| Kod | Značenje |
|---|---|
| `0` | uredan izlaz — proces je **završio posao** (često "nema long-running procesa") |
| `1` / `2` | greška aplikacije — čitaj logove |
| `125` | greška samog podmana (kriv flag) |
| `126` | naredba nije izvršiva |
| `127` | **naredba nije nađena** (kriv PATH, krivo ime binarija) |
| `137` | SIGKILL — **OOMKilled** ili `podman kill` |
| `139` | SIGSEGV |
| `143` | SIGTERM — uredan `stop` |

---

## A · GREŠKE U `podman run` NAREDBAMA

### LO5-1 · Zamijenjeni portovi
```bash
podman run -d --name web -p 80:8080 docker.io/library/nginx     # ✗
```
**Greška:** `-p` je **`HOST:CONTAINER`**, a ovdje je obrnuto. Nginx u kontejneru sluša na **80**, a naredba kaže "šalji promet s hosta:80 na kontejner:8080" — ondje nitko ne sluša → `curl localhost:80` vrati *empty reply / connection reset*.

**Popravak:**
```bash
podman run -d --name web -p 8080:80 docker.io/library/nginx
curl http://localhost:8080
```
**Objašnjenje:** desna strana mora odgovarati portu na kojem aplikacija **stvarno sluša unutar kontejnera** (provjeri s `podman inspect <img> --format '{{.Config.ExposedPorts}}'`). Lijeva strana je slobodan port na hostu.

### LO5-2 · `-e` bez vrijednosti
```bash
podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql   # ✗
podman logs db
# → You need to specify one of MYSQL_ROOT_PASSWORD, MYSQL_ALLOW_EMPTY_PASSWORD
#   and MYSQL_RANDOM_ROOT_PASSWORD
```
**Greška:** `-e VAR` bez `=vrijednost` znači "**preuzmi VAR iz okoline hosta**". Ako na hostu nije postavljen, varijabla se uopće ne prosljeđuje → mysql entrypoint skripta prekine inicijalizaciju i kontejner izađe.

**Popravak:**
```bash
podman run -d --name db -e MYSQL_ROOT_PASSWORD=StrongPw123 docker.io/library/mysql
# bolje — preko secreta:
printf 'StrongPw123' | podman secret create mysqlpw -
podman run -d --name db --secret mysqlpw,type=env,target=MYSQL_ROOT_PASSWORD docker.io/library/mysql
```

### LO5-3 · `--network host` + `-p`
```bash
podman run -d --name app --network host -p 8080:80 docker.io/library/nginx   # ✗
# WARNING: published ports are discarded when using host network mode
```
**Greška:** `--network host` znači da kontejner **nema vlastiti network namespace** — koristi hostov mrežni stack izravno. Nema granice preko koje bi se radio port forwarding, pa `-p` nema što mapirati i **tiho se ignorira**. Nginx je odmah dostupan na **hostovom portu 80**, ne na 8080.

**Popravak — odaberi jedno:**
```bash
podman run -d --name app --network host docker.io/library/nginx   # → localhost:80
# ILI
podman run -d --name app -p 8080:80 docker.io/library/nginx       # bridge + mapiranje
```
**Trade-off:** host mreža je brža (nema NAT-a) i rješava probleme s multicastom, ali gubi mrežnu izolaciju i može sukobiti portove s hostom.

### LO5-4 · Kontejner odmah `Exited (0)`
```bash
podman run -d --name c1 docker.io/library/busybox      # ✗ Exited (0)
```
**Greška:** kontejner živi **točno koliko i njegov PID 1**. Busybox default CMD je `sh`; bez `-it` nema priključenog TTY-a ni stdina, `sh` odmah pročita EOF i uredno izađe (exit 0). To **nije bug** — nema procesa koji bi držao kontejner živim.

**Popravak:**
```bash
podman run -d --name c1 docker.io/library/busybox sleep 3600      # long-running proces
podman run -d --name c1 docker.io/library/busybox sh -c "while true; do sleep 30; done"
podman run -it --name c1 docker.io/library/busybox sh             # interaktivno
```
**Pravilo:** kontejner treba **jedan proces u prvom planu (foreground)**. Zato `nginx -g "daemon off;"`, `httpd -D FOREGROUND` — nikad daemonizirani oblik.

### LO5-5 · `--rm` + `-d` gotcha
```bash
podman run --rm -d --name job docker.io/library/alpine echo hello
podman logs job
# → Error: no container with name or ID "job" found
```
**Greška:** `echo hello` završi u milisekundama, a `--rm` **odmah obriše kontejner** čim izađe. S `-d` se ne vidi ni izlaz ni greška — kontejner (i njegovi logovi) nestanu prije nego stigneš išta pročitati.

**Popravak:**
```bash
podman run --rm docker.io/library/alpine echo hello     # bez -d → izlaz na terminal
# ili zadrži kontejner da mu pročitaš logove:
podman run -d --name job docker.io/library/alpine echo hello
podman logs job
podman rm job
```
**Pravilo:** `--rm` koristi za **kratkotrajne zadatke koje gledaš u prvom planu**. `--rm -d` zajedno ima smisla samo za dugotrajne servise koje ne želiš čistiti ručno.

### LO5-6 · Premalen memory limit
```bash
podman run -d -p 8080:80 --memory 8m docker.io/library/mysql     # ✗
podman ps -a                          # Exited (137)
podman inspect db --format '{{.State.OOMKilled}}'   # true
```
**Greška:** **`--memory 8m`**. MySQL 8 traži minimalno ~**256–512 MB** (InnoDB buffer pool je sam po sebi 128 MB default). Kernel cgroup limiter ubije proces (SIGKILL → **exit 137**) čim prijeđe 8 MB, već tijekom inicijalizacije data direktorija.

**Popravak:**
```bash
podman run -d -p 3306:3306 --memory 512m \
  -e MYSQL_ROOT_PASSWORD=pw docker.io/library/mysql
podman stats --no-stream        # provjeri stvarnu potrošnju
```
**Bonus:** `-p 8080:80` je i sam kriv — mysql sluša na **3306**, ne 80.

### LO5-7 · Nema DNS-a na default mreži
```bash
podman run -d --name a docker.io/library/alpine sleep 1d
podman run -d --name b docker.io/library/alpine sleep 1d
podman exec a ping b            # ✗ ping: bad address 'b'
```
**Greška:** oba su na **default bridge mreži (`podman`)**, koja **nema DNS resolver**. Imena kontejnera se ne razrješavaju — samo IP-evi.

**Popravak — user-defined mreža:**
```bash
podman network create appnet
podman rm -f a b
podman run -d --name a --network appnet docker.io/library/alpine sleep 1d
podman run -d --name b --network appnet docker.io/library/alpine sleep 1d
podman exec a ping -c2 b        # ✓
```
Alternativa: stavi ih u **isti pod** (komuniciraju preko `localhost`) ili koristi `--add-host b:<ip>`.

**Objašnjenje:** na user-defined mrežama podman pokreće **aardvark-dns** i registrira `--name` i `--network-alias` kao DNS zapise. Compose to radi automatski jer sam stvara mrežu.

### LO5-8 · Bind mount bez SELinux oznake
```bash
podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx   # ✗
podman logs web       # Permission denied
```
**Greška:** nedostaje **SELinux mount flag**. Na RHEL/Fedori datoteke na hostu imaju kontekst `user_home_t`, a kontejnerski procesi trče kao `container_t` i smiju čitati samo `container_file_t` → **Permission denied** (ili direktorij izgleda prazan).

**Popravak:**
```bash
podman run -d --name web -v ./html:/usr/share/nginx/html:ro,Z -p 8080:80 docker.io/library/nginx
```
| Flag | Značenje |
|---|---|
| `:Z` | **privatna** oznaka — samo ovaj kontejner (preporuka) |
| `:z` | **dijeljena** oznaka — više kontejnera na istoj putanji |
| `:ro` | read-only |

**Oprez:** `:Z` **rekurzivno relabelira** direktorij na hostu — nikad ga ne stavljaj na `/home` ili `/usr`.
Druga česta zamka: **relativna putanja `./html`** — stariji podman traži apsolutnu (`$(pwd)/html`), inače je tretira kao **ime named volumea** i stvori prazan volume.

---

## B · GREŠKE U CONTAINERFILE-OVIMA

### LO5-10 · `apt-get install` bez `update`
```dockerfile
FROM debian:12
RUN apt-get install -y nginx        # ✗ E: Unable to locate package nginx
```
**Greška:** debian base image dolazi s **praznim APT cacheom** (`/var/lib/apt/lists` je obrisan da image bude manji). Bez `apt-get update` nema popisa paketa.

**Popravak (sve u JEDNOM sloju):**
```dockerfile
FROM debian:12
RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    rm -rf /var/lib/apt/lists/*
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
**Zašto isti sloj:** da se izbjegne **stale cache** — ako je `update` u zasebnom RUN-u, Docker ga cachira i sljedeći build instalira pakete iz zastarjelog indeksa (poznato kao *cache busting* problem).

### LO5-11 · Shell forma vs. exec forma
```dockerfile
FROM python:3.11
COPY app.py /app/app.py
WORKDIR /app
CMD python app.py                   # ✗ shell forma
```
**Greška:** shell forma se izvršava kao `/bin/sh -c "python app.py"` → **PID 1 je `sh`**, a python je njegovo dijete. `podman stop` šalje **SIGTERM PID-u 1**; `sh` signal **ne prosljeđuje** djetetu → aplikacija se ne ugasi uredno, čeka se 10 s grace period, pa SIGKILL. Gubiš graceful shutdown (nedovršeni zahtjevi, neispraznjeni bufferi, neotpuštene konekcije na bazu).

**Popravak:**
```dockerfile
CMD ["python", "app.py"]            # exec forma → python JE PID 1
```
| | shell forma `CMD cmd` | exec forma `CMD ["cmd"]` |
|---|---|---|
| PID 1 | `/bin/sh` | sama aplikacija |
| Signali | **ne prosljeđuju se** | idu izravno aplikaciji |
| Varijabla `$VAR` | ekspandira se | **NE** ekspandira |
| Treba shell u imageu | da (ne radi na `scratch`/distroless) | ne |

Ako ti treba shell ekspanzija u exec formi: `CMD ["sh","-c","exec python app.py $ARGS"]` (`exec` zamijeni sh procesom).

### LO5-12 · Loš redoslijed za layer caching
```dockerfile
FROM node:20
COPY . /app                    # ✗ svaka promjena koda invalidira sve ispod
WORKDIR /app
RUN npm install
```
**Greška:** `COPY . /app` kopira **cijeli** izvorni kod. Bilo koja promjena bilo koje datoteke mijenja checksum tog sloja → svi **sljedeći slojevi se invalidiraju** → `npm install` se vrti od nule pri svakom buildu (minutama).

**Popravak — od najmanje do najviše promjenjivog:**
```dockerfile
FROM node:20
WORKDIR /app
COPY package.json package-lock.json ./     # rijetko se mijenja
RUN npm ci --omit=dev                      # cachira se dok se ovisnosti ne promijene
COPY . .                                   # često se mijenja → zadnje
CMD ["node", "server.js"]
```
**Pravilo:** poredaj instrukcije od **najstabilnijih prema najpromjenjivijima**. Dodaj i `.containerignore` s `node_modules`, `.git`, `*.log`.

### LO5-13 · Prepisan `PATH`
```dockerfile
FROM alpine:3.20
ENV PATH=/app/bin                   # ✗
RUN apk add --no-cache curl
# → /bin/sh: apk: not found
```
**Greška:** `ENV PATH=/app/bin` **potpuno prepisuje** PATH umjesto da mu doda. Nestali su `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` → shell više ne nalazi **nijedan** sistemski binarij (`apk`, `sh`, `ls`, `curl`).

**Popravak:**
```dockerfile
FROM alpine:3.20
ENV PATH="/app/bin:${PATH}"         # dodaj, ne zamijeni
RUN apk add --no-cache curl
```
**Isto vrijedi za `LD_LIBRARY_PATH`, `PYTHONPATH`, `NODE_PATH`.** Uvijek `${VAR}` u vrijednosti.

### LO5-14 / LO5-15 · `EXPOSE` ≠ objavljivanje + neusklađen port
```dockerfile
FROM ubuntu:24.04
EXPOSE 8080                                       # ✗ dokumentira 8080
CMD ["python3", "-m", "http.server", "3000"]      # ✗ sluša 3000
```
```bash
podman run -d -p 8080:8080 myimg     # ništa ne odgovara
```
**Dvije greške:**
1. **Neusklađen port** — aplikacija sluša na **3000**, a mapiraš na kontejnerski **8080**. Ondje nitko ne sluša.
2. **`EXPOSE` ništa ne objavljuje** — to je **isključivo metapodatak/dokumentacija**. Ne otvara port, ne stvara firewall pravilo, ne radi mapiranje. Jedino što radi: `podman run -P` (veliko P) koristi EXPOSE popis da mapira portove na **nasumične** hostove portove, i `podman inspect` ga prikazuje.

**Popravak:**
```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends python3 \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /srv
EXPOSE 8080
CMD ["python3", "-m", "http.server", "8080"]
```
```bash
podman run -d -p 8080:8080 myimg
```
**Bonus greška:** `ubuntu:24.04` **nema python3 instaliran** → `exec: "python3": executable file not found`.

### LO5-16 · Ogroman image nakon `build-essential`
```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y build-essential     # ✗ ~800 MB
```
**Popravak — čišćenje u ISTOM sloju:**
```dockerfile
FROM debian:12
RUN apt-get update && \
    apt-get install -y --no-install-recommends build-essential && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```
**Zašto MORA biti isti sloj:** slojevi su **immutable i aditivni**. Ako u sljedećem `RUN`-u obrišeš datoteke, one i dalje **fizički postoje u prethodnom sloju** — samo su u gornjem sloju označene kao obrisane (whiteout). Ukupna veličina imagea se **ne smanjuje** ni za bajt.

**Još bolje — multi-stage:** compileraj u `build-essential` stageu, u finalni image kopiraj samo gotov binarij.

### LO5-17 · `USER` prije `COPY`/`RUN`
```dockerfile
FROM python:3.11
USER appuser                                    # ✗ korisnik ne postoji
COPY requirements.txt /app/requirements.txt     # ✗ /app ne postoji, nema prava
RUN pip install -r /app/requirements.txt        # ✗ Permission denied
```
**Tri greške:**
1. **`appuser` nikad nije stvoren** → `unable to find user appuser: no matching entries in passwd file`.
2. `COPY` upisuje kao **root vlasnik** po defaultu → neprivilegirani `appuser` ih poslije ne može mijenjati.
3. `pip install` bez `--user` piše u **`/usr/local/lib/python3.11/site-packages`** — vlasništvo roota → **Permission denied**.

**Popravak — instaliraj kao root, prebaci na non-root NA KRAJU:**
```dockerfile
FROM python:3.11
RUN useradd --create-home --uid 1001 appuser
WORKDIR /app
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt      # još kao root
COPY --chown=appuser:appuser . .
USER appuser                                            # ← ZADNJE
EXPOSE 8000
CMD ["python", "app.py"]
```
**Pravilo:** `USER` ide **što kasnije** — nakon svih instalacija. Runtime mora biti non-root (sigurnost), build-time treba root (instalacije).

### LO5-18 · Golang image ~1 GB → multi-stage
```dockerfile
FROM golang:1.22          # ✗ ~800 MB toolchaina ostaje u finalnom imageu
COPY . /src
WORKDIR /src
RUN go build -o app .
CMD ["/src/app"]
```
**Greška:** finalni image sadrži **cijeli Go toolchain, izvorni kod, module cache i git povijest** — a za pokretanje treba **samo jedan statički binarij**.

**Popravak:**
```dockerfile
# ---------- STAGE 1: build ----------
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app .

# ---------- STAGE 2: runtime ----------
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app /app
USER 65532:65532
EXPOSE 8080
ENTRYPOINT ["/app"]
```
**Rezultat: ~800 MB → ~10 MB.**

**Prednosti multi-stagea (napiši sve tri):**
1. **Veličina** — brži pull/push, manje troška u registryju, brži start poda.
2. **Sigurnost** — nema compilera, shella, package managera ni izvornog koda u produkciji → drastično manja napadna površina i manje CVE-ova.
3. **Nema curenja tajni** — build-time credentials (npr. za privatne module) ostaju u build stageu koji se odbacuje.

`CGO_ENABLED=0` daje statički binarij bez ovisnosti o libc → radi na `scratch`/distroless. `-ldflags="-s -w"` uklanja debug simbole.

---

## C · KUBERNETES TROUBLESHOOTING

### Mapa simptoma → uzrok

| Status poda | Najčešći uzroci | Prva naredba |
|---|---|---|
| `Pending` | nedovoljno resursa, nodeSelector/affinity/taint, nevezan PVC | `describe pod` → Events |
| `ImagePullBackOff` / `ErrImagePull` | typo u imenu/tagu, privatni registry bez secreta, rate limit | `describe pod` → Events |
| `CrashLoopBackOff` | app padne pri startu, nema long-running procesa, loša konfiguracija, liveness prerano | `logs --previous` |
| `Error` / `Completed` + restarti | proces uredno završi, a `restartPolicy: Always` | `logs` |
| `OOMKilled` u lastState | memory limit premalen ili memory leak | `get pod -o yaml` |
| `CreateContainerConfigError` | nepostojeći ConfigMap/Secret **ili ključ** | `describe pod` |
| `ContainerCreating` (zaglavljen) | mount PVC-a pada, CNI problem, image se sporo vuče | `describe pod` |
| `Running` ali `READY 0/1` | **readiness probe pada** | `describe pod` → Readiness |
| `Terminating` (zaglavljen) | finalizeri, node nedostupan, dug grace period | `get pod -o yaml` → finalizers |
| `Evicted` | node pod pritiskom (disk/memory), prekoračen emptyDir sizeLimit | `describe node` |

### LO5-19 · Pod zaglavljen u `Pending`
```bash
kubectl get pod <p>
kubectl describe pod <p> | tail -25          # ← Events
```
Tipične poruke i uzroci:

| Event poruka | Uzrok | Popravak |
|---|---|---|
| `0/1 nodes are available: 1 Insufficient cpu` | `requests` veći od kapaciteta nodea | smanji requests ili dodaj node |
| `didn't match Pod's node affinity/selector` | `nodeSelector`/`nodeAffinity` ne odgovara nijednom nodeu | `kubectl label node <n> k=v` ili makni selektor |
| `had untolerated taint {node-role...}` | node ima taint (npr. control-plane) | dodaj `tolerations` |
| `pod has unbound immediate PersistentVolumeClaims` | PVC nema PV / nema StorageClass | stvori PVC/PV, provjeri `kubectl get sc` |
| `node(s) had volume node affinity conflict` | PV je vezan na drugi node | uskladi zone/nodove |

```bash
kubectl get nodes -o wide
kubectl describe node minikube | grep -A8 "Allocated resources"
kubectl get pvc
kubectl get events --sort-by=.lastTimestamp | tail -20
```

### LO5-20 · Pokvarena indentacija u manifestu
```bash
kubectl apply -f broken.yaml
# error converting YAML to JSON: yaml: line 12: did not find expected key
# error: error validating "broken.yaml": ValidationError(Deployment.spec):
#   unknown field "template" in io.k8s.api.apps.v1.DeploymentSpec
```
**Uzrok:** YAML je **osjetljiv na razmake**; uklonjeni razmaci pomaknu ključ na krivu razinu ugniježđenja pa završi kao dijete pogrešnog roditelja (ili se dvije mape spoje).

**Dijagnostika i popravak:**
```bash
python3 -c "import yaml,sys; yaml.safe_load(open('broken.yaml'))"   # gdje puca
kubectl apply -f broken.yaml --dry-run=client                        # struktura
kubectl apply -f broken.yaml --dry-run=server                        # + shema API-ja
yamllint broken.yaml
```
**Pravila:** samo **razmaci, nikad tabovi**; 2 razmaka po razini; `-` liste uvučene ispod ključa; `spec.template.spec.containers` mora biti točno tako ugniježđen.

### LO5-21 · `ImagePullBackOff`
```bash
kubectl describe pod <p> | grep -A5 Events
```
| Poruka | Uzrok | Popravak |
|---|---|---|
| `manifest unknown` / `not found` | **typo u imenu ili tagu** | ispravi image; `podman pull` provjeri lokalno |
| `unauthorized: authentication required` | privatni registry bez pull secreta | `kubectl create secret docker-registry` + `imagePullSecrets` |
| `toomanyrequests` / `rate limit` | anonimni Docker Hub limit | autentificiraj se (LO4-2) |
| `no such host` / timeout | DNS/mreža/proxy na nodeu | provjeri konektivnost nodea |

```bash
kubectl set image deployment/web nginx=nginx:1.27         # popravi tag
kubectl rollout undo deployment/web                        # ili vrati unatrag
kubectl get pod <p> -o jsonpath='{.spec.containers[*].image}'
```
`BackOff` = kubelet eksponencijalno usporava pokušaje (10 s, 20 s, 40 s… do 5 min).

### LO5-22 · `CrashLoopBackOff`
```bash
kubectl logs <p> --previous          # ← logovi PRETHODNE, srušene instance
kubectl logs <p> -c <container> --previous
kubectl describe pod <p> | grep -A5 "Last State"
kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].lastState}'
```
`--previous` je ključ: trenutni kontejner je tek startao ili još ne postoji, pa `kubectl logs` bez njega vrati prazno ili beskorisno.

**Najčešći uzroci:** aplikacija baci exception pri startu (kriva konfiguracija, nedostupna baza) · nema long-running procesa (LO5-29) · liveness probe prekratak `initialDelaySeconds` pa ubija app tijekom bootanja · nedostaje env varijabla/secret · OOMKilled · krivi `command`/`args`.

**Trik za debug:** privremeno zamijeni command sa `sleep 3600`, uđi u pod i pokreni aplikaciju ručno da vidiš pravu grešku.

### LO5-23 · `OOMKilled`
```bash
kubectl get pod <p> -o yaml | grep -A5 lastState
#   lastState:
#     terminated:
#       exitCode: 137
#       reason: OOMKilled
kubectl describe pod <p> | grep -i -A3 "Last State"
kubectl top pod <p>
```
**Popravak — dvije opcije, obje spomeni:**
```yaml
        resources:
          requests: {memory: "256Mi"}
          limits:   {memory: "512Mi"}     # 1) podigni limit
```
2) **Popravi aplikaciju** — memory leak, prevelik JVM heap (`-Xmx` mora biti **ispod** k8s limita), prevelik connection pool, učitavanje cijelog dataseta u memoriju.

**Objašnjenje:** limit provodi **cgroup**, ne Kubernetes. Kernel OOM killer ubije proces sa SIGKILL (**137 = 128 + 9**). Kontejner se restarta prema `restartPolicy`; ako se ponavlja → `CrashLoopBackOff`. Slijepo dizanje limita samo odgađa problem ako je uzrok leak.

### LO5-24 · Podovi nikad ne postanu `Ready`
```bash
kubectl get pods                    # READY 0/1, STATUS Running
kubectl describe pod <p> | grep -A10 -i readiness
# Warning Unhealthy: Readiness probe failed: HTTP probe failed with statuscode: 404
kubectl logs <p>
kubectl exec <p> -- curl -sv localhost:8080/healthz     # testiraj probe RUČNO
```
**Uzroci:** kriva `path` (`/healthz` ne postoji → 404) · krivi `port` (probe na 80, app sluša na 8080) · `initialDelaySeconds` prekratak · `timeoutSeconds: 1` prekratak za spor endpoint · endpoint traži autentikaciju · aplikacija stvarno nije zdrava (ne može do baze).

**Posljedica:** pod **nije u Service Endpointsima** → Service nema kamo slati promet, a rolling update **zapne** jer novi podovi nikad ne postanu Ready.

### LO5-25 · Service ne vraća ništa → label mismatch
```bash
kubectl get endpoints my-svc
# NAME     ENDPOINTS   AGE
# my-svc   <none>      2m          ← PRAZNO = problem
kubectl get svc my-svc -o jsonpath='{.spec.selector}'      # {"app":"web"}
kubectl get pods --show-labels                              # app=webserver  ← ne poklapa se!
```
**Popravak:**
```bash
kubectl label pod <p> app=web --overwrite          # brzo, ali privremeno
# ISPRAVNO — uskladi u manifestu:
kubectl edit svc my-svc          # promijeni selector u app=webserver
kubectl get endpoints my-svc     # sada ima IP-eve
```
**Pravilo:** `service.spec.selector` mora **točno** odgovarati `pod.metadata.labels` (svi ključevi selektora moraju postojati na podu; pod smije imati i dodatne labele). Prazan `ENDPOINTS` ima samo dva uzroka: **selektor ne pogađa nijedan pod**, ili **nijedan pogođeni pod nije Ready**.

### LO5-26 · `selector.matchLabels` ≠ `template.metadata.labels`
```yaml
spec:
  selector:
    matchLabels: {app: web}
  template:
    metadata:
      labels: {app: nginx}        # ✗ ne poklapa se
```
```bash
kubectl apply -f bad.yaml
# The Deployment "web" is invalid: spec.template.metadata.labels:
#   Invalid value: map[string]string{"app":"nginx"}:
#   `selector` does not match template `labels`
```
**Zašto je greška fatalna:** Deployment bi stvorio podove koje **vlastiti selektor ne prepoznaje** → ReplicaSet ih ne bi brojao → mislio bi da ima 0 replika → stvarao bi nove u beskonačnost. API server to odbija pri validaciji.

**Popravak:** uskladi ih (template labele moraju biti **nadskup** selektora):
```yaml
  selector:
    matchLabels: {app: web}
  template:
    metadata:
      labels:
        app: web           # obavezno
        version: v1        # dodatne su OK
```
**Zapamti:** `spec.selector` je **immutable** — na postojećem Deploymentu ga ne možeš promijeniti, moraš obrisati i ponovno stvoriti.

### LO5-27 · `requests` veći od kapaciteta nodea
```yaml
        resources:
          requests:
            cpu: "16"
            memory: "64Gi"        # ✗ nijedan node nema toliko
```
```bash
kubectl describe pod <p>
# 0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.
kubectl describe node minikube | grep -A10 "Allocatable"
kubectl describe node minikube | grep -A8 "Allocated resources"
```
**Objašnjenje:** scheduler uspoređuje `requests` s **allocatable** kapacitetom nodea (kapacitet **minus** rezerve za kubelet/sistem), i to sa **zbrojem requestova već raspoređenih podova** — ne sa stvarnom trenutnom potrošnjom. Ako nijedan node nema dovoljno slobodnog *requestanog* prostora, pod ostaje `Pending` **zauvijek** (nema timeouta).

**Popravak:** smanji requests na realnu vrijednost (`kubectl top pod` daje stvarnu potrošnju kao orijentir), ili dodaj node / povećaj minikube VM:
```bash
minikube start --cpus=4 --memory=8192
```

### LO5-28 · Pod montira nepostojeći PVC
```bash
kubectl describe pod <p>
# Warning FailedMount: persistentvolumeclaim "mydata" not found
kubectl get pvc
```
**Popravak:**
```bash
kubectl create -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: mydata}
spec:
  accessModes: ["ReadWriteOnce"]
  resources: {requests: {storage: 1Gi}}
EOF
kubectl get pvc mydata        # STATUS: Bound
```
**Ako PVC postoji ali je `Pending`:**
```bash
kubectl describe pvc mydata
kubectl get storageclass       # postoji li ijedan? je li default?
```
Uzroci: nema (default) StorageClassa · nema slobodnog PV-a koji odgovara veličini/accessModeu · zatražen `ReadWriteMany` na provisioneru koji ga ne podržava.

**Ključno:** PVC je **namespace-scoped** — mora biti u **istom namespaceu** kao pod.

### LO5-29 · `ubuntu` bez long-running naredbe
```bash
kubectl run test --image=ubuntu
kubectl get pods       # Completed → CrashLoopBackOff
```
**Objašnjenje:** `ubuntu` image ima default CMD `/bin/bash`. Bez TTY-a bash odmah izađe (exit 0) → `Completed`. Ali podovi stvoreni preko `kubectl run`/Deploymenta imaju **`restartPolicy: Always`** → kubelet ga restarta, on opet izađe, restarta ga, opet izađe… → kubelet uvodi backoff → **`CrashLoopBackOff`** (iako aplikacija nije "pala", uredno je završila).

**Popravak:**
```bash
kubectl run test --image=ubuntu --command -- sleep 3600
kubectl run test --image=ubuntu -it --restart=Never -- bash
```
```yaml
      containers:
      - name: app
        image: ubuntu
        command: ["sleep"]
        args: ["3600"]
```
Za jednokratne poslove koristi **Job** (`restartPolicy: Never|OnFailure`), ne Deployment.

### LO5-30 · Triaža preko eventa
```bash
kubectl get events --sort-by=.lastTimestamp
kubectl get events --sort-by=.lastTimestamp -A | tail -30
kubectl get events --field-selector involvedObject.name=<pod>
kubectl events --for pod/<pod>            # noviji kubectl
kubectl get events -w                     # uživo
```
**Oprez:** eventi se čuvaju **samo ~1 sat** (`--event-ttl`). Za stariju povijest treba centralizirano logiranje. `describe` prikazuje samo evente vezane uz taj objekt — `get events` daje širu sliku (npr. scheduler + kubelet + controller zajedno).

### LO5-31 · Debug DNS-a iznutra
```bash
kubectl exec -it <pod> -- sh
  cat /etc/resolv.conf
  # nameserver 10.96.0.10
  # search default.svc.cluster.local svc.cluster.local cluster.local
  # options ndots:5
  nslookup my-svc
  nslookup my-svc.default.svc.cluster.local
  nslookup kubernetes.default
```
```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns       # radi li CoreDNS?
kubectl -n kube-system logs -l k8s-app=kube-dns
kubectl -n kube-system get svc kube-dns                    # mora biti 10.96.0.10
```
`ndots:5` znači: ime s manje od 5 točaka se prvo probava kroz **search** domene — zato `my-svc` postane `my-svc.default.svc.cluster.local`. To je i razlog zašto vanjske domene iz podova imaju 4–5 promašenih DNS upita prije uspjeha.

### LO5-32 · Ephemeral debug kontejner (distroless / bez shella)
```bash
kubectl debug -it <pod> --image=busybox --target=<container>
kubectl debug -it <pod> --image=nicolaka/netshoot --target=app -- bash
```
`--target` dijeli **process namespace** ciljnog kontejnera → vidiš njegove procese i `/proc/<pid>/root/...` filesystem.

Ostale varijante:
```bash
kubectl debug <pod> -it --copy-to=debug-copy --image=busybox      # kopija poda s dodatnim kontejnerom
kubectl debug node/<node> -it --image=busybox                     # debug samog nodea
```
**Zašto treba:** distroless/scratch imageovi nemaju `sh`, `ls`, `curl` → `kubectl exec` puca s `executable file not found`. Ephemeral kontejner ubaci alate **u postojeći pod bez restarta**. Ephemeral kontejneri se ne mogu ukloniti ni mijenjati i nemaju probes/resources.

### LO5-33 · Jednokratni pod za test konektivnosti
```bash
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh
  wget -qO- http://my-svc:80
  nc -zv my-svc 80
  nslookup my-svc
```
`--rm` = obriši nakon izlaska, `--restart=Never` = goli Pod (ne Deployment), `-it` = interaktivno. Ovo je najbrži način da odgovoriš na pitanje **"je li problem u aplikaciji ili u mreži?"** — testiraš iz *unutrašnjosti clustera*, mimo Ingressa i NodePorta.

### LO5-34 · Node `NotReady`
```bash
kubectl get nodes
kubectl describe node <node>          # ← Conditions
```
| Condition | Značenje |
|---|---|
| `Ready=False/Unknown` | kubelet ne javlja heartbeat |
| `MemoryPressure=True` | ponestaje RAM-a → evikcije |
| `DiskPressure=True` | pun disk / image cache |
| `PIDPressure=True` | previše procesa |
| `NetworkUnavailable=True` | CNI plugin nije spreman |

**Što provjeriti:**
```bash
# na minikubeu:
minikube ssh
  sudo systemctl status kubelet
  sudo journalctl -u kubelet -n 100 --no-pager
  df -h                     # DiskPressure?
  free -m                   # MemoryPressure?
  sudo crictl ps            # radi li container runtime?
minikube logs
minikube logs --problems

kubectl -n kube-system get pods -o wide     # CNI, kube-proxy, CoreDNS zdravi?
```
Kad je node `NotReady` duže od `pod-eviction-timeout` (5 min), controller označi podove za evikciju i preraspoređuje ih (ako imaju kontroler).

### LO5-35 · Pod zaglavljen u `Terminating`
```bash
kubectl get pod <p>                                  # Terminating 15m
kubectl get pod <p> -o yaml | grep -A5 finalizers
kubectl describe pod <p>
```
**Uzroci:**
1. **Finalizeri** — polje `metadata.finalizers` traži da neki kontroler odradi čišćenje (odspoji volume, obriše cloud resurs) prije brisanja. Ako je taj kontroler mrtav, pod visi zauvijek.
2. **Grace period** — aplikacija ignorira SIGTERM; čeka se `terminationGracePeriodSeconds` (default 30 s) do SIGKILL-a.
3. **Node nedostupan** — API server ne može potvrditi da je pod stvarno ugašen.
4. Zaglavljen `preStop` hook ili unmount volumena.

**Force delete (zadnja opcija):**
```bash
kubectl delete pod <p> --grace-period=0 --force
# ili ukloni finalizere:
kubectl patch pod <p> -p '{"metadata":{"finalizers":null}}' --type=merge
```
**Rizici koje MORAŠ navesti:** API server samo **obriše zapis iz etcd-a**, a proces može **i dalje raditi** na nodeu → "zombie" kontejner koji i dalje drži port, volume i piše u bazu. Za StatefulSet je posebno opasno: kontroler odmah stvori `pod-N` s **istim identitetom i istim PVC-om** → dvije instance pišu u isti storage → **korupcija podataka i split-brain**. Nikad ne force-deletaj StatefulSet pod dok nisi siguran da je stari proces mrtav.

### LO5-36 · Kriv par `apiVersion`/`kind`
```yaml
apiVersion: apps/v1
kind: Pod              # ✗
```
```bash
kubectl apply -f bad.yaml
# error: unable to recognize "bad.yaml": no matches for kind "Pod" in version "apps/v1"
```
**Popravak: `apiVersion: v1` + `kind: Pod`.**

| Kind | apiVersion |
|---|---|
| Pod, Service, ConfigMap, Secret, PVC, PV, Namespace, ServiceAccount, Node | **`v1`** (core grupa) |
| Deployment, ReplicaSet, StatefulSet, DaemonSet | **`apps/v1`** |
| Job, CronJob | **`batch/v1`** |
| Ingress, NetworkPolicy | **`networking.k8s.io/v1`** |
| Role, RoleBinding, ClusterRole, ClusterRoleBinding | **`rbac.authorization.k8s.io/v1`** |
| HorizontalPodAutoscaler | **`autoscaling/v2`** |

```bash
kubectl api-resources           # ← tablica svih parova, provjeri ovdje
kubectl explain pod             # pokazuje ispravan apiVersion
```

### LO5-37 · `CreateContainerConfigError` — nedostaje ključ ConfigMapa
```bash
kubectl get pod <p>                     # CreateContainerConfigError
kubectl describe pod <p>
# Error: couldn't find key LOG_LEVEL in ConfigMap default/appconfig
```
```yaml
        env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: appconfig
              key: LOG_LEVEL          # ✗ ključ ne postoji
```
**Dijagnostika i popravak:**
```bash
kubectl get cm appconfig -o jsonpath='{.data}' | python3 -m json.tool
kubectl edit cm appconfig            # dodaj ključ
# ILI označi ključ neobaveznim:
```
```yaml
            configMapKeyRef:
              name: appconfig
              key: LOG_LEVEL
              optional: true          # ← pod se digne i bez njega
```
**Razlika koju ispitivač voli čuti:** `CreateContainerConfigError` je greška **prije** pokretanja kontejnera (kubelet ne može sastaviti konfiguraciju) — nema logova jer proces nikad nije startao. `CrashLoopBackOff` je greška **nakon** starta.
Isti error nastaje i kad ConfigMap/Secret **uopće ne postoji** (`configmap "appconfig" not found`).

### LO5-38 · Secret iz drugog namespacea
```bash
kubectl describe pod <p>
# Warning FailedMount: secret "db-cred" not found
```
**Objašnjenje:** Secrets, ConfigMaps, PVC-ovi i ServiceAccounts su **namespaced resursi**. Pod **ne može** referencirati Secret iz drugog namespacea — to je temeljna sigurnosna granica Kubernetesa (inače bi bilo koji pod čitao produkcijske tajne).

**Popravak — kopiraj Secret u ciljni namespace:**
```bash
kubectl get secret db-cred -n prod -o yaml \
  | sed 's/namespace: prod/namespace: dev/' \
  | kubectl apply -n dev -f -

# ili ponovno stvori
kubectl create secret generic db-cred -n dev \
  --from-literal=password=... 
```
Za sustavnu sinkronizaciju: **External Secrets Operator**, **Reflector**, ili Sealed Secrets. (Ne postoji ugrađen "shared" namespace za secrete.)

Provjeri u kojem si namespaceu:
```bash
kubectl config view --minify -o jsonpath='{..namespace}'
kubectl config set-context --current --namespace=dev
```

### LO5-39 · `ProgressDeadlineExceeded`
```bash
kubectl rollout status deployment/web
# error: deployment "web" exceeded its progress deadline
kubectl describe deployment web
#   Progressing  False  ProgressDeadlineExceeded
kubectl get rs -l app=web
kubectl describe pod <novi-pod>      # ← PRAVI uzrok je OVDJE
```
**Objašnjenje:** `spec.progressDeadlineSeconds` (default **600 s**) je vrijeme unutar kojeg rollout mora napredovati. Ako novi podovi ne postanu Ready, Deployment dobije uvjet `Progressing=False`. **Deployment se NE vraća automatski unatrag** — samo prestane pokušavati; stari podovi i dalje služe promet.

**Pravi uzroci su uvijek na razini poda:** `ImagePullBackOff`, `CrashLoopBackOff`, readiness probe pada, `Pending` zbog resursa, nedostupan ConfigMap/Secret, prekoračena kvota.

**Rješenje:**
```bash
kubectl rollout undo deployment/web        # vrati se na zdravu reviziju
# popravi uzrok, pa ponovno deployaj
```

### LO5-40 · Uhvati typo PRIJE deploya
```bash
kubectl apply -f app.yaml --dry-run=client     # samo lokalna struktura
kubectl apply -f app.yaml --dry-run=server     # ← ŠALJE API SERVERU, validira shemu, ne sprema
kubectl apply -f app.yaml --validate=true
kubectl diff -f app.yaml                        # što bi se promijenilo u odnosu na cluster
```
**Zašto `--dry-run=server` hvata više:** klijentski dry-run provjerava samo je li YAML sintaksno ispravan i pretvoriv u JSON. **Serverski** ga provuče kroz **OpenAPI shemu API servera**, admission webhookove i defaulting → hvata **nepoznata polja i typo-e u imenima** (`imagePullpolicy` umjesto `imagePullPolicy`, `contaienrPort`, krivo ugniježđen `ports:`).

**Zamka:** `kubectl create/apply` **tiho ignorira** neka nepoznata polja u starijim verzijama (strategic merge patch) — pa aplikacija radi, ali tvoja postavka nema **nikakav** učinak. Zato uvijek `--dry-run=server`.

Vanjski alati: `kubeval`, `kubeconform`, `kube-linter`, `yamllint`.

### LO5-41 · "Aplikacija ne može do baze" — provjera po redu
```bash
# 1. RADI LI POD BAZE?
kubectl get pods -l app=db                       # Running? READY 1/1?
kubectl logs -l app=db --tail=30

# 2. POSTOJI LI SERVICE?
kubectl get svc db-svc
kubectl get svc db-svc -o jsonpath='{.spec.ports}'

# 3. IMA LI SERVICE ENDPOINTE?          ← NAJČEŠĆI KRIVAC
kubectl get endpoints db-svc                     # <none> = selektor/readiness problem
kubectl get svc db-svc -o jsonpath='{.spec.selector}'
kubectl get pods -l app=db --show-labels

# 4. RAZRJEŠAVA LI SE DNS?
kubectl exec -it <app-pod> -- nslookup db-svc
kubectl exec -it <app-pod> -- nslookup db-svc.default.svc.cluster.local

# 5. JE LI PORT DOBAR?
kubectl exec -it <app-pod> -- nc -zv db-svc 5432
kubectl exec -it <db-pod> -- netstat -tlnp        # sluša li baza stvarno na 5432?
kubectl get svc db-svc -o yaml | grep -A4 ports   # port vs targetPort

# 6. KONFIGURACIJA APLIKACIJE
kubectl exec <app-pod> -- env | grep -i 'DB_\|HOST\|PORT'
# → pokazuje li na "localhost" umjesto na "db-svc"? kriv namespace u FQDN-u?

# 7. NETWORKPOLICY?
kubectl get networkpolicy -A
```
**Najčešća tri krivca, po učestalosti:** (1) prazan `endpoints` zbog label mismatcha, (2) aplikacija konfigurirana na `localhost` umjesto na ime Servicea, (3) `targetPort` ne odgovara `containerPort`.

### LO5-42 · `restartCount` i `lastState` preko jsonpatha
```bash
kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# pregledno za sve podove:
kubectl get pods -o custom-columns=\
'NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount,\
REASON:.status.containerStatuses[0].lastState.terminated.reason,\
EXIT:.status.containerStatuses[0].lastState.terminated.exitCode'
```
**Čitanje rezultata:**
- `state` = **trenutno** stanje (`running` / `waiting` / `terminated`)
- `lastState` = stanje **prethodne** instance (postoji samo ako je bilo restarta)
- `reason: OOMKilled` + `exitCode: 137` → premalen memory limit
- `reason: Error` + `exitCode: 1` → aplikacija je pala → `logs --previous`
- `reason: Completed` + `exitCode: 0` → nema long-running procesa (LO5-29)
- visok `restartCount` sa stabilnim `Running` → povremeni liveness failovi

### LO5-43 · OpenShift **Route** vs Kubernetes **Ingress**

| | **Ingress** (Kubernetes) | **Route** (OpenShift) |
|---|---|---|
| API grupa | `networking.k8s.io/v1` | `route.openshift.io/v1` |
| Treba controller? | **DA** — sam po sebi ne radi ništa (nginx-ingress, Traefik, HAProxy…) | **NE** — ugrađeni HAProxy router dolazi s platformom |
| Standard | CNCF standard, prenosiv | Red Hat specifično |
| TLS načini | `edge` (terminacija na LB-u) preko `spec.tls` | **`edge`, `passthrough`, `re-encrypt`** |
| Automatski hostname | ne — moraš navesti | **da** — `<name>-<ns>.apps.<cluster-domain>` |
| Traffic splitting / canary | preko anotacija specifičnih za controller | **ugrađeno** — `alternateBackends` s težinama |
| Cilj (backend) | Service | Service (ili izravno) |
| Sticky sessions, rate limit | anotacije po controlleru | anotacije `haproxy.router.openshift.io/*` |

```yaml
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: web}
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: {name: web-svc, port: {number: 80}}
---
# OpenShift Route
apiVersion: route.openshift.io/v1
kind: Route
metadata: {name: web}
spec:
  host: app.apps.cluster.example.com    # opcionalno — generira se automatski
  to:
    kind: Service
    name: web-svc
  port:
    targetPort: 8080
  tls:
    termination: edge
```
```bash
oc expose service web-svc          # ← jednom naredbom stvori Route
oc get routes
```
**Zaključak za ispit:** Route je nastao **prije** nego je Ingress postao stabilan i nudi bogatije TLS opcije (posebno `passthrough` i `re-encrypt`, koje Ingress standardno nema) uz nula konfiguracije. Ingress je **prenosiv** između svih distribucija Kubernetesa. OpenShift 4 podržava oba — kad stvoriš Ingress, OpenShift ga **automatski konvertira u Route** iza kulisa. Moderna zamjena za oboje je **Gateway API**.

---

## D · COMPREHENSIVE REVIEW: TROUBLESHOOT AND SCALE

> DO180 Chapter 8 ima **dva** lab zadatka: `compreview-deploy` (obrađen u LO4) i **`compreview-scale`** — *Troubleshoot and Scale Applications*. Drugi je doslovno LO5. Ovo je popis svega što taj lab traži, u obliku u kojem ga možeš odraditi na ispitu.

```bash
lab start compreview-scale
oc login -u developer -p developer
oc project <projekt-iz-zadatka>
```

### Rutina od šest koraka za „aplikacija ne radi, popravi je"

```bash
# 1. ŠTO JE SLOMLJENO
oc get all
oc status                         # OpenShift odmah javi "pod is crash-looping" i slično
oc get pods -o wide

# 2. ZAŠTO
oc describe pod <p> | tail -25    # Events
oc logs <p> --previous
oc get events --sort-by=.lastTimestamp | tail -20

# 3. JE LI KONFIGURACIJA TU
oc get secret,cm
oc set env deployment/<d> --list
oc set volumes deployment/<d>

# 4. JE LI MREŽA SPOJENA
oc get svc,endpoints,route
oc get endpoints <svc>            # prazno = label/readiness problem

# 5. POPRAVI
oc set env|image|resources|probe|volumes deployment/<d> ...
oc edit deployment/<d>

# 6. POTVRDI
oc rollout status deployment/<d>
oc get pods
curl http://<route-host>
```

### Tipične podmetnute greške u tom labu

| Simptom | Gdje gledati | Popravak |
|---|---|---|
| `CrashLoopBackOff` odmah po startu | `oc logs --previous` → *"unknown variable"* / *"access denied"* | nedostaje env varijabla ili je krivi prefiks — `oc set env --from=secret/... --prefix=...` |
| `CreateContainerConfigError` | `describe` → *couldn't find key* | krivo ime ključa u Secretu/ConfigMapu |
| `ImagePullBackOff` | `describe` → *manifest unknown* | typo u tagu; ako je ImageStream — `oc import-image ... --confirm` |
| Pod `Running`, `READY 0/1` | `describe` → *Readiness probe failed* | kriv `path` ili `port` u probeu → `oc set probe --readiness --get-url=http://:8080/` |
| Route vraća **503** | `oc get endpoints <svc>` prazno | Service selektor ne pogađa podove **ili** nijedan pod nije Ready |
| Route vraća **504 / timeout** | `oc get route -o yaml` → `targetPort` | Service `targetPort` ≠ port na kojem app sluša |
| Pod `Pending` | `describe` → *Insufficient* / *unbound PVC* | smanji `requests` ili stvori PVC |
| `OOMKilled`, restarti rastu | `oc get pod -o yaml` → `lastState` | podigni `memory` limit s `oc set resources` |
| Skaliranje ne radi | `oc get rs` | `replicas` promijenjen na RS-u umjesto na Deploymentu — Deployment ga vrati natrag |

### Skaliranje — što se traži

```bash
oc scale deployment/frontend --replicas=3
oc get pods -l app=frontend -o wide
oc get rs

# provjeri da Route stvarno balansira preko svih replika
for i in $(seq 6); do curl -s http://<route-host> | grep -o 'pod-[a-z0-9]*'; done
```

**Preduvjeti da skaliranje uopće ima smisla — spomeni ih:**
1. **Readiness probe** — bez njega novi podovi primaju promet prije nego su spremni, pa dio zahtjeva pada.
2. **Requests/limits** — bez `requests` scheduler ne zna koliko mjesta treba i može pretrpati node; bez `limits` jedan pod izgladni ostale.
3. **Stateless aplikacija** — replike ne smiju dijeliti `ReadWriteOnce` volume; sesije moraju biti u bazi/cacheu, ne u memoriji poda (vidi LO3-22).
4. **`maxSurge`/`maxUnavailable`** — da update triju replika ne ostavi nula spremnih.

Automatsko skaliranje (ako pitaju):
```bash
oc autoscale deployment/frontend --min=2 --max=6 --cpu-percent=70
oc get hpa
```
**HPA traži postavljen `resources.requests.cpu`** — bez njega nema baze za postotak i HPA ostaje na `<unknown>/70%`.

### Popravak probea `oc`-om, bez ručnog editiranja YAML-a

```bash
oc set probe deployment/frontend --readiness --get-url=http://:8000/ \
  --initial-delay-seconds=5 --period-seconds=10 --failure-threshold=3
oc set probe deployment/frontend --liveness --get-url=http://:8000/ \
  --initial-delay-seconds=15 --period-seconds=10
oc set probe deployment/quotesdb --readiness --open-tcp=3306
oc set probe deployment/frontend --remove --liveness      # ukloni pokvaren probe
```
`--get-url=http://:8000/` — **prazan host je namjeran**, znači „sam pod". Ako upišeš hostname, probe ide na krivu adresu i uvijek pada.

### Zadnji korak koji studenti zaboravljaju
```bash
lab grade compreview-scale        # ispisuje i hintove što još fali
lab finish compreview-scale
```

---

### Mapa: koja LO5 pitanja pokriva Comprehensive Review 2

Prolazak kroz svih 43 praktična pitanja LO5 nasuprot poglavlju *Comprehensive Review* iz DO180. Poglavlje ima **dva** laba — `compreview-deploy` (Famous Quotes) i `compreview-scale` (Troubleshoot and Scale). Prvi ima punu specifikaciju, drugom je u dokumentu naveden **samo naslov**, pa je njegov sadržaj izveden iz tog naslova i iz upozorenja prvog laba. Gdje je tako, izrijekom je označeno.

**Ključna razlika u odnosu na LO3 i LO4 mape:** ovdje se ne gleda „koji korak laba odgovara kojem pitanju", nego **koji kvar nastaje dok radiš lab**. Comprehensive Review 2 je čisti Kubernetes/OpenShift, pa **cijela podman polovica LO5 pitanja otpada.**

#### A · Kvarovi koje ćeš stvarno proizvesti radeći `compreview-deploy`

| Pitanje | Kako nastaje u labu | Zašto je izdvojeno |
|---|---|---|
| **LO5-41** · „aplikacija ne može do baze" — provjera redom pod → Service → endpoints → DNS → port | `QUOTES_HOSTNAME=quotesdb` mora se razriješiti u Service `quotesdb` | **Najjače preklapanje od svih 43 pitanja.** Lab je doslovno taj scenarij: frontend govori s bazom isključivo preko imena Servicea. Ako bilo koja karika pukne, aplikacija ne radi — a redoslijed provjere iz pitanja je točna procedura za ovaj lab. |
| **LO5-25** · Service ne vraća ništa, label mismatch, `get endpoints` | Route iznad Servicea bez endpointa vraća **503** | Lab stvara dva Servicea s `oc expose deployment`. Ako Service napišeš ručno ili preimenuješ deployment, selektor promaši podove i `oc get endpoints quotesdb` ostane prazan. Route to pretvara u 503, što je prvi simptom koji vidiš u browseru. |
| **LO5-37** · `CreateContainerConfigError` — nedostaje ključ | Secret `dbparams` ima ključeve `user`, `password`, `database`; injektiraju se s dva različita prefiksa | Najlakše mjesto za grešku u cijelom labu. Krivo napisan ključ (`passwd` umjesto `password`) ili krivi `--prefix` daju točno ovaj status. Bitno: kontejner **nikad ne starta**, pa `oc logs` je prazan — moraš u `describe`. |
| **LO5-22** · `CrashLoopBackOff` + `logs --previous` | MySQL pada pri inicijalizaciji ako env varijable nisu ispravno postavljene | Ako `--prefix=MYSQL_` izostaviš ili pogriješiš, mysql-80 image ne dobije parametre i entrypoint prekine inicijalizaciju. Kontejner se restarta u petlji, a jedini način da vidiš razlog je `--previous` — jer trenutna instanca još nije ništa ispisala. |
| **LO5-28** · pod montira PVC koji ne postoji / PVC `Pending` | Zahtjev traži **2 GiB** i storage class **`lvms-vg1`** | Krivo ime storage classa (ili izostavljen `--claim-class`) ostavi PVC u `Pending`, a pod zaglavi u `ContainerCreating` s `FailedMount`. Zato lab i traži da prvo pogledaš `oc get sc` — što je istovremeno LO4-23. |
| **LO5-21** · `ImagePullBackOff` | ImageStream `mysql8:1` + image trigger | Tri načina da se dogodi baš ovdje: `oc import-image` bez `--confirm`, krivo napisan istag u `--from-image`, ili trigger koji pokazuje na tag koji ne postoji. Test korak s `mysql-80:1-237` je dodatna prilika da pukne. |
| **LO5-39** · rollout zaglavljen s `ProgressDeadlineExceeded` | image trigger pokreće rollout automatski | Zahtjev *„must be automatically deployed whenever the source container changes"* znači da rollout kreće **bez tvoje naredbe**. Ako novi image ne može krenuti, deployment tiho visi 10 minuta pa prijavi ovaj uvjet. Pravi uzrok je uvijek na razini poda — točno kako pitanje traži. |
| **LO5-24** · podovi nikad ne postanu `Ready` | nakon što dodaš probes koje lab traži za produkciju | Lab upozorava da moraš dodati probes. Čim ih dodaš, kriv `path` ili `port` daje `Running` ali `READY 0/1`, pod ispada iz endpointa i Route vraća 503. Prirodni nastavak laba. |
| **LO5-43** · OpenShift **Route** vs **Ingress** | *„accessible from outside the cluster by using the `http://frontend-review.apps.ocp4.example.com` **route**"* | Jedino LO5 pitanje koje je eksplicitno OpenShift, i lab ga rješava u cijelosti. Odradiš li lab, imaš gotov praktični dio odgovora; ostaje samo naučiti razlike u TLS načinima i to da Ingress treba controller, a Route ne. |

#### B · Drugi lab poglavlja — *Troubleshoot and Scale Applications*

> U dokumentu je naveden **samo naslov** (`compreview-scale`), bez specifikacije. Sljedeća pitanja su izvedena iz naslova laba i iz upozorenja prvog laba: *„you must configure probes and **resource limits**. Another comprehensive review exercise covers these subjects."*

| Pitanje | Zašto je izdvojeno |
|---|---|
| **LO5-23** · `OOMKilled` | Izravna posljedica dodavanja resource limita — postaviš li memoriju prenisko, MySQL dobije SIGKILL pri inicijalizaciji. Lab koji uči postavljati limite nužno uči i prepoznati što se dogodi kad su krivi. |
| **LO5-27** · `requests` veći od kapaciteta nodea | Druga strana istog novčića: previsoki `requests` ne ubiju pod nego ga ostave u `Pending` zauvijek. Par s LO5-23 — jedan kvar od premalo, drugi od previše. |
| **LO5-19** · pod zaglavljen u `Pending` | Skupni oblik prethodnog: nedovoljno resursa, nezadovoljen `nodeSelector`, nevezan PVC. Kod skaliranja je najčešći scenarij — replike prolaze dok ima mjesta, pa se zaustave. |
| **LO5-42** · `restartCount` i `lastState` preko jsonpatha | Vještina **čitanja** svega gore. Bez `lastState.terminated.reason` ne razlikuješ `OOMKilled` od obične greške aplikacije. Ovo je alat kojim potvrđuješ LO5-23. |
| **LO5-30** · triaža preko `get events --sort-by` | Kad skaliraš i pola replika ne krene, `describe` po podu je prespor. Eventi sortirani po vremenu pokažu scheduler, kubelet i controller na jednom mjestu. |
| **LO5-34** · node `NotReady` | Plauzibilno u labu o skaliranju — gomilanje replika izaziva `MemoryPressure`/`DiskPressure` na nodeu. Označeno kao **izvedeno**, ne potvrđeno specifikacijom. |
| **LO5-35** · pod zaglavljen u `Terminating` | Javlja se pri skaliranju prema dolje. Također **izvedeno**. Vrijedi znati zbog rizika force-deletea, pogotovo jer lab ima bazu s PVC-om. |

#### C · Greške u manifestima — nastaju jer je lab ograničen na 35 minuta

| Pitanje | Zašto je izdvojeno |
|---|---|
| **LO5-20** · maknuti razmaci u deployment manifestu | Lab ima **rok od 35 minuta**, pa ljudi rade brzo i kopiraju YAML. Ako lab rješavaš deklarativno (izvezeš manifest pa ga uređuješ), pogrešna indentacija je najčešći način da izgubiš minute. Pitanje traži baš to: prepoznati poruku i popraviti je. |
| **LO5-26** · `selector.matchLabels` ≠ `template.metadata.labels` | Nastaje čim rukom uređuješ izvezeni Deployment i promijeniš labelu samo na jednom mjestu. API server to odbija pri validaciji, s porukom koju moraš znati pročitati. |
| **LO5-36** · kriv par `apiVersion` / `kind` | Isti uzrok — pisanje manifesta napamet umjesto generiranja. U labu se piše PVC i Service, gdje je lako staviti `apps/v1` umjesto `v1`. |
| **LO5-40** · `--dry-run=server` za hvatanje typo-a prije primjene | **Obrana od sva tri prethodna.** U labu s rokom od 35 minuta razlika između `--dry-run=server` i „primijeni pa vidi" je razlika između završenog i nezavršenog zadatka. Serverski dry-run hvata i nepoznata polja koja bi inače tiho prošla. |

#### D · Dijagnostički alati kojima lab provjeravaš

| Pitanje | Zašto je izdvojeno |
|---|---|
| **LO5-31** · `exec` + `nslookup` + `/etc/resolv.conf` | Ovime **dokazuješ** da se `quotesdb` razrješava iz frontend poda. Bez toga na LO5-41 samo nagađaš gdje je karika pukla. |
| **LO5-33** · jednokratni `busybox` pod za test konektivnosti | Način da testiraš Service baze **bez** frontenda i tako odgovoriš je li problem u mreži ili u aplikaciji. Nezamjenjivo u labu gdje dvije komponente ovise jedna o drugoj. |
| **LO5-32** · ephemeral debug kontejner (`kubectl debug`) | Rezervna opcija ako `famous-quotes:2-42` nema shell — `oc rsh` tada puca s *executable file not found*, a `oc debug` ili `kubectl debug --target` ubaci alate bez restarta poda. |

#### E · Što Comprehensive Review 2 NE pokriva

| Skupina | Pitanja | Zašto |
|---|---|---|
| **Cijela podman polovica** | **LO5-1 do LO5-18** | Comprehensive Review 2 je **DO180 — čisti Kubernetes/OpenShift**. Sve greške u `podman run` naredbama i Containerfileovima pripadaju DO188 gradivu, dakle Comprehensive Reviewu **1** i Beeper labu. Ovo je 18 od 43 pitanja — najveći pojedinačni blok. |
| Kontejner bez long-running procesa | **LO5-29** | Lab koristi stvarne aplikacijske imagee (`mysql-80`, `famous-quotes`) s ispravnim entrypointom. Goli `ubuntu` ili `busybox` bez naredbe se ne pojavljuje. |
| Secret iz drugog namespacea | **LO5-38** | Sve je u jednom projektu `review`. Namespace scoping se ne vježba jer nema drugog namespacea. |

#### Sažetak

Od 43 LO5 pitanja Comprehensive Review 2 **izravno proizvodi 9** (LO5-21, 22, 24, 25, 28, 37, 39, 41, 43), **pokriva još 14** kroz drugi lab, greške u manifestima i dijagnostičke alate (LO5-19, 20, 23, 26, 27, 30, 31, 32, 33, 34, 35, 36, 40, 42), a **20 ne dotiče** — od čega su 18 podman i Containerfile greške koje pripadaju Comprehensive Reviewu 1.

Praktična posljedica: **LO5 se ne može pokriti jednim labom.** Podijeljen je gotovo na pola između dva poglavlja:

- **Podman polovica (LO5-1…18)** → vježbaj kroz **Beeper** (Comprehensive Review 1). Najbrži način: namjerno pokvari svaki korak Beepera — zamijeni portove, makni `:Z`, makni `apt-get update`, stavi `--memory 8m` — pa popravi. Devet od osamnaest pitanja proizvedeš na jednom stacku.
- **Kubernetes polovica (LO5-19…43)** → vježbaj kroz **Famous Quotes**, i to tako da lab **prvo namjerno pokvariš**: krivi prefiks secreta, nepostojeći storage class, tag koji ne postoji u ImageStreamu, readiness probe na krivom portu. Svaki od tih kvarova daje jedno pitanje iz kategorije A.

---

## E · PROJEKT · troubleshooting dnevnik

> Bodovanje projekta za LO5 glasi doslovno: *„Noted down and reflected on troubleshooting steps for fixing issues which occurred while working on the project — 1 point"* i *„Showcased details of solution in the recorded video — 2 points."* Dakle **2 od 3 boda nose video i dokumentacija, ne sam popravak.** Ako si problem riješio a nisi ga zapisao, bodova nema.

### Format zapisa koji nosi bod

Za svaki problem napiši **pet stavki**. Bez zadnje dvije to je samo dnevnik, ne refleksija.

| Polje | Primjer |
|---|---|
| **Simptom** | `tempconv-app` u `CrashLoopBackOff`, stranica vraća 500 |
| **Dijagnostika** | `podman logs --previous tempconv-app` → `OperationalError: (2005) Unknown MySQL server host 'db'` |
| **Uzrok** | kontejneri na default mreži — nema DNS-a, ime `db` se ne razrješava |
| **Popravak** | `podman network create tempconv-net` i oba kontejnera s `--network tempconv-net` |
| **Pouka** | DNS po imenu radi **samo** na user-defined mrežama; compose to radi automatski jer sam stvara mrežu |

Uz svaki zapis **screenshot prije i poslije** — poruka greške i ispravan rad. To je ono što se u videu pokazuje.

### Problemi koji se na ovom projektu stvarno pojave

Prođi ovaj popis — velika je šansa da si barem pola ovoga doživio i da se možeš sjetiti detalja.

| # | Simptom | Uzrok | Popravak |
|---|---|---|---|
| 1 | `curl localhost:8080` → prazan odgovor, a kontejner radi | Flask sluša na `127.0.0.1` unutar kontejnera | `app.run(host="0.0.0.0", port=5000)` |
| 2 | `Unknown MySQL server host 'db'` | default mreža bez DNS-a | user-defined mreža (LO5-7) |
| 3 | `Can't connect to MySQL server ... (111)` u prvih 30 s | app krenuo prije nego se baza inicijalizirala | healthcheck + `depends_on: condition: service_healthy` (LO3-8) |
| 4 | `Access denied for user 'tempuser'@'%'` | `MYSQL_USER` je dodan **nakon** prve inicijalizacije — entrypoint stvara korisnika samo kad je data direktorij prazan | `podman volume rm` pa ponovno pokreni, ili ručni `CREATE USER` + `GRANT` |
| 5 | Podaci nestali nakon `podman rm` | nema named volumea | `-v tempconv-data:/var/lib/mysql` (LO3-4) |
| 6 | `E: Unable to locate package` u buildu | `apt-get install` bez `apt-get update` | oba u istom `RUN` (LO5-10) |
| 7 | Build traje 3 min pri svakoj promjeni koda | `COPY . .` prije `pip install` | `COPY requirements.txt` prvo (LO5-12) |
| 8 | Image ~1,2 GB | pun `python` image + build alati ostali unutra | `python:3.12-slim`, `--no-install-recommends`, čišćenje u istom sloju, po potrebi multi-stage |
| 9 | `Permission denied` na bind mountu | nema SELinux oznake | `:Z` ili `:z` (LO5-8) |
| 10 | `podman push` → `unauthorized` | nema prijave ili krivi namespace u tagu | `podman login quay.io`, tag mora biti `quay.io/<user>/<repo>:<tag>` |
| 11 | `ImagePullBackOff` na k8s s privatnim repoom | nema pull secreta | `kubectl create secret docker-registry` + `imagePullSecrets` (LO4-2) |
| 12 | `toomanyrequests` s Docker Huba | anonimni rate limit | autentificiraj se ili koristi Quay |
| 13 | Treća replika `Pending` nakon skaliranja | `podAntiAffinity: required` a manje nodova nego replika | `preferred`, ili `minikube start --nodes=3` |
| 14 | Service vraća 503 | prazan `endpoints` — selektor ne pogađa labele ili pod nije Ready | uskladi labele / popravi readiness probe (LO5-25) |
| 15 | Swarm replika `Pending`, *no suitable node* | `max_replicas_per_node: 1` uz premalo nodova | podigni broj ili prijeđi na `preferences: spread` |
| 16 | Aplikacija radi lokalno, pada u clusteru | env varijable postoje samo u `podman run`, nisu prenesene u manifest | ConfigMap/Secret + `envFrom` (LO4-35) |

### Struktura poglavlja u projektnoj dokumentaciji

Predložak (`project_template_algebra_bernays_EN_2025.docx`) traži odgovore na pet pitanja. Za troubleshooting dio uklopi ovako:

- **Pitanje 3 — „Što sam napravio da riješim problem"**: ovdje ide dnevnik gore, kronološki, s naredbama i ispisima grešaka.
- **Pitanje 4 — „Rezultati i tehnička rješenja"**: konačne verzije Containerfilea, compose datoteke i manifesta, sa screenshotovima aplikacije koja radi.
- **Pitanje 5 — „Sljedeći koraci"**: ono što lab sam priznaje da nedostaje — probes, resource limiti, NetworkPolicy, StatefulSet za bazu, HPA, centralizirano logiranje, skeniranje imagea (`podman scan` / Trivy), potpisivanje imagea.

Formalni zahtjevi predloška: **DIN A4, sve margine 20 mm, Times New Roman, prored 1, obostrano poravnanje, najmanje 3 pune stranice, najmanje 4 reference u uglatim zagradama `[1]`.** Tablice i slike se numeriraju arapskim brojevima s potpisom **ispod** (10 pt).

### Video (2 boda)

Najviše bodova nosi upravo video, a najlakše ga je podbaciti. Pokaži, redom:

1. `podman images` i `podman ps` — da image i kontejneri stvarno postoje
2. aplikaciju u browseru s **tvojim imenom i fakultetom** (varijable `STUDENT` i `COLLEGE`)
3. konverziju temperature koja se stvarno izvrši
4. dokaz spajanja na bazu **kao non-root**: `SELECT CURRENT_USER();`
5. pokretanje pipelinea — testovi prolaze, image se gradi
6. deploy na jednostavan orkestrator + `docker service ps` s replikama na različitim nodovima
7. **skaliranje uživo** na 3 replike i dokaz da promet ide na više replika
8. deploy na složeni orkestrator + `kubectl get pods -o wide` (različiti NODE stupci)
9. barem jedan **troubleshooting slučaj** — namjerno pokvari nešto, pokaži grešku, pa popravak

Deveta točka je ono što razlikuje „radi" od „razumijem zašto radi" i izravno hrani LO5 bodove.

---

## ROKOVI · LO5 greške s lipanjskog roka

> Bilješka s roka, doslovno (uz napomenu *„bile su greške, otprilike ne sjecam se tocno"*):
>
> ```
> podman -name nesta -p 8080:80 arm32v5/nesta
>
> FROM python3.8
> WRKDIR /app
> EXPOSE 800
> ENTRYPOINT ["pythn", "-m", "http.nesta"]
> ```
> *„zadnji je bio neki duzi popis naredbi"*
>
> `nesta` u bilješci znači „nešto" — student se nije sjetio točnog imena. Same **greške** su zapamćene točno i to je ono što vrijedi.

**Format odgovora:** za svaku grešku napiši **(1) što je krivo, (2) ispravak, (3) zašto je bilo krivo**. Sva tri dijela nose bodove.

---

### Rok 1 · Pokvarena `podman` naredba

```bash
podman -name nesta -p 8080:80 arm32v5/nesta        # ✗
```

#### Šest grešaka u jednom retku

| # | Greška | Ispravak | Zašto |
|---|---|---|---|
| 1 | **Nema podnaredbe** | `podman **run** ...` | `podman` je alat s podnaredbama (`run`, `build`, `ps`…). Bez nje: `Error: unrecognized command` |
| 2 | **`-name` s jednom crticom** | `--name` | Duge opcije traže **dvije** crtice. `-n` nije kratica za name → `Error: unknown shorthand flag: 'n'` |
| 3 | **Nema `-d`** | `-d` | Bez toga kontejner ostaje u prvom planu i blokira terminal |
| 4 | **`arm32v5/...` — kriva arhitektura** | `docker.io/library/nginx` | ARM image na x86_64 stroju → `no matching manifest for linux/amd64` ili `exec format error` |
| 5 | **Nema registra u imenu** | `docker.io/library/...` | Kratko ime traži short-name razrješavanje; u neinteraktivnom načinu puca |
| 6 | **`-p 8080:80` neprovjereno** | provjeri stvarni port | Desna strana mora biti port na kojem aplikacija **stvarno sluša** u kontejneru |

#### Ispravljena naredba

```bash
podman run -d --name mojkontejner -p 8080:80 docker.io/library/nginx
podman ps
curl -i http://localhost:8080
```

#### Provjere koje potvrđuju ispravak

```bash
podman image inspect docker.io/library/nginx --format '{{.Architecture}} {{.Os}}'
podman image inspect docker.io/library/nginx --format '{{.Config.ExposedPorts}}'
podman port mojkontejner
```

**Rečenica za bodove:** `-p` je **`HOST:CONTAINER`**. Lijeva strana je slobodan port na hostu, desna mora odgovarati portu na kojem proces sluša unutar kontejnera. Arhitektura imagea mora odgovarati arhitekturi hosta — `arm32v5`, `arm64v8` i `amd64` nisu zamjenjivi bez emulacije (`qemu-user-static`).

---

### Rok 2 · Pokvaren Containerfile

```dockerfile
FROM python3.8                                   # ✗
WRKDIR /app                                      # ✗
EXPOSE 800                                       # ✗
ENTRYPOINT ["pythn", "-m", "http.nesta"]         # ✗ ✗
```

#### Šest grešaka

**1 · `FROM python3.8` — nedostaje dvotočka**
Ime i tag odvajaju se **dvotočkom**: `python:3.8`. Ovako napisano podman traži image imena `python3.8`, koji ne postoji → `short-name resolution error` / `manifest unknown`.
Uz to: Python 3.8 je izvan podrške; bez registra ime je dvosmisleno.
→ `FROM docker.io/library/python:3.12-slim`

**2 · `WRKDIR` — typo**
Nema takve instrukcije. Build pada odmah: `Error: unknown instruction: WRKDIR`.
→ `WORKDIR /app`

**3 · `EXPOSE 800` — dvostruka greška**
Vjerojatno je mišljeno **8000** (zadani port `http.server`). I bitnije: **`EXPOSE` ne objavljuje port** — to je samo metapodatak. Objavljuje `-p` pri pokretanju.
→ `EXPOSE 8000` + `podman run -p 8000:8000`

**4 · `pythn` — typo u imenu programa**
Runtime greška, ne build greška: image se izgradi, kontejner odmah padne s
`exec: "pythn": executable file not found in $PATH` → **exit 127**.
→ `python`

**5 · `http.nesta` — nepostojeći modul**
Modul se zove **`http.server`**. Inače: `No module named http.nesta` → exit 1.
→ `http.server`

**6 · Nema `COPY` — poslužuje prazan direktorij**
`http.server` poslužuje sadržaj radnog direktorija. Bez `COPY` u `/app` nema ničega — server radi, ali vraća prazan popis.
→ `COPY . /app`

#### Ispravljeni Containerfile

```dockerfile
FROM docker.io/library/python:3.12-slim
WORKDIR /app
COPY . /app
EXPOSE 8000
ENTRYPOINT ["python", "-m", "http.server", "8000"]
```

```bash
podman build -t mystatic .
podman run -d --name web -p 8000:8000 mystatic
curl -i http://localhost:8000
```

**Zašto eksplicitni port u `ENTRYPOINT`:** `http.server` bez argumenta koristi 8000, ali oslanjanje na default je krhko — i ovako je usklađeno s `EXPOSE`.

#### Tablica: gdje koja greška puca

| Greška | Puca pri | Poruka |
|---|---|---|
| `FROM python3.8` | **build**, prvi korak | `manifest unknown` / short-name error |
| `WRKDIR` | **build**, parsiranje | `unknown instruction: WRKDIR` |
| `EXPOSE 800` | **nigdje** — tiho krivo | `curl` ne dobiva odgovor |
| `pythn` | **runtime** | `executable file not found`, exit **127** |
| `http.nesta` | **runtime** | `No module named`, exit 1 |
| nema `COPY` | **runtime**, tiho | prazan popis datoteka |

**Ovo je poanta zadatka:** greške pucaju na **tri različite razine** — build, runtime i „tiho krivo". Zadnje dvije kategorije se ne vide iz `podman build`, nego tek iz `podman ps -a` i `podman logs`.

---

### Rok 3 · „Duži popis naredbi"

Bilješka ne čuva sadržaj, ali čuva **oblik zadatka**: niz naredbi u kojima treba naći greške. Metoda je uvijek ista.

#### Kontrolna lista po naredbi

```bash
podman run -d --name web -p 8080:80 docker.io/library/nginx
#      │   │    │          │          └─ registar/putanja + tag? arhitektura?
#      │   │    │          └─ HOST:CONTAINER — je li obrnuto? sluša li app na tom portu?
#      │   │    └─ dvije crtice? ime već zauzeto?
#      │   └─ treba li pozadina? je li --rm u sukobu s -d?
#      └─ postoji li podnaredba?
```

1. **Podnaredba** — `run`, `build`, `exec`, `pull`, `logs`… postoji li uopće?
2. **Crtice** — `-name` → `--name`, `-network` → `--network`
3. **Redoslijed** — opcije idu **prije** imagea; sve iza imagea je naredba za kontejner
4. **`-p HOST:CONTAINER`** — najčešće zamijenjeno (LO5-1)
5. **`-e VAR` bez `=`** — preuzima s hosta; ako nije postavljeno, ne prosljeđuje ništa (LO5-2)
6. **`--network host` + `-p`** — `-p` se tiho ignorira (LO5-3)
7. **Nema long-running procesa** → `Exited (0)` (LO5-4)
8. **`--rm` + `-d`** → logovi nestanu (LO5-5)
9. **Premali `--memory`** → exit 137, OOMKilled (LO5-6)
10. **Default mreža** → nema DNS-a između kontejnera (LO5-7)
11. **Bind mount bez `:Z`** → Permission denied na RHEL-u (LO5-8)
12. **Ime/tag imagea** — dvotočka, registar, arhitektura

#### Redoslijed dijagnostike na ispitu

```bash
podman ps -a                                   # Created / Exited (kod) / Up?
podman logs <c>                                # zadnja poruka prije smrti
podman inspect <c> --format '{{.State.ExitCode}} {{.State.OOMKilled}}'
podman port <c>                                # je li port stvarno mapiran
podman exec <c> ss -tlnp                       # sluša li app na očekivanom portu
```

| Exit | Značenje |
|---|---|
| `0` | proces uredno završio — nema long-running naredbe |
| `1` | greška aplikacije → čitaj logove |
| `125` | greška samog podmana (kriv flag) |
| `127` | **naredba nije nađena** — typo u imenu programa (kao `pythn`) |
| `137` | SIGKILL — OOMKilled |

---

### Što ovo znači za pripremu

Lipanjski LO5 je bio **dvodijelan**: pokvarena `podman run` naredba i pokvaren Containerfile, plus duži niz naredbi. To se točno poklapa s podjelom praktičnih pitanja:

| Dio roka | Odgovarajuća pitanja |
|---|---|
| Pokvarena `podman` naredba | **LO5-1 do LO5-8** |
| Pokvaren Containerfile | **LO5-10 do LO5-18** |
| Duži popis naredbi | kombinacija svega gore |

**Kubernetes dijagnostika (LO5-19…42) nije zabilježena ni na jednom roku** — ali je u praktičnim pitanjima i u DO180 gradivu, pa se ne smije preskočiti.

**Najbolja vježba:** uzmi svoj ispravan Beeper ili tempconverter stack i **namjerno ga pokvari** na svaki od 12 načina s kontrolne liste, pa svaki popravi i zapiši uzrok. Time istovremeno pokrivaš i format odgovora koji se traži.

---

### LO5 · Šalabahter dijagnostike

```bash
# PODMAN
podman ps -a ; podman logs <c> ; podman inspect <c>
podman inspect <c> --format '{{.State.ExitCode}} {{.State.OOMKilled}}'
podman diff <c> ; podman top <c> ; podman stats --no-stream ; podman events

# KUBERNETES — 4 naredbe koje rješavaju 90 %
kubectl get pods
kubectl describe pod <p>              # ← EVENTS na dnu
kubectl logs <p> --previous
kubectl get events --sort-by=.lastTimestamp

# DUBLJE
kubectl get pod <p> -o yaml
kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].lastState}'
kubectl exec -it <p> [-c c] -- sh
kubectl debug -it <p> --image=busybox --target=<c>
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh
kubectl get endpoints <svc>           # ← prvo za svaki mrežni problem
kubectl apply -f f.yaml --dry-run=server
kubectl rollout status|undo deploy/<d>
kubectl delete pod <p> --grace-period=0 --force     # zadnja opcija
```


---

*Skripta složena iz materijala u folderu „Intro to devops priprema za ispit“ — bilješke s rokova, praktična pitanja, labovi DO188/DO180, RHA comprehensive review i projektni zadatak.*
