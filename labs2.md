---

# **Lab : Maîtrise des Capacités d’un Cluster AKS avec ResourceQuotas et LimitRanges**
**Objectif global** :
Apprendre à contrôler et optimiser l’utilisation des ressources dans un cluster AKS en utilisant des **ResourceQuotas** (quotas de ressources) et des **LimitRanges** (limites par défaut pour les conteneurs). Ce lab utilise uniquement des images publiques (`nginx`), donc **aucun registre ACR n’est nécessaire**.

---

## **Prérequis**
- Un cluster AKS (`cluster1`) déjà déployé (comme dans le lab précédent).
- Azure CLI installé et configuré (`az login`).
- `kubectl` installé (outils en ligne de commande pour Kubernetes).

---

## **Étape 1 : Configurer l’accès au cluster avec `kubectl`**
**Pourquoi ?**
`kubectl` est l’outil principal pour interagir avec votre cluster Kubernetes. Cette étape permet de configurer votre machine locale pour communiquer avec le cluster AKS.

### 1.1 Installer `kubectl`
- **Linux/macOS** :
  ```bash
  az aks install-cli  # Installe kubectl via Azure CLI
  ```
- **Windows** :
  Téléchargez et installez `kubectl` depuis [la documentation officielle](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/).

### 1.2 Récupérer les identifiants du cluster AKS
```bash
az login  # Se connecter à votre compte Azure
az aks get-credentials --resource-group <nom-du-groupe-de-ressources> --name cluster1
```
**Description** :
- `az login` : Authentifie votre session Azure CLI.
- `az aks get-credentials` : Télécharge les informations d’authentification du cluster AKS et configure `kubectl` pour pointer vers `cluster1`.
  *Remplacez `<nom-du-groupe-de-ressources>` par le nom du groupe de ressources où `cluster1` est déployé.*

### 1.3 Vérifier la connexion
```bash
kubectl get nodes  # Liste les nœuds du cluster
```
**Résultat attendu** :
- Une liste des nœuds (ex: `agentpool-123456`) avec leur statut (`Ready`).
- Confirme que `kubectl` est correctement configuré pour communiquer avec AKS.

---

## **Étape 2 : Créer un Namespace Isolé**
**Pourquoi ?**
Les **namespaces** permettent de segmenter un cluster en environnements logiques (ex: `dev`, `prod`). Ici, nous créons un namespace dédié à une équipe de développement.

```bash
kubectl create namespace dev-team  # Crée un namespace nommé "dev-team"
```

---

## **Étape 3 : Configurer un ResourceQuota**
**Pourquoi ?**
Un **ResourceQuota** limite la quantité totale de ressources (CPU, mémoire, pods) qu’un namespace peut consommer. Cela évite qu’une équipe monopolise les ressources du cluster.

### 3.1 Fichier de configuration : `resource-quota.yaml`
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-team-quota  # Nom du quota
  namespace: dev-team   # Namespace cible
spec:
  hard:
    requests.cpu: "2"    # Limite totale des requêtes CPU (2 unités)
    requests.memory: 4Gi # Limite totale des requêtes mémoire (4 Go)
    limits.cpu: "4"      # Limite totale des limites CPU (4 unités)
    limits.memory: 8Gi   # Limite totale des limites mémoire (8 Go)
    pods: "10"           # Nombre maximal de pods
```
**Explications** :
- `requests.cpu/memory` : Ressources **garanties** pour les conteneurs.
- `limits.cpu/memory` : Ressources **maximales** autorisées.
- `pods` : Nombre maximal de pods dans le namespace.

### 3.2 Appliquer le quota
```bash
kubectl apply -f resource-quota.yaml  # Applique le quota au namespace
```
### 3.3 Vérifier le quota
```bash
kubectl describe resourcequota dev-team-quota -n dev-team
```
**Résultat attendu** :
- Affichage des limites configurées et de l’utilisation actuelle (devrait être à 0 au début).

---

## **Étape 4 : Configurer un LimitRange**
**Pourquoi ?**
Un **LimitRange** définit des **valeurs par défaut** pour les requêtes et limites des conteneurs dans un namespace. Cela évite que des conteneurs sans limites consomment trop de ressources.

### 4.1 Fichier de configuration : `limit-range.yaml`
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-team-limits  # Nom de la limite
  namespace: dev-team    # Namespace cible
spec:
  limits:
  - default:
      cpu: "500m"        # Limite CPU par défaut (0.5 unité)
      memory: "512Mi"    # Limite mémoire par défaut (512 Mo)
    defaultRequest:
      cpu: "200m"        # Requête CPU par défaut (0.2 unité)
      memory: "256Mi"    # Requête mémoire par défaut (256 Mo)
    type: Container       # Applique ces limites aux conteneurs
```
**Explications** :
- `default` : Limites maximales appliquées si le conteneur n’en définit pas.
- `defaultRequest` : Requêtes minimales appliquées si le conteneur n’en définit pas.

### 4.2 Appliquer les limites
```bash
kubectl apply -f limit-range.yaml
```
### 4.3 Vérifier les limites
```bash
kubectl describe limitrange dev-team-limits -n dev-team
```
**Résultat attendu** :
- Affichage des limites et requêtes par défaut pour les conteneurs.

---

## **Étape 5 : Déployer une Application de Test**
**Pourquoi ?**
Nous déployons une application simple (`nginx`) pour tester si les quotas et limites sont respectés.

### 5.1 Fichier de déploiement : `test-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app       # Nom du déploiement
  namespace: dev-team  # Namespace cible
spec:
  replicas: 2          # Nombre de réplicas (pods)
  selector:
    matchLabels:
      app: test-app    # Étiquette pour sélectionner les pods
  template:
    metadata:
      labels:
        app: test-app  # Étiquette des pods
    spec:
      containers:
      - name: nginx
        image: nginx:latest  # Image publique, pas besoin d'ACR
        resources:
          requests:
            cpu: "300m"      # Requête CPU (0.3 unité)
            memory: "300Mi"  # Requête mémoire (300 Mo)
          limits:
            cpu: "600m"      # Limite CPU (0.6 unité)
            memory: "600Mi"  # Limite mémoire (600 Mo)
```
**Explications** :
- `resources.requests` : Ressources **garanties** pour le conteneur.
- `resources.limits` : Ressources **maximales** autorisées.

### 5.2 Appliquer le déploiement
```bash
kubectl apply -f test-deployment.yaml
```
### 5.3 Vérifier les pods
```bash
kubectl get pods -n dev-team  # Liste les pods dans le namespace
```
**Résultat attendu** :
- 2 pods en état `Running`.

---

## **Étape 6 : Tester les Limites et Quotas**
**Pourquoi ?**
Valider que les quotas et limites fonctionnent en essayant de dépasser les ressources allouées.

### 6.1 Vérifier l’utilisation des ressources
```bash
kubectl describe resourcequota dev-team-quota -n dev-team
```
**Résultat attendu** :
- Utilisation partielle des ressources (CPU/mémoire) par les pods `test-app`.

### 6.2 Tester le dépassement de quota
1. Modifiez `test-deployment.yaml` pour demander plus de ressources que le quota (ex: `requests.cpu: "3"`).
2. Réappliquez le déploiement :
   ```bash
   kubectl apply -f test-deployment.yaml
   ```
**Résultat attendu** :
- Erreur : `Exceeded quota: dev-team-quota, requested: requests.cpu=3, used: requests.cpu=600m, limited: requests.cpu=2`.
- Le déploiement échoue car il dépasse le quota CPU du namespace.

---

## **Étape 7 : Nettoyage**
**Pourquoi ?**
Supprimer les ressources créées pour éviter de consommer des ressources inutiles.

```bash
kubectl delete deployment test-app -n dev-team  # Supprime le déploiement
kubectl delete namespace dev-team               # Supprime le namespace
```

---

## **Script Bash Complet (Automatisation)**
**Pourquoi ?**
Un script permet d’exécuter toutes les étapes en une seule commande, idéal pour des tests rapides ou des démonstrations.

```bash
#!/bin/bash
# Lab AKS : ResourceQuotas et LimitRanges

# 1. Configuration de kubectl
echo "=== Configuration de kubectl ==="
az login
az aks get-credentials --resource-group <nom-du-groupe-de-ressources> --name cluster1

# 2. Création du namespace
echo "=== Création du namespace 'dev-team' ==="
kubectl create namespace dev-team

# 3. Application du ResourceQuota
echo "=== Application du ResourceQuota ==="
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-team-quota
  namespace: dev-team
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "10"
EOF

# 4. Application du LimitRange
echo "=== Application du LimitRange ==="
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-team-limits
  namespace: dev-team
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
    type: Container
EOF

# 5. Déploiement de l'application de test
echo "=== Déploiement de l'application 'test-app' ==="
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: dev-team
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: "300m"
            memory: "300Mi"
          limits:
            cpu: "600m"
            memory: "600Mi"
EOF

# 6. Affichage des résultats
echo "=== Pods ==="
kubectl get pods -n dev-team
echo "=== Quota ==="
kubectl describe resourcequota dev-team-quota -n dev-team
echo "=== Limits ==="
kubectl describe limitrange dev-team-limits -n dev-team
```
**Exécution** :
```bash
chmod +x lab-aks.sh  # Rendre le script exécutable
./lab-aks.sh         # Lancer le script
```

---
