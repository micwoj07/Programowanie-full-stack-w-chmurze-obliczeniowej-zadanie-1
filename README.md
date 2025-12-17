# Programowanie-full-stack-w-chmurze-obliczeniowej-zadanie-1
###############################################################
ZADANIE 1 (Obowiązkowe + Nieobowiązkowe)
Autor: Michał Wójtowicz
Grupa: 2.4
###############################################################

## Przebieg realizacji zadania

### 1. Uruchomienie środowiska
Wykonano komendę startową dla klastra 3-węzłowego z pluginem Calico:
minikube start --nodes=3 --driver=docker --network-plugin=cni --cni=calico
kubectl get nodes

### 2. Konfiguracja Namespace i limitów
Zastosowano plik 01-namespaces-quotas.yaml:
kubectl apply -f 01-namespaces-quotas.yaml
Zrobiono weryfikację limitów:
kubectl describe quota -n frontend

### 3. Wdrażanie aplikacji i baz danych
Wykonano wdrożenie workloads:
kubectl apply -f 02-workloads.yaml
Sprawdzono rozmieszczenie podów komendą:
kubectl get pods -A -o wide
Wynik: Pody frontend trafiły na inne węzły niż backend i mysql dzięki regułom Affinity.

### 4. Usługi i zabezpieczenia sieciowe
Wdrożono serwisy i polityki:
kubectl apply -f 03-services.yaml
kubectl apply -f 04-network-policy.yaml

### 5. Skalowanie
Uruchomiono HPA:
kubectl apply -f 05-hpa.yaml


## Testy (Punkt G)

### 1. Test komunikacji (NetworkPolicy)
Zrobiono test z poda frontend do mysql:
kubectl run net-test -n frontend --image=busybox --restart=Never -- sh -c "nc -zv mysql-svc.backend 3306"
Wynik: bad address / timeout (dostęp zablokowany).

Test z poda backend do mysql:
kubectl run net-test-ok -n backend --image=busybox --restart=Never -- sh -c "nc -zv mysql-svc 3306"
Wynik: open (połączenie poprawne).

### 2. Test obciążenia HPA i Quota
Wykonano pętlę generującą ruch:
kubectl run load-gen -n frontend --image=busybox -- sh -c "while true; do wget -q -O- http://frontend-svc; done"
Obserwowano wynik:
kubectl get hpa -n frontend --watch
Wynik: Replikacja wzrosła do 10. Przy próbie stworzenia 11 poda serwer API odrzucił żądanie z powodu przekroczenia ResourceQuota.


## Uzasadnienie NetworkPolicy
Polityki działają poprawnie, ponieważ zastosowano selektory etykiet (podSelector oraz namespaceSelector). Ruch do MySQL jest dopuszczony wyłącznie z podów posiadających etykietę app: backend wewnątrz tego samego namespace. Ruch do backendu jest dopuszczony tylko z namespace oznaczonym jako frontend.


## Część nieobowiązkowa

1. Czy możliwa jest aktualizacja przy aktywnym HPA?
TAK. 
Link: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#autoscaling-during-rolling-update
Uzasadnienie: Dokumentacja wskazuje, że HPA zarządza polem replicas Deploymentu, a kontroler Deploymentu koordynuje proces RollingUpdate między ReplicaSetami.

2. Parametry rollingUpdate:
- maxUnavailable: 1 (zapewnia min. 2 aktywne pody przy 3 bazowych)
- maxSurge: 0 (blokuje tworzenie nadmiarowych podów, co chroni przed przekroczeniem ResourceQuota = 10 pods)

Uzasadnienie: Przy maxSurge: 0 Kubernetes najpierw zabija stary pod, a potem tworzy nowy. Dzięki temu nigdy nie przekroczymy twardego limitu 10 podów ustalonego w Quocie, nawet jeśli HPA wyskalowało nas do maksimum.
