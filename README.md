# Projet — phpMyAdmin + MySQL sur Kubernetes (kind)

Stack MySQL 8 + phpMyAdmin déployée en YAML sur un cluster kind 3 nœuds.
Sécurité : Secret (mot de passe), ConfigMap (config PMA), NetworkPolicy (isolation MySQL),
persistance via PVC, 1 replicas phpMyAdmin.

## 1. Créer le cluster (3 nœuds + Calico)

Calico est indispensable : le CNI par défaut de kind (kindnet) n'applique pas les NetworkPolicy.

    kind create cluster --config kind-cluster.yaml --name esgi
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/calico.yaml
    kubectl wait -n kube-system --for=condition=ready pod --selector=k8s-app=calico-node --timeout=180s
    kubectl get nodes        # 3 noeuds Ready

## 2. Déployer

    kubectl apply -f .
    kubectl -n projet-pma get pods -w

Si le 1er apply signale "namespace projet-pma not found", relancer `kubectl apply -f .` (idempotent).

## 3. Tester

1. Ouvrir http://localhost:8080 (login phpMyAdmin)
2. Serveur = mysql, utilisateur = root, mot de passe = ceciestmonmdp!
3. Créer une base de test, y insérer des données
4. Persistance : `kubectl -n projet-pma delete pod -l app=mysql` -> le pod redémarre, données conservées

## 4. NetworkPolicy

Les communications réseau sont sécurisées à l'aide de plusieurs NetworkPolicies :

- deny-all : bloque tout le trafic par défaut.
- allow-internal-dns : autorise les requêtes DNS vers CoreDNS.
- pma-egress-to-mysql : autorise phpMyAdmin à communiquer avec MySQL sur le port 3306.
- mysql-ingress-from-pma : autorise uniquement phpMyAdmin à accéder à MySQL.

## Fichiers

- kind-cluster.yaml : cluster 3 noeuds, Calico, port 30080->8080
- namespace / secret / pvc / mysql-deploy / mysql-svc
- pma-configmap / pma-deploy / pma-svc (NodePort) / networkpolicy
