# msc-de1-distributed-systems-docker-k8s

Containerisation, sécurisation, publication et orchestration locale d'une application Flask existante, dans le cadre du module *Distributed Systems* (MSc DE1).

## 1. Objectif et architecture

Ce projet prend l'application starter [UBC Flask Sample App](https://github.com/ubc/flask-sample-app) — une API REST minimale en Flask, sans Docker — et construit autour :

- une image Docker de production (utilisateur non-root, filesystem en lecture seule, health check) ;
- un fichier Docker Compose pour l'exécution locale durcie ;
- un scan de vulnérabilités (Trivy) et un SBOM (Syft) ;
- une publication sur Docker Hub ;
- un cluster Kubernetes local (kind, 1 control-plane + 2 workers) exécutant 2 replicas de l'application, avec probes, limites de ressources, NetworkPolicy et contexte de sécurité.

**Architecture** : `Client → kubectl port-forward → Service (ClusterIP) → 2 pods Flask/gunicorn (répartis sur 2 workers)`.

Lien vers l'application d'origine : https://github.com/ubc/flask-sample-app

## 2. Prérequis

- Python 3.13 (le projet a été développé et testé avec cette version)
- Docker Desktop
- kind
- kubectl
- (optionnel) Trivy et Syft pour reproduire le scan de sécurité

## 3. Exécuter l'application d'origine en local (sans Docker)

```bash
git clone https://github.com/aniasadoudi000-ops/distributed-systems-docker-k8s.git
cd distributed-systems-docker-k8s
python3 -m venv venv
source venv/bin/activate      # Windows : venv\Scripts\activate
pip install -r requirements.txt
python run.py
```

L'application écoute alors sur `http://127.0.0.1:5000` (uniquement en local, voir la section Sécurité et limites connues).

Tester les routes :
```bash
curl -i http://localhost:5000/
curl -i http://localhost:5000/items
curl -i -X POST http://localhost:5000/items -H "Content-Type: application/json" -d '{"name":"test"}'
curl -i http://localhost:5000/items/0
```

Lancer les tests unitaires :
```bash
python -m unittest discover tests
```

Les preuves de cette étape (sorties de commandes) sont dans `evidence/00-*` à `evidence/09-*`.

## 4. Construire et exécuter l'image Docker

```bash
docker build -t aniasadoudi/msc-de1-flask-app:1.0.0 -t aniasadoudi/msc-de1-flask-app:latest .
docker run -d --name flask-app -p 5000:5000 aniasadoudi/msc-de1-flask-app:1.0.0
curl -i http://localhost:5000/
docker logs flask-app
docker exec flask-app id          # confirme l'exécution non-root (uid=10001)
docker ps                         # confirme le statut "healthy"
docker stop flask-app && docker rm flask-app
```

## 5. Exécuter avec Docker Compose

```bash
docker compose up -d --build
docker compose ps
curl -i http://localhost:5000/
docker compose down
```

## 6. Docker Hub

Image publique : **https://hub.docker.com/r/aniasadoudi/msc-de1-flask-app**

Tags publiés : `1.0.0`, `1.1.0` (version de démonstration pour le rolling update), `latest`.

L'image déployée sur Kubernetes est `aniasadoudi/msc-de1-flask-app:1.0.0`.

Vérification après publication :
```bash
docker pull aniasadoudi/msc-de1-flask-app:1.0.0
docker run --rm -p 5000:5000 aniasadoudi/msc-de1-flask-app:1.0.0
curl -i http://localhost:5000/
```

## 7. Créer le cluster kind

```bash
kind create cluster --config kind/kind-config.yaml
kubectl config current-context   # doit afficher kind-msc-de1
kubectl get nodes                # 1 control-plane + 2 workers
```

## 8. Déployer les manifests Kubernetes

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/
kubectl -n msc-de1-project rollout status deploy/flask-app
kubectl -n msc-de1-project get pods -o wide
```

## 9. Accéder à l'application et la tester

```bash
kubectl -n msc-de1-project port-forward svc/flask-app 8080:80
```
Dans un autre terminal :
```bash
curl -i http://localhost:8080/
curl -i http://localhost:8080/items
```

**Note** : le port-forward est attaché à un pod précis, pas au Service. Il se coupe automatiquement quand ce pod est remplacé (rollout, rollback, suppression manuelle) et doit être relancé.

## 10. Supprimer le cluster local

```bash
kind delete cluster --name msc-de1
```

## 11. Décisions de sécurité et limites connues

**Décisions appliquées** (cohérentes entre Docker, Compose et Kubernetes) :
- exécution en utilisateur non-root (uid/gid 10001) dans l'image, Compose et Kubernetes ;
- filesystem racine en lecture seule (`read_only` / `readOnlyRootFilesystem`), avec `/tmp` monté en `tmpfs`/`emptyDir` pour les besoins de gunicorn ;
- `no-new-privileges` (Compose) et `allowPrivilegeEscalation: false` (Kubernetes) ;
- toutes les capabilities Linux supprimées (`cap_drop: ALL` / `capabilities.drop: [ALL]`) ;
- limites CPU/mémoire définies à tous les niveaux ;
- `seccompProfile: RuntimeDefault` dans Kubernetes ;
- aucun secret dans l'image, le Dockerfile ou les fichiers commités ;
- `NetworkPolicy` restreignant l'ingress vers les pods de l'application au namespace `msc-de1-project`.

**Limites connues** :
- **État en mémoire** : l'application starter stocke les items dans une liste Python en mémoire (`items = []`), sans base de données. Avec 2 replicas, chaque pod a son propre état : un `POST /items` sur un pod n'est pas visible depuis l'autre. Ce comportement n'a pas été modifié pour respecter la consigne de ne pas redessiner l'application. **Amélioration recommandée pour la production** : externaliser l'état dans une base de données partagée (PostgreSQL, Redis).
- **NetworkPolicy et kind** : le CNI par défaut de kind (kindnet) n'applique pas nécessairement les `NetworkPolicy`. Le manifeste est fourni et documente l'intention d'isolation réseau ; son application effective nécessiterait un CNI compatible tel que Calico ou Cilium.
- **Scan de vulnérabilités** : voir `security/vulnerability-scan.txt` — 0 CRITICAL, 44 HIGH restants (majoritairement des paquets système Debian sans correctif amont disponible à ce jour ; deux paquets Python hérités de l'image de base, non utilisés directement par l'application). Détail dans le rapport PDF, section 3.