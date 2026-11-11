# Procédure du laboratoire

Ces commandes décrivent la réalisation sur une **nouvelle instance**. Ne pas réinitialiser une instance déjà initialisée et ne pas rejouer les écritures sur l’instance terminée : chaque `kv put` crée une nouvelle version et chaque rotation augmente le `Key Term`.

## 1. Installation

```bash
uname -m
df -h /
command -v vault
sudo apt update
sudo apt install -y curl unzip
mkdir -p ~/tp1-vault/installation
cd ~/tp1-vault/installation
curl -fLO https://releases.hashicorp.com/vault/2.2.1/vault_2.2.1_linux_amd64.zip
curl -fLO https://releases.hashicorp.com/vault/2.2.1/vault_2.2.1_SHA256SUMS
grep ' vault_2.2.1_linux_amd64.zip$' vault_2.2.1_SHA256SUMS | sha256sum -c -
```

Continuer uniquement si la vérification affiche `OK`. Cette comparaison contrôle l’intégrité par rapport au fichier de sommes téléchargé ; la signature de ce fichier n’a pas été vérifiée dans cette procédure.

```bash
unzip vault_2.2.1_linux_amd64.zip
sudo install -m 0755 vault /usr/local/bin/vault
vault version
```

## 2. Configuration et démarrage

Placer le dossier du projet dans `~/vault-incident-response`. La configuration est fournie dans `config/vault.hcl`.

```bash
cd ~/vault-incident-response
sudo install -d -m 0700 -o "$(id -un)" -g "$(id -gn)" /opt/vault/data /vault/logs
vault server -config="$HOME/vault-incident-response/config/vault.hcl"
```

Laisser ce terminal ouvert. Dans un deuxième terminal :

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
vault status
```

Sur une instance neuve : `Initialized: false`, `Sealed: true`. Le code de retour 2 de `vault status` indique un coffre-fort scellé.

## 3. Initialisation, déverrouillage et connexion

Créer les informations d’accès hors du dépôt, dans un fichier lisible uniquement par l’utilisateur. Exécuter l’initialisation une seule fois.

```bash
mkdir -p -m 700 ~/.local/share/vault-tp1
(
  umask 077
  set -o noclobber
  vault operator init -key-shares=1 -key-threshold=1 \
    > ~/.local/share/vault-tp1/initialisation.txt
)
cat ~/.local/share/vault-tp1/initialisation.txt
```

Le choix d’une part et d’un seuil de 1 est une simplification pour ce laboratoire individuel. Ne pas publier la sortie du fichier.

```bash
vault operator unseal
```

À l’invite masquée, saisir `Unseal Key 1`.

```bash
vault login -no-print
```

À l’invite, saisir `Initial Root Token`. La CLI mémorise par défaut le token dans `~/.vault-token`, hors du projet.

```bash
vault status
```

Attendu : `Initialized: true`, `Sealed: false`.

## 4. Audit et secret initial

```bash
vault audit enable file file_path=/vault/logs/audit.log
vault audit list
vault secrets enable -path=secret kv-v2
vault kv put secret/prod/db password="mot-de-passe-labo-v1"
vault kv get secret/prod/db
vault operator key-status
```

Le mot de passe est une valeur d’exemple pour reproduire le laboratoire. Attendu sur l’instance neuve : secret en version 1 et `Key Term: 1`.

## 5. Simulation et analyse

Ajouter l’événement une seule fois. L’ajout manuel dans un journal actif est imposé par cet exercice ; il ne représente pas une collecte de preuve réelle.

```bash
cat >> /vault/logs/audit.log <<'EOF'
{"time":"2025-10-24T22:45:00Z","type":"request","auth":{"client_token":"s.fake"},"request":{"path":"sys/key-status","operation":"read","remote_address":"203.0.113.200"}}
EOF
grep 'key-status' /vault/logs/audit.log
```

La date et l’adresse sont celles de l’énoncé. Comparer la ligne fictive aux requêtes locales. Le rapport distingue cet indicateur de la preuve d’une exfiltration.

## 6. Rotation de la clé interne

L’isolement de la sauvegarde exposée relève d’une autre équipe dans le scénario et n’a pas été réalisé sur cette VM.

```bash
vault operator key-status
vault operator rotate
vault operator key-status
vault kv get secret/prod/db
```

Attendu : `Key Term` de 1 à 2 ; secret initial toujours lisible en version 1. Ne pas confondre cette opération avec `operator rekey`, qui concerne le matériel de déverrouillage.

## 7. Modification du mot de passe stocké

```bash
vault kv put secret/prod/db password="mot-de-passe-labo-v2"
vault kv get secret/prod/db
vault status
vault operator key-status
```

Attendu : nouvelle valeur en version 2, coffre-fort déverrouillé et `Key Term: 2`. Aucune base réelle n’est modifiée par ces commandes.

## 8. Reprise après arrêt du serveur

Les données persistent. Relancer `vault server` avec la même configuration, puis rétablir `VAULT_ADDR` dans le terminal client et exécuter `vault operator unseal` avec la clé conservée. Ne pas relancer `operator init` et ne pas supprimer `/opt/vault/data`.

## 9. Pièces à conserver

- Statut initialisé et déverrouillé.
- Audit activé et secret initial.
- Indicateur simulé retrouvé dans le journal.
- État de la clé avant et après `operator rotate`.
- Lecture du secret préexistant après rotation.
- **Commande et sortie de `kv put` pour le nouveau mot de passe**, puis lecture de la version 2.
- État final.

Le dossier fourni comprend les captures reçues dans la conversation, notamment l’écriture du nouveau mot de passe et sa lecture en version 2. Il n’est pas nécessaire de rejouer les opérations pour compléter les preuves.
