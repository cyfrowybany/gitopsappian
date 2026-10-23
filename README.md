# gitopsappian

1. Docelowa struktura katalogów
Tak powinno wyglądać repozytorium dla TST:


appian-gitops/
├── base/
│   ├── appian.yaml
│   ├── storage-patch.yaml
│   └── kustomization.yaml
│
├── environments/
│   ├── tst/
│   │   ├── kustomization.yaml
│   │   ├── route.yaml
│   │   └── patch-appian.yaml
│   │
│   └── lifecycle/
│       └── tst/
│           └── kustomization.yaml
│
└── argocd/
    └── noprod/
        └── appian-tst.yaml

Nie tworzymy osobnych katalogów active i stopped. Job GitLab będzie zmieniał jeden plik: environments/lifecycle/tst/kustomization.yaml. Dzięki temu ścieżka źródłowa aplikacji lifecycle pozostaje stała.

2. base/kustomization.yaml
To baza dla właściwego zasobu Appian. Umieszczamy tu bazowy CR i dotychczasowy patch storage.

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - appian.yaml

patches:
  - path: storage-patch.yaml

Pliki base/appian.yaml oraz base/storage-patch.yaml pozostają Twoimi aktualnymi manifestami. Nie przepisuję ich zawartości, bo trzeba zachować Twoje obecne definicje komponentów, storage, licencji i konfiguracji.

3. environments/tst/patch-appian.yaml
Ten plik zostaje w katalogu TST, ale nie jest bezpośrednio stosowany przez główną aplikację TST. Będzie używany przez aplikację lifecycle do złożenia pełnego CR.
Pozostaw w nim swój obecny patch wersji i konfiguracji Appian, np. te zmiany, które teraz wprowadzasz przy upgrade.
Nie dodawaj tego pliku do environments/tst/kustomization.yaml. W przeciwnym razie patch nie miałby właściwego zasobu bazowego, do którego można go zastosować.
4. environments/tst/kustomization.yaml
Ta aplikacja zarządza tylko zasobami środowiskowymi, takimi jak Route. Nie może zarządzać CR Appian, ponieważ za niego odpowiada lifecycle.


apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - route.yaml

Jeżeli w Twojej obecnej konfiguracji są dodatkowe manifesty środowiskowe, pozostaw je na liście resources. Nie dodawaj tu ../base, appian.yaml ani patch-appian.yaml.
5. environments/lifecycle/tst/kustomization.yaml
To jest jedyny plik, którym job GitLab będzie sterował.
Stan normalny — Appian ma działać


apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../../base

patches:
  - path: ../../tst/patch-appian.yaml

Ścieżki są następujące:
../../../base — katalog base z bazowym CR i patchem storage.
../../tst/patch-appian.yaml — patch środowiska TST.
Kustomize składa z nich kompletny zasób Appian.
Stan zatrzymany — GitOps ma usunąć CR
Job zmienia zawartość tego samego pliku na:


apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources: []


Po commicie aplikacja lifecycle otrzyma pusty zestaw manifestów. Dzięki automated.prune: true i allowEmpty: true Argo CD może usunąć CR Appian.
Przed wdrożeniem sprawdź renderowanie pustego zestawu przez swoją wersję Kustomize i Argo CD. Usunięcie CR jest operacją destrukcyjną na poziomie zasobu niestandardowego, więc trzeba też potwierdzić zachowanie finalizerów i operatora Appian 205.

6. argocd/noprod/appian-tst.yaml — ApplicationSet
Wykorzystujemy jeden generator list i dwa elementy. Główna aplikacja ma sync manualny, lifecycle ma auto-sync i pruning, ponieważ musi wykonywać zmiany obecności CR na podstawie Git.
Zachowuję ServerSideApply=true z Twojej wcześniejszej konfiguracji. Dla lifecycle również go ustawiamy.


apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: appian-tst
  namespace: openshift-gitops

spec:
  goTemplate: true
  goTemplateOptions:
    - missingkey=error

  generators:
    - list:
        elements:
          # Zasoby środowiskowe TST - sync manualny
          - appName: appian-tst
            sourcePath: environments/tst
            autoSync: false
            autoPrune: false
            allowEmpty: false

          # Appian CR - cykl życia sterowany przez GitOps
          - appName: appian-tst-lifecycle
            sourcePath: environments/lifecycle/tst
            autoSync: true
            autoPrune: true
            allowEmpty: true

  template:
    metadata:
      name: '{{.appName}}'
      labels:
        appian-environment: tst

    spec:
      project: default

      source:
        repoURL: https://gitlab.example.com/platform/appian-gitops.git
        targetRevision: main
        path: '{{.sourcePath}}'

      destination:
        server: https://kubernetes.default.svc
        namespace: appian-tst

      syncPolicy:
        syncOptions:
          - CreateNamespace=false
          - ServerSideApply=true

  templatePatch: |
    {{- if .autoSync }}
    spec:
      syncPolicy:
        automated:
          enabled: true
          prune: {{ .autoPrune }}
          selfHeal: false
          allowEmpty: {{ .allowEmpty }}
        syncOptions:
          - CreateNamespace=false
          - ServerSideApply=true
    {{- end }}


7. Job GitLab CI — wyłączenie Appiana
Dostosowałem rozwiązanie do wzorca, którego używałeś wcześniej: podman run --rm, montowanie pliku jako /manifest.yaml i wykonywanie yq -y -i wewnątrz kontenera. 
Zakładam, że zmienne YQ_IMAGE i MANIFEST_PATH masz już zdefiniowane w swoim pipeline. MANIFEST_PATH ma wskazywać na plik:
environments/lifecycle/tst/kustomization.yaml
W stanie wyłączonym plik ma zawierać:


stages:
  - appian_lifecycle

appian_stop:
  stage: appian_lifecycle

  variables:
    MANIFEST_PATH: "environments/lifecycle/tst/kustomization.yaml"

  script:
    - |
      podman run --rm \
        -v "${MANIFEST_PATH}:/manifest.yaml" \
        "${YQ_IMAGE}" \
        yq -y -i '.resources = [] | del(.patches)' /manifest.yaml

    - |
      if git diff --quiet -- "$MANIFEST_PATH"; then
        echo "Appian jest już w stanie stopped."
        exit 0
      fi

    - git config user.name "GitLab CI"
    - git config user.email "gitlab-ci@example.com"
    - git add "$MANIFEST_PATH"
    - git commit -m "chore(appian-tst): stop Appian"
    - git push origin "HEAD:${CI_COMMIT_BRANCH}"

  rules:
    - when: manual


8. Job GitLab CI — ponowne włączenie Appiana
Ten job przywraca zawartość kustomization.yaml do stanu aktywnego:


appian_start:
  stage: appian_lifecycle

  variables:
    MANIFEST_PATH: "environments/lifecycle/tst/kustomization.yaml"

  script:
    - |
      podman run --rm \
        -v "${CI_PROJECT_DIR}/${MANIFEST_PATH}:/manifest.yaml" \
        "${YQ_IMAGE}" \
        yq -y -i '.resources = ["../../../base"] | .patches = [{"path": "../../tst/patch-appian.yaml"}]' /manifest.yaml

    - |
      if git diff --quiet -- "$MANIFEST_PATH"; then
        echo "Appian jest już w stanie active."
        exit 0
      fi

    - git config user.name "GitLab CI"
    - git config user.email "gitlab-ci@example.com"
    - git add "$MANIFEST_PATH"
    - git commit -m "chore(appian-tst): start Appian"
    - git push origin "HEAD:${CI_COMMIT_BRANCH}"

  rules:
    - when: manual


Docelowy przebieg pipeline
Pipeline 1 — zatrzymanie Appiana
Job zmienia environments/lifecycle/tst/kustomization.yaml na resources: [] i wykonuje commit. Argo CD automatycznie usuwa CR Appian.
Pipeline 1 — przygotowanie obrazów
Pobranie obrazów z Appian Hub, skanowanie, retagowanie i publikacja w lokalnym registry. Te etapy mogą działać po zatrzymaniu Appiana.
Pipeline 2 — downstream
DevOps sprawdza w Argo CD, czy CR Appian został usunięty i czy komponenty zostały zatrzymane. Dopiero wtedy uruchamia job zmiany wersji.
Job — zmiana wersji
Aktualizuje environments/tst/patch-appian.yaml i wykonuje commit. Główna aplikacja pozostaje w trybie manualnego sync.
Job — uruchomienie Appiana
Przywraca aktywną zawartość environments/lifecycle/tst/kustomization.yaml. Argo CD automatycznie odtwarza CR z nową wersją.



