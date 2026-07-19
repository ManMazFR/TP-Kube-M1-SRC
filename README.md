# Projet — phpMyAdmin + MySQL sur Kubernetes (kind)

Stack MySQL 8 + phpMyAdmin déployée en YAML sur un cluster kind 3 nœuds.
Sécurité : Secret (mot de passe), ConfigMap (config PMA), NetworkPolicy (isolation MySQL),
persistance via PVC, 1 replicas phpMyAdmin.
