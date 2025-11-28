---

## 📋 Lab AKS : Monitoring avec Azure Monitor et Azure Managed Grafana
**Objectifs :**
- Déployer un cluster AKS avec Azure Monitor et Log Analytics
- Configurer Container Insights
- Implémenter Azure Managed Grafana
- Créer des dashboards et alertes
- Analyser les performances

---

## 🛠️ Prérequis
### Outils nécessaires
```bash
az --version
kubectl version --client
helm version  # Optionnel
```
### Compte Azure
- Abonnement Azure avec droits contributeur
- Resource Group dédié

---

## 🔧 Partie 1 : Déploiement AKS avec Monitoring et Log Analytics

### 1.1. Configuration initiale
```bash
RESOURCE_GROUP="aks-monitoring-rg-<votre prénom>"
AKS_NAME="aks-monitoring-cluster-<votre prénom>"
LOCATION="francecentral"
LOG_ANALYTICS_WORKSPACE_NAME="aks-monitoring-logs-<votre prénom>"
```

### 1.2. Création du Resource Group
```bash
az group create --name $RESOURCE_GROUP --location $LOCATION
```

### 1.3. Création du workspace Log Analytics
```bash
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \
  --workspace-name $LOG_ANALYTICS_WORKSPACE_NAME \
  --location $LOCATION
```

### 1.4. Récupération de l'ID du workspace Log Analytics
```bash
LOG_ANALYTICS_WORKSPACE_ID=$(az monitor log-analytics workspace list --resource-group $RESOURCE_GROUP --query "[0].id" -o tsv)
echo "ID du workspace Log Analytics : $LOG_ANALYTICS_WORKSPACE_ID"
```

### 1.5. Création du cluster AKS avec monitoring et association à Log Analytics
```bash
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME \
  --node-count 3 \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --node-vm-size Standard_B2s \
  --enable-managed-identity \
  --workspace-resource-id $LOG_ANALYTICS_WORKSPACE_ID
```

### 1.6. Récupération des credentials
```bash
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_NAME --overwrite-existing
kubectl get nodes
```

---

## 📊 Partie 2 : Déploiement d'Application de Test
### 2.1. Création des fichiers de déploiement
```bash
cat <<EOF > application-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: sample-service
spec:
  selector:
    app: sample-app
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
EOF
```

```bash
cat <<EOF > load-generator.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-generator
spec:
  replicas: 2
  selector:
    matchLabels:
      app: load-generator
  template:
    metadata:
      labels:
        app: load-generator
    spec:
      containers:
      - name: loader
        image: busybox
        command: ["/bin/sh"]
        args: ["-c", "while true; do wget -q -O- http://sample-service; sleep 1; done"]
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
EOF
```

### 2.2. Déploiement des applications
```bash
kubectl apply -f application-test.yaml
kubectl apply -f load-generator.yaml
kubectl get pods,svc,deploy
```

---

## 🔍 Partie 3 : Exploration Azure Monitor et Container Insights
### 3.1. Accès à Container Insights
```bash
az aks show --resource-group $RESOURCE_GROUP --name $AKS_NAME --query id -o tsv
```
1. Allez dans le [portail Azure](https://portal.azure.com).
2. Naviguez vers votre cluster AKS.
3. Cliquez sur **Insights** dans la section Monitoring.

### 3.2. Requêtes KQL dans Log Analytics
```bash
echo "Espace de travail Log Analytics: $LOG_ANALYTICS_WORKSPACE_NAME"
```
**Exemples de requêtes :**
```kql
InsightsMetrics
| where TimeGenerated > ago(1h)
| where Name in ("cpuUsageNanoCores", "memoryWorkingSetBytes")
| summarize AvgValue = avg(Val) by bin(TimeGenerated, 1m), Name
| render timechart
```

---

## ⚠️ Partie 4 : Configuration d'Alerte Azure Monitor
### 4.1. Création d'un groupe d'actions
```bash
ACTION_GROUP_ID=$(az monitor action-group create \
  --name "AKSAlertActionGroup" \
  --resource-group $RESOURCE_GROUP \
  --action email "NomDeVotreGroupeDActions" "votre@email.com" \
  --query id -o tsv)
echo "ID du groupe d'actions : $ACTION_GROUP_ID"
```

### 4.2. Alerte sur utilisation CPU élevée
```bash
az monitor metrics alert create \
  --name "AKS-HighCPUUsage" \
  --resource-group $RESOURCE_GROUP \
  --scopes $LOG_ANALYTICS_WORKSPACE_ID \
  --condition "avg node_cpu_usage_percentage > 70" \
  --description "Alerte quand l'utilisation CPU dépasse 70%" \
  --evaluation-frequency 5m \
  --window-size 5m \
  --severity 2 \
  --action-group $ACTION_GROUP_ID
```

### 4.3. Alerte sur pods en échec
```bash
az monitor scheduled-query create \
  --name "AKS-FailedPods" \
  --resource-group $RESOURCE_GROUP \
  --scopes $LOG_ANALYTICS_WORKSPACE_ID \
  --condition "count (KubePodInventory | where PodStatus == 'Failed') > 0" \
  --description "Alerte quand des pods sont en échec" \
  --evaluation-frequency 5m \
  --window-size 5m \
  --severity 1 \
  --action-group $ACTION_GROUP_ID
```

---

## 📈 Partie 5 : Azure Managed Grafana
### 5.1. Création de l'instance Grafana
```bash
GRAFANA_NAME="aks-grafana-$(date +%s)"
az grafana create \
  --name $GRAFANA_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --skip-role-assignments false \
  --api-key enabled \
  --deterministic-outbound-ip enabled
```

### 5.2. Configuration des permissions
```bash
GRAFANA_ID=$(az grafana show --name $GRAFANA_NAME --resource-group $RESOURCE_GROUP --query id -o tsv)
az role assignment create \
  --assignee $(az grafana show --name $GRAFANA_NAME --resource-group $RESOURCE_GROUP --query identity.principalId -o tsv) \
  --role "Monitoring Reader" \
  --scope $(az aks show --resource-group $RESOURCE_GROUP --name $AKS_NAME --query id -o tsv)
az role assignment create \
  --assignee $(az grafana show --name $GRAFANA_NAME --resource-group $RESOURCE_GROUP --query identity.principalId -o tsv) \
  --role "Log Analytics Reader" \
  --scope $LOG_ANALYTICS_WORKSPACE_ID
```

### 5.3. Accès à Grafana
```bash
GRAFANA_URL=$(az grafana show --name $GRAFANA_NAME --resource-group $RESOURCE_GROUP --query properties.endpoint -o tsv)
echo "URL de Grafana: $GRAFANA_URL"
```

---

## 🎨 Partie 6 : Dashboards Grafana
### 6.1. Import de dashboards populaires
1. Allez sur l'URL de Grafana.
2. Importez les dashboards avec les IDs suivants :
   - **13105** : Kubernetes Cluster Monitoring
   - **10956** : Azure Monitor Container Insights

---

## 🚀 Partie 7 : Tests de Performance
### 7.1. Génération de charge supplémentaire
```bash
kubectl scale deployment sample-app --replicas=5
kubectl scale deployment load-generator --replicas=3
kubectl get pods -o wide
```

---

## 🧹 Nettoyage
### 9.1. Suppression des ressources
```bash
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

---

## 📚 Ressources et Documentation
- [Azure Monitor pour conteneurs](https://learn.microsoft.com/azure/azure-monitor/containers/container-insights-overview)
- [Azure Managed Grafana](https://learn.microsoft.com/azure/managed-grafana/)

---
