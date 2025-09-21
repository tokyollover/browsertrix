# Déploiement de Browsertrix sur DigitalOcean (DOKS) avec Cloudflare

## Outils à installer localement

Avant de démarrer, installez ces outils sur votre machine:

- git
- curl
- jq
- Python 3 + pip
- Ansible (>= 2.16 recommandé)
- doctl (CLI DigitalOcean)
- kubectl
- Helm 3

Option Ubuntu (rapide):

```bash
sudo apt-get update -y
sudo apt-get install -y git curl jq python3 python3-pip
pip3 install --user ansible

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod +x get_helm.sh && ./get_helm.sh

# doctl (exemple de version packagée par DO)
DOCTL_VER=1.105.0
curl -fsSL -o doctl.tgz https://github.com/digitalocean/doctl/releases/download/v${DOCTL_VER}/doctl-${DOCTL_VER}-linux-amd64.tar.gz
sudo tar -xzf doctl.tgz -C /usr/local/bin doctl && rm -f doctl.tgz

# Vérifications
ansible --version | head -n 1
doctl version
kubectl version --client=true --output=yaml | head -n 5
helm version
```

Ce guide explique comment reproduire le déploiement de Browsertrix Cloud (UI + API + crawlers) sur DigitalOcean Kubernetes (DOKS), avec DNS géré par Cloudflare, MongoDB et MinIO en cluster (coûts maîtrisés), et des estimations de coûts mensuels.


## Prés-requis

- Un compte DigitalOcean avec un Token API (lecture/écriture).
- Un domaine géré par Cloudflare (ex. winnersboss.com) et votre Global API Key + e‑mail Cloudflare.
- Une machine Linux avec bash (Codespaces/Ubuntu OK).
- Ce dépôt cloné localement.

Astuce sécurité: exportez les secrets dans votre shell, ne les committez pas.


## Paramétrage des variables d’environnement

Définissez au minimum:

```bash
export DO_API_TOKEN="<votre_do_token>"
export CF_API_EMAIL="<votre_email_cloudflare>"
export CF_API_KEY="<votre_global_api_key_cloudflare>"
# Optionnel si vous la connaissez déjà
# export CF_ZONE_ID="<zone_id_cloudflare>"
# Voir fichier exports_var_browsertrix.txt dans le dossier WB
```

Dans ce guide, nous utilisons Mongo et MinIO “in-cluster” (chart local), pour éviter les coûts et la complexité des services managés. Aucun DO Spaces ni base de données managée n’est requis.


## Préparation de l’inventaire

Dupliquez le fichier d’exemple et adaptez-le:

```bash
cp ansible/inventory/digital_ocean/mirrors.winnersboss.vars.yaml \
   ansible/inventory/digital_ocean/monprojet.vars.yaml
```

Points importants dans ce fichier:
- `project_name`: identifiant court (ex. "monprojet")
- `domain` et `subdomain`: ex. `domain: "votredomaine.com"`, `subdomain: "app"` → FQDN app.votredomaine.com
- `dns_provider: "cloudflare"`
- `mongo_local: true`
- `minio_local: true`
- `enable_signing: true` (optionnel) et `signing_host: "signing"` si vous voulez signer les téléchargements
- `superuser_email` et `superuser_password` (changez le mot de passe!)

Vous pouvez également adapter la région et les tailles des nœuds via les extra-vars au moment du run.


## Installation des collections Ansible (une fois)

```bash
ansible-galaxy collection install -r ansible/requirements.yml --force
```


## Lancement du déploiement

Exécutez le playbook principal avec vos variables:

```bash
DO_API_TOKEN="$DO_API_TOKEN" \
CF_API_EMAIL="$CF_API_EMAIL" \
CF_API_KEY="$CF_API_KEY" \
ansible-playbook -i ansible/inventory/digital_ocean/hosts.ini ansible/do_setup.yml \
  -e @ansible/inventory/digital_ocean/monprojet.vars.yaml \
  -e droplet_region=sfo3 \
  -e main_node_size=s-4vcpu-8gb \
  -e crawl_node_size=c-4
```

Ce que fait le playbook:
- Vérifie/installe helm, kubectl, jq.
- Crée/vérifie un cluster DOKS et sauvegarde le kubeconfig localement.
- Installe ingress-nginx, cert-manager.
- Récupère l’IP du Load Balancer.
- Crée/Met à jour les enregistrements A sur Cloudflare (proxied=false au départ pour ACME HTTP-01).
- Génère `chart/<project_name>-values.yaml` en activant Mongo/MinIO locaux.
- Déploie Browsertrix (backend, frontend, emails, crawlers, signer) via Helm.

À la fin, accédez à: https://<subdomain>.<domain>


## Post-déploiement

- Patientez quelques minutes que Let’s Encrypt émette le certificat. Vous pouvez ensuite activer le proxy Cloudflare (orange) si vous le souhaitez.
- Connectez-vous avec le superuser défini, changez immédiatement le mot de passe.
- Pour vérifier l’état:

```bash
kubectl get pods -A
kubectl get ingress -A
```


## Guide de démarrage rapide

Voici les étapes minimales pour commencer à utiliser Browsertrix après le déploiement:

1) URL d’accès
- Ouvrez: `https://<subdomain>.<domain>`
- Exemple si vous avez suivi l’exemple du guide: `https://mirrors.winnersboss.com`

2) Identifiants par défaut
- Ces identifiants proviennent de vos variables Ansible:
  - Si vous avez utilisé le modèle `mirrors.winnersboss.vars.yaml` sans changement:
    - Email: `admin@winnersboss.com`
    - Mot de passe: `CHANGE_ME_STRONG`
    - Voir Fichier Local pour pwd actuel
  - Si vous n’avez rien surchargé et gardez les valeurs globales par défaut (`ansible/inventory/digital_ocean/group_vars/main.yml`):
    - Email: `dev@webrecorder.net`
    - Mot de passe: `PassW0rd!`

Important: changez immédiatement ce mot de passe après la première connexion (Paramètres du compte) et utilisez un coffre de secrets.

3) Créer votre première collecte
- Créez (ou utilisez) votre Organisation par défaut.
- Allez dans “Crawl Configs” et créez une configuration:
  - Seeds: ajoutez une ou plusieurs URLs (ex: `https://example.org/`).
  - Mode: On-demand (lancement manuel) ou Scheduler si vous souhaitez planifier.
  - Limites: durée max (ex. 5–15 min), profondeur, nombre de pages, etc.
  - Navigateur(s): laissez par défaut pour débuter.
- Enregistrez la configuration.

4) Lancer un crawl
- Depuis la config, cliquez “Start Crawl”.
- Suivez l’exécution en temps réel: onglet “Logs” et “Pages”.

5) Consulter les résultats
- Une fois terminé, ouvrez la Run pour voir la liste des pages collectées.
- Vous pouvez prévisualiser les captures et/ou exporter un paquet WACZ.

6) Option: Sous-domaine de signature
- Si `enable_signing=true`, les téléchargements peuvent être servis via un sous-domaine de signature (ex: `signing.<subdomain>.<domain>`).
- Ce mécanisme est géré par le chart (auth token injecté côté backend/ingress). Aucun réglage supplémentaire n’est requis pour un usage standard via l’UI.


## Mise à l’échelle (basique)

- Augmenter/réduire les nœuds: ajustez `node_pools` dans `ansible/inventory/digital_ocean/group_vars/main.yml` ou passez d’autres tailles en `-e main_node_size=... -e crawl_node_size=...` puis relancez le playbook.
- Espaces de stockage des crawlers: la valeur `crawler_storage: "220Gi"` dans `do-values.template.yaml` (déduite dans `chart/<project>-values.yaml`) pilote la taille des volumes persistants (DO Block Storage via CSI). Ajustez puis redeployez (attention aux migrations de PV).


## Teardown (nettoyage)

- Supprimez les enregistrements DNS Cloudflare (A records) si vous avez activé le proxy.
- Détruisez le cluster et ressources:

```bash
ansible-playbook ansible/playbooks/do_teardown.yml \
  -e k8s_name="<project_name>" \
  -e droplet_region=sfo3
```

Remarque: le teardown fourni est orienté DigitalOcean (doctl). Si vous avez gardé DNS chez Cloudflare, supprimez les A records manuellement ou ajoutez une tâche Cloudflare dédiée.


## Estimation des coûts mensuels (indicatif)

Les coûts DigitalOcean varient par région et évoluent dans le temps. Vérifiez toujours la page de tarification DigitalOcean avant décision. Ci-dessous, un ordre de grandeur pour la configuration par défaut de ce guide (2 nœuds + 1 Load Balancer + volumes persistants):

- Nœud principal (main): taille `s-4vcpu-8gb` → typiquement dans la fourchette d’un droplet Standard 4 vCPU / 8 GB RAM (≈ quelques dizaines d’€/mois).
- Nœud de crawl (crawling): taille `c-4` (Compute-Optimized 4 vCPU) → généralement un peu plus cher que le standard (≈ quelques dizaines d’€/mois, souvent au‑dessus du standard).
- Load Balancer géré (ingress): coût mensuel fixe (≈ faible dizaine d’€/mois).
- Volumes (Block Storage via CSI): facturation au Go/mois. Avec `crawler_storage: 220Gi` + Mongo/MinIO locaux, comptez quelques centaines de Go si vous gardez des crawls, sinon beaucoup moins pour un PoC. Multipliez la capacité retenue par le prix/Go.

Exemple d’ordre de grandeur pour un environnement de test léger:
- 1 nœud standard 4 vCPU/8GB + 1 nœud compute 4 vCPU → combinaison autour de la centaine d’€/mois (±) selon le type exact.
- 1 Load Balancer → ≈ faible dizaine d’€/mois.
- 250 GiB de volumes → prix/Go × 250 (reportez-vous au tarif DO Block Storage).

En additionnant, un budget indicatif pour 2 nœuds + LB + ~250 GiB peut se situer grossièrement autour de la fourchette basse à moyenne des centaines d’€/mois, selon:
- Les tailles exactes choisies (Standard vs Compute)
- La région (SFO3, FRA1, etc.)
- La capacité de stockage effectivement provisionnée et conservée

Optimisations possibles:
- Pour un PoC court: utilisez un seul nœud Standard et réduisez `crawler_storage` (ex. 50–100 GiB)
- Éteignez/détruisez le cluster dès la fin des tests pour éviter des coûts inutiles
- Si vous avez déjà un stockage objet externe, vous pouvez basculer MinIO vers un service existant (mais cela ajoute des coûts côté provider).

Important: les services managés évités ici (DO MongoDB, DO Spaces) auraient ajouté des coûts récurrents supplémentaires. Ce guide privilégie le déploiement “in-cluster” pour contenir les coûts.


## Dépannage (FAQ courte)

- « Failed to login using API token »: vérifiez que `DO_API_TOKEN` est bien présent dans la session où vous lancez Ansible.
- ACME/Let’s Encrypt ne passe pas: vérifiez que les enregistrements A Cloudflare ne sont pas “proxied” au moment de l’émission du certificat (mettez proxied=false), et que l’ingress répond bien en HTTP/80.
- Le Load Balancer n’a pas d’IP: attendez quelques minutes et relancez la tâche; vérifiez `kubectl get svc -n ingress-nginx`.
- Les pods restent en `ContainerCreating`: regardez les events `kubectl describe pod <pod>` et les volumes CSI (droits, quotas, capacité…).


## Références

- Répertoire Ansible: `ansible/`
- Playbook principal: `ansible/do_setup.yml`
- Valeurs DO → chart: `ansible/roles/digital_ocean/setup/templates/do-values.template.yaml`
- Fichier d’override généré: `chart/<project_name>-values.yaml`
- Chart Helm: `chart/`

---

Si vous souhaitez une version “proxied” Cloudflare (CDN/WAF) et/ou un teardown complet incluant les DNS Cloudflare, je peux fournir les tâches Ansible complémentaires.
