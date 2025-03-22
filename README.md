# Mise en place d'un cluster Kubernetes via playbook ANSIBLE

**CONSEIL :** Après le playbook `install-k8s.yaml`, faire une snapshot avant l'initialisation du cluster.

## Architecture

![Architecture](img/mermaid-diagram-2025-03-20-153759.png)

### Dimensionnement

Pour un cluster "Production Ready" sans stockage HA... :

- 6 VM (4vCPU, 8Go RAM, Stockage 20Go pour du PoC)

Pour un PoC (master - worker) :

- 2 VM (4vCPU, 8Go RAM, Stockage 20Go pour du PoC)

Pour un PoC - ressources faibles :

- 2 VM (2vCPU, 4Go RAM, 20Go pour du PoC)

> Peut être "ric rac" au niveau de la RAM.

### Plan d'addressage

- Il faudra une @IP par nœud.  
- Il faudra une @IP dédiée à la VIP ([IP Failover](https://www.aukfood.fr/ip-failover-avec-hearbeat-et-pacemaker/#:~:text=On%20appelle%20IP%20flottante%20une,parle%20aussi%20d%27IP%20failover.)), qui servira à accéder au control-plan.  
- Il faudra une @IP dédiée à l'[Ingress Traefik](https://kubernetes.io/docs/concepts/services-networking/ingress/), qui permettra de créer des règles (host, vpath) pour exposer des pods vers l'extérieur.  

## Avant exécution - mise à jour plan addressage

Il faudra définir dans votre plan réseau une VIP, par exemple dans `172.16.20.0/24`, avec une VIP à `172.16.20.99`, qui permettra d'accéder au control-plan.  

Si le plan d'adressage/et ou la VIP est différents :  

Éditez les fichiers suivants :  

- `roles/keepalived/templates/backup.cfg`  
- `roles/keepalived/templates/master.cfg`  

Au niveau de :  

```txt
    virtual_ipaddress {
        172.16.20.99
    }
```  

> Remplacez par votre IP du plan d'adressage.  

Dans le fichier `init-k8s.yaml`  

Au niveau de :  

```yaml
    - name: Initialiser le cluster Kubernetes
      ansible.builtin.command: kubeadm init --control-plane-endpoint "172.16.20.99:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
      when: not kube_cluster.stat.exists
```  

Remplacez `172.16.20.99` par la VIP pour le control-plan.  

Au niveau de :

```yaml
- name: Installation Traefik
  hosts: master[0]
  become: true
  tasks:
    - name: Create a IPPool
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: metallb.io/v1beta1
          kind: IPAddressPool
          metadata:
            namespace: metallb-system
            name: ippool
          spec:
            addresses:
              - 172.16.20.100/32
```

Remplacez par `172.16.20.100` par l'@IP pour l'Ingress Traefik.

### Entrées DNS (hosts / server DNS)

Il faudra déclarer les entrées suivantes avec @IP de l'Ingress Traefik choisie :

```txt
monitoring.kube.lan A 172.16.20.100
prometheus.kube.lan A 172.16.20.100
```

Remplacez par `172.16.20.100` par l'@IP pour l'Ingress Traefik.

### Inventaire ansible

Editer le fichier d'inventaire : `inventory/inventaire.yaml`

1. Mettre à jour les @IP des noeuds `ansible_host: @IP`
2. Mettre à jour l'utilisateur `ansible_user: user` => **Cet utilisateur doit pouvoir faire des commandes `sudo` sans mot de passe. Sinon ajouter `-k` ou `-K` qui permet de donner le mot de passe pour devenir sudoers.
3. Ajouter ou supprimer les hosts nécessaires.

## Exécution  

### Copier la clé publique vers le bastion

Générer un couple de clé publique privé :

`ssh-keygen -t rsa -b 2048` -> `ENTER, ENTER, ENTER...`

Copier la clé publique sur chaque noeud du cluster :

```bash
ssh-copi-id user@IP_NOEUD
```

### ANSIBLE

Depuis `bastion-ansible` :  

1. SSH sur chaque VM cible (ajout du fingerprint SSH).

Pour lancer l'installation de Kubernetes (pas initialisation cluster):  

```bash
ansible-playbook -i inventory/inventaire.yaml install-k8s.yml
```  

Pour initialiser le cluster K8s :  

```bash
ansible-playbook -i inventory/inventaire.yaml init-k8s.yaml
```

> Il initialise le cluster, puis installe un pluging réseau (Calico), installe Traefik avec la configuration réseau.
> A ce niveau sur votre vm `ansible-bastion` vous aurez le fichier `Kubeconfig-master-00` (équivalent)

Pour installer la stack monitoring (Prometheus, Loki, Grafana)

```bash
ansible-playbook -i inventory/inventaire.yaml install-monitoring.yaml
```

## Vérification  

Il faudra vous connecter sur `master-00` en `root`, puis exécuter :  

```bash
kubectl get nodes
```  

Sortie attendue :  

```txt
master-00   Ready    control-plane   2m29s   v1.29.14
master-01   Ready    control-plane   23s     v1.29.14
master-02   Ready    control-plane   21s     v1.29.14
worker-00   Ready    <none>          71s     v1.29.14
worker-01   Ready    <none>          72s     v1.29.14
worker-02   Ready    <none>          71s     v1.29.14
```

### Kubectl

Sur windows :

- Télécharger le binaire kubectl
- Placer le binaire dans un dossier quelquonque (C:/BIN/kubectl)
- Ajouter `C:/BIN/kubectl` dans le $PATH de `Variable Envrionnement Systems`
- Copier le fichier `kubeconfig-master-00` vers `C:\Users\Timai\.kube\config` (fichier sans extension)

Sur Linux :

```bash
curl -LO https://dl.k8s.io/release/v1.32.0/bin/linux/amd64/kubectl
sudo mv kubectl /usr/local/bin
sudo chmod +x /usr/local/bin/kubectl
```

Faire l'export :

```bash
export K8S_AUTH_KUBECONFIG=/chemin/vers/kubeconfig-master-00
```

Vérification : `kubectl version`

## Interraction avec le cluster

Faire les exports (si depuis la VM master-00):

```bash
export K8S_AUTH_KUBECONFIG=/root/.kube/config
```

### Récupérer mot de passe Grafana (depuis master-00)
 
Le login est : admin

```bash
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

## Grafana 

Les URL de datatsources sont :

- prometheus: http://prometheus-server/
- Loki : http://loki:3100/

La récupération de l'information se fait avec la commande suivante :

`kubectl get svc -n monitoring`

- svc : service Kubernetes
- -n moniroting : Spécifique de récupérer les objets dans le namespace `monitoring`

## Prometheus récupération des données HORS-CLUSTER 

Editer le fichier `install-monitoring.yaml` :

```yaml
    - name: Deploy latest version of Prometheus chart inside namespace and create it
      kubernetes.core.helm:
        name: prometheus
        chart_ref: prometheus-community/prometheus
        release_namespace: monitoring
        create_namespace: true
        values:
          server:
            persistentVolume:
              enabled: false
          serverFiles:
            prometheus.yml:
              rule_files:
                - /etc/config/recording_rules.yml
                - /etc/config/alerting_rules.yml
              ## Below two files are DEPRECATED will be removed from this default values file
                - /etc/config/rules
                - /etc/config/alerts
          
              scrape_configs:
                - job_name: prometheus
                  static_configs:
                    - targets:
                      - localhost:9090
                - job_name: prometheus-node-exporter-vm-deportee 
                  static_configs: 
                    - targets:
                      - 172.16.20.36:9100
```

La section `values` est le contenu du fichier YAML `prometheus.yml`.

Rejouer le playbook :

```bash
ansible-playbook -i inventory/inventaire.yaml install-monitoring.yaml
```

Ressource : `stephane-robert.info` : https://blog.stephane-robert.info/docs/observer/metriques/prometheus/

## Bonus - Ajout d'un noeud dans le cluster

0. Déclarer le/noeuds dans l'inventaire `inventory/inventaire.yaml` (**Un noeud ne peut être à la fois control-plan et worker**)
1. Avoir exécuter le playbook `install-k8s.yaml` sur le noeud
2. Jouer le playbook - `add-node-k8s.yaml`

`ànsible-playbook -i inventory/inventaire.yaml add-nodes-k8s.yaml`

# K9s - Terminal User Interface pour Kubernetes (exploration des objets Kubernetes)

L'installation est similaire à `kubectl`

Pour interragir avec le cluster en spécifiant le `kubeconfig` :

```bash
k9s --kubeconfig /chemin/vers/le/kubeconfig
```

https://enix.io/en/blog/k9s/

https://blog.stephane-robert.info/docs/conteneurs/orchestrateurs/outils/k9s/

# Installation manuelle monitoring avec Dashboard déjà fait 

Ne pas avoir exécuter le playbook : `install-monitoring.yaml`.

Accès kubectl (master-00 ou depuis le poste de travail avec le kubeconfig-master-00).

HELM doit être installer sur la machine où les commandes sont exécutées.

Ajouter le dépôt :

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Créer le namespace et le définir comme contexte par défaut :

```bash
kubectl create ns monitoring
kubectl config set-context --current --namespace=monitoring
```

Installation via HELM :

```bash
helm install prometheus prometheus-community/kube-prometheus-stack
```

Créer une règle dans notre Ingress Traefik `monitoring-ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: monitoring.kube.lan
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-grafana
            port:
              number: 80
```

Puis faire :

```bash
kubectl apply -f monitoring-ingress.yaml
```

Login : `admin`

Password : `kubectl get secret grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`
