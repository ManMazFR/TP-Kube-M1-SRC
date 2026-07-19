# Projet — phpMyAdmin + MySQL sur Kubernetes (kind)

Stack MySQL 8 + phpMyAdmin déployée en YAML sur un cluster kind 3 nœuds.
Sécurité : Secret (mot de passe), ConfigMap (config PMA), 5 NetworkPolicies
(modèle deny-all), persistance via PVC, probes liveness/readiness.

Namespace : `projet-pma` · phpMyAdmin en `replicas: 1` (sessions PHP sur disque local).

## 1. Créer le cluster (3 nœuds + Calico)

Calico est nécessaire : le CNI par défaut de kind n'applique pas les NetworkPolicy
sur cette version. Le fichier `kind-cluster.yaml` est à la racine du dépôt.

    kind create cluster --config ../kind-cluster.yaml --name esgi
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/calico.yaml
    kubectl wait -n kube-system --for=condition=ready pod -l k8s-app=calico-node --timeout=180s
    kubectl get nodes        # 3 noeuds Ready

## 2. Déployer

    kubectl apply -f .
    kubectl -n projet-pma get pods -w      # attendre Running, Ctrl-C

Les manifests sont préfixés 00 à 08 pour garantir l'ordre d'application
(namespace en premier, NetworkPolicies en dernier). Ré-appliquable à l'infini.

## 3. Tester

### 3.1 Accès web — capture `01-pma-connected.png`

    curl -I -m 5 http://localhost:8080     # doit renvoyer HTTP/1.1 200 OK

Ouvrir http://localhost:8080 puis se connecter :
serveur = `mysql` · utilisateur = `root` · mot de passe = `ceciestmonmdp!`
→ capture de l'accueil phpMyAdmin avec une base visible.

### 3.2 Persistance MySQL — capture `persistance.png`

Dans phpMyAdmin, créer la base `test_persistance` puis, onglet SQL :

    CREATE TABLE preuve (id INT PRIMARY KEY, note VARCHAR(50));
    INSERT INTO preuve VALUES (1, 'avant delete pod');
    SELECT * FROM preuve;

Détruire le pod MySQL et attendre le nouveau :

    kubectl -n projet-pma delete pod -l app=mysql
    kubectl -n projet-pma get pod -l app=mysql -w    # attendre 1/1 Running, Ctrl-C

Recharger phpMyAdmin et rejouer `SELECT * FROM preuve;`
→ la ligne est toujours là : les données vivent sur le PVC, pas dans le conteneur.

### 3.3 NetworkPolicies — capture `networkpolicy-blocked.png`

Chemin autorisé (phpMyAdmin porte le label `app=phpmyadmin`) :

    kubectl -n projet-pma exec deploy/phpmyadmin -- \
      bash -c '(echo > /dev/tcp/mysql/3306) && echo "PMA -> MySQL OK"'

Chemin bloqué (pod tiers sans le bon label) :

    kubectl -n projet-pma run hacker --rm -it --image=alpine -- sh
      apk add --no-cache netcat-openbsd
      nc -vz -w 3 mysql 3306        # -> timed out (bloqué)
      exit

→ capture du `timed out`. Même cible, deux pods, deux résultats : seul le label change.

### 3.4 Kite (GUI cluster) — capture `kite.png`

    kubectl apply -f https://raw.githubusercontent.com/kite-org/kite/main/deploy/install.yaml
    kubectl get pod -A | grep kite                  # repérer le namespace réel
    kubectl port-forward -n kube-system --address 0.0.0.0 svc/kite 9000:8080

Ouvrir http://localhost:9000 → compte admin → type `in-cluster` → namespace
`projet-pma` → menu Pods. Capture des pods mysql + phpmyadmin. Ctrl-C pour arrêter.

### 3.5 Idempotence

    kubectl apply -f .     # 2e passage : tout en "unchanged", aucune erreur

## 4. NetworkPolicies déployées

- `deny-all` — bloque tout le trafic entrant et sortant du namespace
- `allow-internal-dns` — autorise tous les pods à joindre CoreDNS (port 53)
- `pma-egress-to-mysql` — phpMyAdmin peut sortir vers MySQL:3306
- `mysql-ingress-from-pma` — MySQL n'accepte que phpMyAdmin sur 3306
- `pma-ingress-nodeport` — autorise le trafic NodePort entrant vers phpMyAdmin:80

> La 5e policy a été ajoutée après test : avec un deny-all strict, le trafic
> NodePort n'atteint pas phpMyAdmin et la page est injoignable depuis le navigateur.

## 5. Fichiers

| Fichier | Rôle |
|---|---|
| `../kind-cluster.yaml` | cluster 3 nœuds, Calico, port 30080 -> 8080 |
| `00-namespace.yaml` | namespace `projet-pma` |
| `01-mysql-secret.yaml` | mot de passe root (base64, jamais en clair) |
| `02-mysql-pvc.yaml` | volume persistant 1 Gi (RWO) |
| `03-mysql-deployment.yaml` | MySQL 8, secretKeyRef, probes, strategy Recreate |
| `04-mysql-service.yaml` | Service ClusterIP `mysql` (DNS interne) |
| `05-phpmyadmin-configmap.yaml` | `PMA_ARBITRARY=1` |
| `06-phpmyadmin-deployment.yaml` | phpMyAdmin, replicas 1, envFrom, probes |
| `07-phpmyadmin-service.yaml` | Service NodePort 30080 |
| `08-networkpolicy.yaml` | les 5 NetworkPolicies |
