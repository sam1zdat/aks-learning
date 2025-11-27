
---

### **1. `01-prepare-environment.sh`**
```bash
#!/bin/bash
# --------------------------------------------------------------------
# Script : 01-prepare-environment.sh
# Description : Crée le groupe de ressources Azure pour le lab AKS.
# Prérequis : Azure CLI installé et connecté (`az login`).
# --------------------------------------------------------------------

# Nom du groupe de ressources qui regroupera toutes les ressources du lab
RESOURCE_GROUP="AKS-Lab-RG"
# Région Azure où les ressources seront déployées
LOCATION="westeurope"

# Vérifie que l'utilisateur est bien connecté à Azure CLI
echo "🔍 Vérification de la connexion à Azure CLI..."
az account show > /dev/null 2>&1
if [ $? -ne 0 ]; then
    echo "❌ Erreur : Vous n'êtes pas connecté à Azure CLI. Exécutez 'az login' et réessayez."
    exit 1
fi

# Crée un groupe de ressources Azure :
# --name : Nom du groupe de ressources
# --location : Région Azure (ex: westeurope, francecentral)
echo "📦 Création du groupe de ressources '$RESOURCE_GROUP' dans '$LOCATION'..."
az group create --name $RESOURCE_GROUP --location $LOCATION
if [ $? -ne 0 ]; then
    echo "❌ Erreur : Impossible de créer le groupe de ressources."
    exit 1
fi

echo "✅ Groupe de ressources '$RESOURCE_GROUP' créé avec succès."
```

---

### **2. `02-create-aks-cluster.sh`**
```bash
#!/bin/bash
# --------------------------------------------------------------------
# Script : 02-create-aks-cluster.sh
# Description : Crée un cluster AKS avec les paramètres spécifiés.
# Prérequis : Le groupe de ressources doit exister (voir 01-prepare-environment.sh).
# --------------------------------------------------------------------

# Nom du groupe de ressources (doit correspondre à celui créé précédemment)
RESOURCE_GROUP="AKS-Lab-RG"
# Nom du cluster AKS
CLUSTER_NAME="aks-lab-cluster"
# Nombre de nœuds workers dans le cluster
NODE_COUNT=2
# Taille des machines virtuelles pour les nœuds (Standard_DS2_v2 = 2 vCPUs, 7 GiB RAM)
VM_SIZE="Standard_DS2_v2"
# Version de Kubernetes à déployer (doit être supportée dans la région choisie)
K8S_VERSION="1.34.0"

# Crée un cluster AKS avec les options suivantes :
# --resource-group : Groupe de ressources cible
# --name : Nom du cluster
# --node-count : Nombre de nœuds workers
# --node-vm-size : Taille des VMs pour les nœuds
# --enable-managed-identity : Active une identité managée pour le cluster (meilleure pratique pour la sécurité)
# --generate-ssh-keys : Génère automatiquement une clé SSH si elle n'existe pas
# --network-plugin kubenet : Plugin réseau simple pour les tests (alternative : azure)
# --kubernetes-version : Version de Kubernetes à utiliser
# --yes : Confirme automatiquement la création (mode non-interactif)
echo "⚙️ Création du cluster AKS '$CLUSTER_NAME' (version $K8S_VERSION)..."
az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --node-count $NODE_COUNT \
    --node-vm-size $VM_SIZE \
    --enable-managed-identity \
    --generate-ssh-keys \
    --network-plugin kubenet \
    --kubernetes-version $K8S_VERSION \
    --yes

if [ $? -ne 0 ]; then
    echo "❌ Erreur : La création du cluster AKS a échoué."
    exit 1
fi

echo "✅ Cluster AKS '$CLUSTER_NAME' en cours de création (5-10 min)."
echo "   Exécutez './03-connect-and-verify.sh' une fois la création terminée."
```

---

### **3. `03-connect-and-verify.sh`**
```bash
#!/bin/bash
# --------------------------------------------------------------------
# Script : 03-connect-and-verify.sh
# Description : Configure kubectl pour accéder au cluster et vérifie l'état des nœuds.
# Prérequis : Le cluster AKS doit être créé (voir 02-create-aks-cluster.sh).
# --------------------------------------------------------------------

# Nom du groupe de ressources
RESOURCE_GROUP="AKS-Lab-RG"
# Nom du cluster AKS
CLUSTER_NAME="aks-lab-cluster"

# Récupère les identifiants du cluster pour kubectl :
# --resource-group : Groupe de ressources du cluster
# --name : Nom du cluster
# Cette commande met à jour le fichier ~/.kube/config pour permettre à kubectl de communiquer avec le cluster
echo "🔑 Récupération des identifiants pour '$CLUSTER_NAME'..."
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
if [ $? -ne 0 ]; then
    echo "❌ Erreur : Impossible de récupérer les identifiants."
    exit 1
fi

# Vérifie que les nœuds du cluster sont prêts :
# kubectl get nodes : Liste les nœuds et leur statut (doit afficher "Ready")
echo "🖥️ Vérification des nœuds du cluster..."
kubectl get nodes
if [ $? -ne 0 ]; then
    echo "❌ Erreur : Impossible de lister les nœuds."
    exit 1
fi

echo "✅ Les nœuds sont prêts. Exécutez './04-deploy-test-app.sh' pour déployer Nginx."
```

---

### **4. `04-deploy-test-app.sh`**
```bash
#!/bin/bash
# --------------------------------------------------------------------
# Script : 04-deploy-test-app.sh
# Description : Déploie une application Nginx et l'expose via un LoadBalancer.
# Prérequis : kubectl doit être configuré (voir 03-connect-and-verify.sh).
# --------------------------------------------------------------------

# Déploie une application Nginx dans le cluster :
# kubectl create deployment : Crée un déploiement Kubernetes
# --image=nginx:latest : Utilise l'image Docker officielle de Nginx (dernière version)
echo "🚀 Déploiement de l'application Nginx..."
kubectl create deployment nginx-deployment --image=nginx:latest
if [ $? -ne 0 ]; then
    echo "❌ Erreur : Impossible de déployer Nginx."
    exit 1
fi

# Expose le déploiement Nginx via un service de type LoadBalancer :
# kubectl expose deployment : Crée un service pour exposer le déploiement
# --type=LoadBalancer : Type de service qui provisionne un équilibreur de charge externe
# --port=80 : Port sur lequel le service écoute (port HTTP par défaut)
echo "🌐 Exposition du service Nginx via LoadBalancer..."
kubectl expose deployment nginx-deployment --type=LoadBalancer --port=80
if [ $? -ne 0 ]; then
    echo "❌ Erreur : Impossible d'exposer le service."
    exit 1
fi

# Surveille l'attribution de l'IP publique par Azure :
# kubectl get service --watch : Affiche en temps réel l'état du service
# L'IP publique apparaîtra dans la colonne EXTERNAL-IP une fois prête
echo "🕒 Attente de l'IP publique (appuyez sur Ctrl+C pour arrêter une fois l'IP affichée)..."
kubectl get service nginx-deployment --watch
```

---

### **5. `05-cleanup.sh`**
```bash
#!/bin/bash
# --------------------------------------------------------------------
# Script : 05-cleanup.sh
# Description : Supprime le groupe de ressources et toutes les ressources AKS associées.
# Attention : Cette opération est irréversible et supprimera toutes les ressources du groupe.
# --------------------------------------------------------------------

# Nom du groupe de ressources à supprimer
RESOURCE_GROUP="AKS-Lab-RG"

# Supprime le groupe de ressources et toutes ses ressources :
# --name : Nom du groupe de ressources
# --yes : Confirme automatiquement la suppression
# --no-wait : Ne pas attendre la fin de la suppression (s'exécute en arrière-plan)
echo "🗑️ Suppression du groupe de ressources '$RESOURCE_GROUP'..."
az group delete --name $RESOURCE_GROUP --yes --no-wait
if [ $? -ne 0 ]; then
    echo "❌ Erreur : Impossible de supprimer le groupe de ressources."
    exit 1
fi

echo "✅ Nettoyage terminé. Toutes les ressources du groupe '$RESOURCE_GROUP' sont en cours de suppression."
```

---

### **Points Clés Expliqués**
- **`--enable-managed-identity`** :
  Active une **identité managée** pour le cluster AKS, ce qui permet à Azure de gérer automatiquement les identifiants et les permissions, améliorant la sécurité (évite de stocker des secrets statiques).

- **`--network-plugin kubenet`** :
  Utilise le plugin réseau **kubenet**, simple et adapté pour les tests. Pour un environnement de production, le plugin **azure** est recommandé pour une intégration optimale avec les services Azure.

- **`--type=LoadBalancer`** :
  Crée un **équilibreur de charge public** dans Azure, qui attribue une IP publique accessible depuis Internet.

- **`kubectl get nodes`** :
  Vérifie que les nœuds sont prêts (`Ready`), ce qui indique que le cluster est opérationnel.

- **`az group delete --no-wait`** :
  Lance la suppression en arrière-plan, ce qui permet de gagner du temps (la suppression peut prendre plusieurs minutes).

---

### **Comment Utiliser Ces Scripts ?**
1. **Rendez les scripts exécutables** :
   ```bash
   chmod +x 0*.sh
   ```
2. **Exécutez-les dans l'ordre** :
   ```bash
   ./01-prepare-environment.sh
   ./02-create-aks-cluster.sh
   ./03-connect-and-verify.sh
   ./04-deploy-test-app.sh
   ```
3. **Nettoyez après utilisation** :
   ```bash
   ./05-cleanup.sh
   ```

---