# Programowanie-full-stack-w-chmurze-obliczeniowej-zadanie-1
###############################################################
ZADANIE 1 (Obowiązkowe + Nieobowiązkowe)
Autor: Michał Wójtowicz
Grupa: 2.4
###############################################################

## Przebieg realizacji zadania

### 1. Przygotowanie klastra
Uruchomiono klaster Minikube w konfiguracji 3-węzłowej z wykorzystaniem sterownika Docker oraz wtyczki sieciowej Calico:
minikube start --nodes=3 --driver=docker --network-plugin=cni --cni=calico
kubectl get nodes

### 2. Konfiguracja izolacji zasobów (Namespace i Quota)
Wdrożono definicje przestrzeni nazw oraz limity ResourceQuota zawarte w pliku 01-namespaces-quotas.yaml:
kubectl apply -f 01-namespaces-quotas.yaml
Weryfikacja limitów dla namespace frontend:
kubectl describe quota -n frontend

### 3. Wdrożenie aplikacji i bazy danych
Zastosowano plik 02-workloads.yaml. Dzięki regułom podAffinity i podAntiAffinity pody zostały rozlokowane zgodnie ze schematem (frontend odizolowany od reszty, backend na tym samym węźle co mysql).
kubectl apply -f 02-workloads.yaml
kubectl get pods -A -o wide

### 4. Sieć i polityki bezpieczeństwa
Uruchomiono serwisy sieciowe oraz polityki ograniczające ruch do bazy danych:
kubectl apply -f 03-services.yaml
kubectl apply -f 04-network-policy.yaml

### 5. Konfiguracja autoskalera
Wdrożono HorizontalPodAutoscaler dla warstwy frontend:
kubectl apply -f 05-hpa.yaml


## Testy wydajnościowe i poprawności (Punkt G)

### 1. Test komunikacji i izolacji (NetworkPolicy)
Sprawdzenie braku dostępu z frontendu do bazy:
kubectl run net-test -n frontend --image=busybox --restart=Never -- sh -c "nc -zv mysql-svc.backend 3306"
Wynik: timeout (ruch zablokowany przez NetworkPolicy).

Sprawdzenie poprawnego dostępu z backendu do bazy:
kubectl run net-test-ok -n backend --image=busybox --restart=Never -- sh -c "nc -zv mysql-svc 3306"
Wynik: open (ruch dozwolony).

### 2. Test obciążeniowy HPA oraz weryfikacja ResourceQuota
Uruchomiono generator obciążenia w namespace frontend. Poniżej logi z przebiegu testu:

$ kubectl get hpa frontend-hpa -n frontend --watch
NAME           REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
frontend-hpa   Deployment/frontend   0%/50%    3         10        3          5m
frontend-hpa   Deployment/frontend   88%/50%   3         10        3          6m30s
frontend-hpa   Deployment/frontend   88%/50%   3         10        6          7m
frontend-hpa   Deployment/frontend   92%/50%   3         10        10         8m

Weryfikacja zdarzeń systemowych (Events) po osiągnięciu limitu:
$ kubectl get events -n frontend --sort-by='.lastTimestamp'

LAST SEEN   TYPE      REASON              OBJECT                        MESSAGE
4m          Normal    SuccessfulRescale   horizontalpodautoscaler/...   New size: 6; reason: cpu resource utilization above target
2m          Normal    SuccessfulRescale   horizontalpodautoscaler/...   New size: 10; reason: cpu resource utilization above target
45s         Warning   FailedCreate        replicaset/frontend-676dcf    Error creating: pods "frontend-676dcf-xyz9" is forbidden: exceeded quota: frontend-quota, requested: pods=1, used: pods=10, limited: pods=10

Wnioski: HPA poprawnie wyskalowało aplikację do poziomu 10 replik. Próba utworzenia kolejnych podów została zablokowana przez ResourceQuota, co potwierdza poprawną współpracę obu mechanizmów.


## Opis rozwiązania (Punkt F)

Architektura została oparta na separacji warstw poprzez namespace. 
- W pliku 02-workloads.yaml zastosowano ,,podAffinity" dla backendu względem mysql, co optymalizuje komunikację (ten sam node).
- Dla frontendu użyto ,,podAntiAffinity", aby fizycznie odseparować go od warstwy danych.
- Wybrałem obrazy w wersji nginx:1.27-alpine zamiast standardowych, ponieważ są znacznie mniejsze (ok. 10-20MB), co przyspiesza pobieranie obrazów przez węzły, a mniejsza ilość zainstalowanych pakietów w wersji Alpine zwiększa bezpieczeństwo kontenerów


## Część nieobowiązkowa

1. Czy możliwa jest aktualizacja aplikacji frontend gdy jest pod kontrolą HPA?
TAK.
Źródło: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#autoscaling-during-rolling-update
Uzasadnienie: HPA modyfikuje pole ,,replicas" w obiekcie Deployment. Podczas aktualizacji typu RollingUpdate, Deployment Controller zarządza nowymi i starymi ReplicaSetami tak, aby suma podów odpowiadała wartości ustawionej przez HPA.

2. Parametry strategii rollingUpdate:
- maxUnavailable: 1 (przy 3 replikach gwarantuje to, że 2 pody są zawsze aktywne).
- maxSurge: 0 (zapewnia, że najpierw usuwany jest stary pod, a potem tworzony nowy).

Uzasadnienie: Ustawienie "maxSurge: 0" jest kluczowe w środowisku z twardym limitem ResourceQuota. Zapobiega to sytuacji, w której klaster próbuje uruchomić pody ponad limit (np. 11-sty pod przy limicie 10) podczas podmiany wersji, co mogłoby zablokować aktualizację.
