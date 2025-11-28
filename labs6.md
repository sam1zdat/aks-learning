---

**Exécuter des conteneurs à l’aide d’Azure Container Instances**

### Introduction
Dans ce lab, vous allez vous exercer avec Azure Container Instances et Azure Container Registry. Vous commencerez par utiliser Azure Cloud Shell pour créer un Azure Container Registry. Ensuite, vous ajouterez une image Docker simple au registre. Une fois l’image du conteneur poussée vers le registre, vous passerez au portail Azure pour exécuter une instance du conteneur dans Azure Container Instances. 

---

### Solution

#### Connexion
Connectez-vous au portail Azure en utilisant les identifiants fournis sur la page du lab. Assurez-vous d’utiliser une fenêtre de navigation privée ou incognito pour vous assurer d’utiliser le compte du lab plutôt que le vôtre.

---

#### Préparation (Configuration de Cloud Shell)
1. Cliquez sur l’icône **Cloud Shell** (>_) en haut à droite du portail Azure.
2. Lorsque vous y êtes invité, sélectionnez un shell **Bash** (plutôt que PowerShell).
3. Choisissez **"Monter un compte de stockage"**.
4. Sélectionnez l’abonnement du lab et cliquez sur **Appliquer**.
5. Choisissez le compte de stockage déjà déployé pour vous.
6. Sélectionnez le groupe de ressources existant.
7. Créez un nouveau partage de fichiers et nommez-le **"fileshare"** (en minuscules).
8. Cliquez sur **Appliquer**.
9. Attendez que l’invite de commande apparaisse.
   *Remarque : Cette opération peut prendre quelques instants.*

---

#### Créer une instance d’Azure Container Registry (ACR)
1. Depuis l’invite de commande, listez les groupes de ressources :
   ```bash
   az group list
   ```
2. Copiez le nom du groupe de ressources dans le champ **name** pour une utilisation ultérieure.
3. Créez des variables d’environnement pour le groupe de ressources et Azure Container Registry (ACR). Remplacez **<RESOURCE_GROUP_NAME>** par le nom que vous venez de copier :
   ```bash
   rg=<RESOURCE_GROUP_NAME>
   name=acrlabdemo
   acr="$name$RANDOM"
   ```
4. Créez un ACR en utilisant les variables d’environnement :
   ```bash
   az acr create --resource-group $rg --name $acr --sku Basic
   ```
   *Remarque : La création peut prendre plusieurs minutes.*
5. Une fois l’ACR créé, activez un utilisateur administrateur pour le registre :
   ```bash
   az acr update -n $acr --admin-enabled true
   ```
   *Remarque : Il peut falloir une ou deux minutes supplémentaires pour que l’ACR termine son déploiement. Si vous recevez une erreur indiquant que l’ACR n’existe pas, attendez quelques minutes et réessayez.*

---

#### Construire et pousser une image de conteneur vers l’ACR
2. Créez un fichier **Dockerfile** :
   ```bash
   echo FROM mcr.microsoft.com/hello-world > Dockerfile
   ```
3. Construisez et poussez l’image vers l’ACR en utilisant le Dockerfile :
   ```bash
   az acr build --image sample/hello-world:v1 --registry $acr --file Dockerfile .
   ```
   *Remarque : Le point après l’argument `--file` n’est pas une erreur. Il est nécessaire pour indiquer que l’emplacement source du fichier est à la racine. La construction de l’image peut prendre plusieurs minutes.*
4. Minimisez **Azure Cloud Shell**.
5. Dans le portail, vous devriez être sur la page **Vue d’ensemble** du groupe de ressources. Trouvez le registre de conteneurs listé sous l’onglet **Ressources** et sélectionnez-le pour accéder au registre. Si vous ne le voyez pas, cliquez sur **Actualiser**.
6. Dans le menu de navigation de gauche, cliquez sur **Services**, puis sélectionnez **Dépôts**. Vous devriez maintenant voir l’image **sample/hello-world** affichée.

---

#### Déployer une image vers Azure Container Instances (ACI)
1. Retournez au groupe de ressources.
2. Cliquez sur **Créer**.
3. Dans la barre de recherche, tapez et sélectionnez **Container Instances**.
4. Cliquez sur **Créer**.
5. Définissez les valeurs suivantes :
   - **Groupe de ressources** : Sélectionnez le groupe de ressources existant (ne créez pas de nouveau groupe).
   - **Nom du conteneur** : Entrez un nom unique.
   - **Région** : Sélectionnez **(États-Unis) Est des États-Unis**.
   - **Source de l’image** : Sélectionnez **Azure Container Registry**.
   *Remarque : Les champs **Registre** et **Image** doivent être préremplis avec l’Azure Container Registry que vous avez créé et la seule image stockée dans ce registre.*
6. Cliquez sur **Vérifier + créer**.
7. Cliquez sur **Créer**.
   *Remarque : Le déploiement de la ressource peut prendre plusieurs minutes.*
8. Une fois déployé, cliquez sur **Aller à la ressource**.
9. Cliquez sur **Démarrer** > **Oui**.
   *Remarque : Il peut falloir 1 à 2 minutes pour que la ressource démarre. Vous pouvez vérifier les notifications pour voir quand l’instance de conteneur est démarrée avec succès. Si vous obtenez un message d’erreur indiquant que le conteneur est "toujours en transition", attendez quelques minutes et réessayez.*
10. Dans le menu de navigation de gauche, sélectionnez **Conteneurs**. Pendant que votre conteneur est en cours d’exécution, vous pouvez voir ses logs dans l’onglet **Logs**. Lorsque l’état de votre conteneur est **Terminé**, vous pouvez vérifier ses logs depuis l’interface Azure CLI avec la commande :
    ```bash
    az container logs --resource-group $rg --name myacicontainer
    ```

---
