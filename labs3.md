
---

# **Lab : Gestion de l'Autoscalabilité dans Kubernetes**
**Objectifs** :
1. Comprendre les **mécanismes d'autoscaling** (HPA, VPA, Cluster Autoscaler).
2. Configurer et tester chaque type d'autoscaling.
3. Analyser l'impact des **LimitRanges** sur l'autoscaling.
4. Utiliser des métriques personnalisées pour déclencher l'autoscaling.

---

## **Prérequis**
- Un cluster AKS (`cluster1`) déjà déployé.
- `kubectl` configuré pour accéder au cluster.
- Metrics Server installé (nécessaire pour HPA/VPA).

---

## **Étape 0 : Installer Metrics Server(option)**
**Pourquoi ?**
Le **Metrics Server** collecte les métriques de ressources (CPU, mémoire) utilisées par HPA et VPA.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
**Vérification** :
```bash
kubectl top nodes
kubectl top pods -n kube-system
```

---

## **Partie 1 : Horizontal Pod Autoscaler (HPA)**
**Présentation** :
HPA ajuste automatiquement le nombre de réplicas d'un déploiement en fonction de la charge CPU/mémoire.

### 1.1 Déployer une application de test
**Fichier : `hpa-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: "100m"  # Requête CPU pour déclencher HPA
```

**Appliquer** :
```bash
kubectl apply -f hpa-deployment.yaml
```

### 1.2 Configurer HPA
```bash
kubectl autoscale deployment hpa-demo --cpu-percent=50 --min=1 --max=5
```
**Explications** :
- `--cpu-percent=50` : HPA déclenche si l'utilisation CPU dépasse 50%.
- `--min=1 --max=5` : Nombre minimal/maximal de réplicas.

**Vérifier** :
```bash
kubectl get hpa
```

### 1.3 Simuler une charge CPU
```bash
kubectl run -it --rm load-generator --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://hpa-demo; done"
```
**Observer** :
```bash
kubectl get hpa --watch  # Voir HPA ajuster le nombre de réplicas
```

---

## **Partie 2 : Vertical Pod Autoscaler (VPA)**
**Présentation** :
VPA ajuste automatiquement les **ressources allouées** (CPU/mémoire) aux conteneurs.

### 2.1 Installer VPA
```bash
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.13.0/vertical-pod-autoscaler-crd.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.13.0/vertical-pod-autoscaler-deployment.yaml
```

### 2.2 Configurer VPA
**Fichier : `vpa-config.yaml`**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-demo
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: hpa-demo
  updatePolicy:
    updateMode: "Auto"
```

**Appliquer** :
```bash
kubectl apply -f vpa-config.yaml
```
**Vérifier** :
```bash
kubectl get vpa
kubectl describe vpa vpa-demo
```

---

## **Partie 3 : Cluster Autoscaler (CA)**
**Présentation** :
CA ajuste automatiquement le nombre de **nœuds** dans le cluster en fonction des besoins.

### 3.1 Activer CA sur AKS
```bash
az aks update --resource-group <nom-du-groupe> --name cluster1 --enable-cluster-autoscaler --min-count 1 --max-count 3
```
**Vérifier** :
```bash
kubectl get nodes --watch  # Observer l'ajout/suppression de nœuds
```

### 3.2 Simuler une demande de ressources
Déployer un pod nécessitant plus de ressources que disponibles :
```bash
kubectl run big-pod --image=nginx --requests=cpu=2
```
**Observer** :
```bash
kubectl get nodes --watch  # CA ajoute un nœud si nécessaire
```

---

## **Partie 4 : Impact des LimitRanges sur l'Autoscaling**
**Présentation** :
Les **LimitRanges** définissent des limites par défaut pour les conteneurs, ce qui influence HPA et VPA.

### 4.1 Appliquer un LimitRange
**Fichier : `limit-range.yaml`**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: "500m"
    defaultRequest:
      cpu: "200m"
    type: Container
```

**Appliquer** :
```bash
kubectl apply -f limit-range.yaml
```

### 4.2 Tester l'impact
1. Déployer un pod sans spécifier de ressources :
   ```bash
   kubectl run test-pod --image=nginx
   ```
2. Vérifier les ressources allouées :
   ```bash
   kubectl describe pod test-pod
   ```
   → Le pod hérite des limites définies dans `LimitRange`.

3. **Impact sur HPA** :
   - Si les `requests` sont trop basses, HPA peut déclencher trop tôt.
   - Si les `limits` sont trop basses, VPA ne peut pas augmenter les ressources.

---

## **Partie 5 : Métriques Personnalisées (Optionnel)**
**Présentation** :
HPA peut utiliser des métriques personnalisées (ex: requêtes HTTP/s).

### 5.1 Installer Prometheus Adapter
```bash
kubectl apply -f https://github.com/kubernetes-sigs/prometheus-adapter/releases/download/v0.10.0/custom-metrics-apiserver-deployment.yaml
```

### 5.2 Configurer HPA avec une métrique personnalisée
```bash
kubectl autoscale deployment hpa-demo --min=1 --max=5 --metric-name=requests_per_second --target-value=10
```

---

## **Partie 6 : Nettoyage**
```bash
kubectl delete deployment hpa-demo
kubectl delete vpa vpa-demo
kubectl delete limitrange cpu-limit-range
```

---

## **Script Bash Complet**
```bash
#!/bin/bash
# Lab Autoscaling : HPA, VPA, CA, et LimitRanges

# 1. Installer Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 2. Déployer l'application pour HPA
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: "100m"
EOF

# 3. Configurer HPA
kubectl autoscale deployment hpa-demo --cpu-percent=50 --min=1 --max=5

# 4. Installer VPA
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.13.0/vertical-pod-autoscaler-crd.yaml
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.13.0/vertical-pod-autoscaler-deployment.yaml

# 5. Configurer VPA
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-demo
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: hpa-demo
  updatePolicy:
    updateMode: "Auto"
EOF

# 6. Appliquer LimitRange
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: "500m"
    defaultRequest:
      cpu: "200m"
    type: Container
EOF

# 7. Afficher les résultats
echo "=== HPA ==="
kubectl get hpa
echo "=== VPA ==="
kubectl get vpa
echo "=== LimitRange ==="
kubectl describe limitrange cpu-limit-range
```

---

## **Conclusion**
- **HPA** : Ajuste le nombre de réplicas en fonction de la charge.
- **VPA** : Ajuste les ressources allouées aux conteneurs.
- **Cluster Autoscaler** : Ajuste le nombre de nœuds.
- **LimitRanges** : Influence les décisions d'autoscaling en définissant des limites par défaut.

---
