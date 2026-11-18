# Rapport CI/CD – Legeni WebApp

## Architecture

Le dépôt GitHub est la source de vérité. Jenkins exécute les contrôles sur les
branches et les pull requests : tests avec couverture minimale de 80 %, flake8,
Hadolint et vérification de syntaxe Ansible en parallèle. Une image versionnée
`<VERSION>-<BUILD_NUMBER>` est construite et scannée par Trivy. Seule `main`
peut publier l’image, déployer en staging et, après approbation manuelle,
déployer en production.

Ansible applique les mêmes ressources dans `legeni-staging` et `legeni-prod`.
Chaque environnement possède son ConfigMap, son Secret Redis, son StatefulSet
Redis avec PVC, son Deployment web, son Service ClusterIP et son Ingress.
Staging utilise deux réplicas et `staging.legeni.local`; la production en
utilise trois et `legeni.local`. L'application utilise les probes `/ready` et
`/health`, des requests/limits et une stratégie RollingUpdate avec
`maxUnavailable: 0`. Le playbook attend Redis et le rollout web, valide la
version via le Service, puis annule le Deployment si le rollout ou le smoke
test échoue.

## Déroulement du pipeline

1. Les quatre contrôles qualité tournent en parallèle. Le contrôle Ansible
   remplace temporairement le Vault chiffré par des variables factices pour
   `--syntax-check`; il ne demande donc aucun secret aux builds de PR.
2. Docker construit une image portant le tag dérivé de `VERSION` et du numéro
   Jenkins. Trivy bloque les vulnérabilités HIGH/CRITICAL non corrigées.
3. Le push, le déploiement staging, l'approbation et le déploiement production
   sont réservés à `main`. Aucun tag `latest` n'est publié.
4. En cas d'échec, la console Jenkins signale le build. Un credential Secret
   text de webhook Discord peut être sélectionné par le paramètre
   `NOTIFICATION_CREDENTIAL_ID`.

## Credentials Jenkins

Les credentials suivants ne sont pas stockés dans Git :

| ID | Type Jenkins | Usage |
|---|---|---|
| `github-api` | Username with password | Découverte GitHub et statuts de commit |
| `kubeconfig` | Secret text | Kubeconfig Minikube avec endpoint joignable depuis Jenkins |
| `ansible-vault-pass` | Secret text | Déchiffrement du Vault au déploiement |
| `registry-credentials` | Username with password | Push Docker Hub (à créer avec un access token d'écriture) |
| `notification-webhook` | Secret text, facultatif | Webhook Discord si `NOTIFICATION_CREDENTIAL_ID` vaut cet ID |

Les deux credentials Kubernetes/Ansible et le credential GitHub sont présents
dans l'instance Jenkins locale. `registry-credentials` n'a pas pu être créé :
aucun access token Docker Hub n'est configuré sur la machine. Le paramètre
`DOCKERHUB_ID` (par défaut `legeni07`) doit correspondre au nom d'utilisateur
de ce credential et au propriétaire de l'image `legeni-webapp`.

## Configuration Multibranch

Créer dans Jenkins un item **Multibranch Pipeline** avec la source GitHub
`Willoxw/legeni-webapp`, credential `github-api`, découverte des branches et
des PR d'origine, et `Jenkinsfile` comme Script Path. Un scan périodique
(par exemple toutes les cinq minutes) sert de repli au webhook.

Pour le déclenchement instantané, il faut une URL Jenkins joignable depuis
GitHub et un webhook `push`/`pull_request`. Le Jenkins local est derrière WSL;
aucun tunnel public n'est configuré. Le jeton GitHub disponible ne possède pas
le scope `admin:repo_hook`, donc le webhook n'a pas été créé.

Après un build Jenkins réussi sur une PR, rendre son contexte de statut
obligatoire dans la protection de `main`. La règle actuelle exige déjà une PR,
mais aucun statut Jenkins n'est encore requis.

## Vérifications réalisées

- Tests : **6 réussis**, couverture **83,64 %** (seuil 80 %).
- `flake8` : réussi.
- Hadolint : réussi sans diagnostic.
- Docker : image locale `legeni-webapp:1.0.0-ci` construite.
- Trivy : **0 HIGH, 0 CRITICAL** pour cette image.
- Jenkins Pipeline Model Converter : Jenkinsfile valide.
- Ansible : syntax-check de `deploy.yml` réussi.
- Staging : Redis `1/1`, web `2/2`, Ingress et PVC Redis disponibles; smoke
  test avec la version `1.0.0-ci` réussi.
- Idempotence : deux exécutions Ansible successives avec `changed=0`,
  `failed=0`.
- Rollback : un tag d'image inexistant a fait échouer le déploiement; le
  playbook a restauré `legeni-webapp:1.0.0-ci`, puis la version précédente a
  répondu au smoke test.

## Actions restantes avant la démonstration complète

1. Ajouter `registry-credentials` avec un access token Docker Hub en écriture.
2. Créer/configurer le job Multibranch et exécuter un build de PR. Le push
   Docker, les déploiements staging et production n'ont pas encore été exécutés
   par Jenkins, faute de credential de registre.
3. Fournir un endpoint public et le scope GitHub `admin:repo_hook` pour activer
   le webhook; sinon, conserver le scan périodique.
4. Rendre le statut Jenkins réussi obligatoire avant le merge de `main`.
5. Si souhaité, ajouter un credential de notification Discord.
6. Vérifier staging, puis approuver explicitement le déploiement de production
   dans Jenkins.
