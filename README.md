# Déploiement d'un Cluster Kubernetes Automatisé avec Ansible

## Description du Projet
Ce projet a pour objectif de déployer de bout en bout un cluster Kubernetes (1 Control Plane et 2 Worker Nodes) de manière entièrement automatisée. 
L'infrastructure repose sur **4 machines virtuelles Ubuntu** (hébergées via VMware Workstation). La configuration des nœuds est entièrement gérée par **Ansible**. L'architecture respecte les bonnes pratiques en séparant le nœud d'administration (Bastion/Ansible) du Control Plane Kubernetes. Le cluster utilise **containerd** comme runtime de conteneurs et **Calico** comme CNI.

## Architecture

L'infrastructure se compose de 4 machines virtuelles interconnectées sur un réseau local (mode pont) :

| Rôle | Hostname | IP | Rôle détaillé |
| :--- | :--- | :--- | :--- |
| **Node Ansible (Bastion)** | `k8s-ansible` | `192.168.1.12` | Machine d'administration exécutant les playbooks Ansible. |
| **Control Plane** | `k8s-master` | `192.168.1.93` | Initialisation du cluster, API Server. |
| **Worker Node 1** | `k8s-worker1` | `192.168.1.16` | Exécution des charges de travail (Pods). |
| **Worker Node 2** | `k8s-worker2` | `192.168.1.17` | Exécution des charges de travail (Pods). |

*   **Hyperviseur** : VMware Workstation (sur hôte Windows)
*   **Outil de configuration** : Ansible (exécuté depuis `k8s-ansible`)
*   **Runtime** : Containerd
*   **Réseau (CNI)** : Calico (CIDR Pods : `10.244.0.0/16`)

## Structure du Projet (Infrastructure as Code)

Le projet est structuré de manière modulaire avec des rôles Ansible pour garantir la réutilisabilité du code :

```text
k8s-ansible/
├── ansible.cfg              # Configuration d'Ansible
├── inventory.ini            # Inventaire (Hôtes, IPs)
├── site.yml                 # Playbook principal (orchestration)
├── group_vars/
│   └── all.yml              # Variables globales
└── roles/
    ├── common/              # Prérequis (swap, modules noyau, sysctl, /etc/hosts)
    ├── containerd/          # Installation et configuration du runtime
    ├── kubeadm/             # Installation de kubeadm, kubelet, kubectl
    ├── master/              # Initialisation (kubeadm init) et déploiement de Calico
    └── workers/             # Jonction des workers (kubeadm join)

```
Procédure de Déploiement
1. Prérequis
4 VMs Ubuntu configurées avec accès SSH par clé.

2. Configuration de l'inventaire (inventory.ini)
L'inventaire définit les nœuds et les groupes cibles :
```
[master]
k8s-master ansible_host=192.168.1.93 ansible_user=liel

[workers]
k8s-worker1 ansible_host=192.168.1.16 ansible_user=liel
k8s-worker2 ansible_host=192.168.1.17 ansible_user=liel

[k8s_cluster:children]
master
workers
Ansible installé sur la machine k8s-ansible.
```
3. Exécution du Playbook
Depuis le dossier ~/k8s-ansible sur la machine k8s-ansible, lancez le déploiement complet
```
# Tester la connectivité
ansible -i inventory.ini all -m ping

# Vérifier la syntaxe
ansible-playbook -i inventory.ini site.yml --syntax-check

# Lancer le déploiement
ansible-playbook -i inventory.ini site.yml
```

Vérification du Cluster
Connectez-vous au master pour vérifier l'état de santé du cluster :
```
ssh liel@192.168.1.93

# Vérifier que les 3 nœuds sont "Ready"
kubectl get nodes -o wide

# Vérifier que les pods système (Calico, CoreDNS, kube-proxy) sont "Running"
kubectl get pods -A
```

Test de Communication Réseau (Validation)
Nous avons validé le bon fonctionnement du DNS interne et du réseau Calico en déployant un service Nginx et en l'interrogeant depuis un pod temporaire :
```
# Créer un déploiement Nginx
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort

# Tester la résolution DNS et le routage inter-pods
kubectl run curl-test --image=curlimages/curl --restart=Never --rm -it -- curl -s http://nginx
```
Résultat : Le code HTML par défaut de Nginx s'affiche, prouvant que Calico et CoreDNS fonctionnent correctement.

Auteur
Nom : Liel Makosso

Dépôt GitHub : https://github.com/LielMakosso/k8s-ansible
