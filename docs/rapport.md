# Rapport de laboratoire

## 1. Contexte et périmètre

Le scénario décrit une sauvegarde complète du stockage de Vault copiée sur un serveur externe non sécurisé. Les données sont chiffrées, mais la perte de contrôle de cette copie impose d’évaluer le risque pour les secrets et les clés.

La réponse technique a été reproduite sur une VM Kali : configuration de Vault, activation de l’audit, analyse d’un indicateur simulé, rotation de la clé interne, changement d’un secret et contrôles de fonctionnement. L’isolement du serveur ou de la sauvegarde exposée relève d’une autre équipe dans l’énoncé ; il n’a pas été exécuté dans ce laboratoire.

## 2. Préparation

Vault 2.2.1 utilise un stockage fichier dans `/opt/vault/data`. L’API écoute sur `127.0.0.1:8200`. Après initialisation et déverrouillage, le statut indique `Initialized: true` et `Sealed: false`.

Le périphérique d’audit fichier a été activé vers `/vault/logs/audit.log`, puis le moteur KV v2 sous `secret/`. Le secret `secret/prod/db` a été créé avec une valeur de test. La lecture a confirmé la présence du secret en version 1.

Preuves : [statut initialisé](../screenshots/02-vault-initialise-deverrouille.png), [audit](../screenshots/03-audit-active.png), [secret initial](../screenshots/04-secret-initial.png).

## 3. Détection et analyse

L’entrée JSON simulée a été ajoutée au journal, puis retrouvée avec `grep` en recherchant `key-status`.

```json
{"time":"2025-10-24T22:45:00Z","type":"request","auth":{"client_token":"s.fake"},"request":{"path":"sys/key-status","operation":"read","remote_address":"203.0.113.200"}}
```

L’entrée indique une requête de lecture sur `sys/key-status`, avec une adresse source fictive `203.0.113.200` et le token fictif `s.fake`. Sa date est celle du scénario, antérieure aux manipulations réalisées le 2 novembre 2026.

Les autres résultats de la recherche comprennent la consultation locale, depuis `127.0.0.1`, ainsi que sa réponse. Un même appel peut donc produire plusieurs lignes ; leur nombre ne correspond pas directement au nombre d’accès suspects.

Cette entrée constitue l’indicateur de l’exercice. Elle ne prouve ni la copie du stockage ni une extraction de clé. Une requête de lecture seule, sans réponse associée, ne démontre pas non plus qu’elle a réussi. Dans un incident réel, il faudrait corréler les journaux de Vault avec ceux du système, du réseau et de l’outil de sauvegarde. Une copie directe des fichiers peut ne pas apparaître dans l’audit de l’API Vault.

Preuve : [capture recadrée sur l’entrée simulée](../screenshots/06-evenement-suspect.png). Aucun journal complet n’est publié.

## 4. Évaluation et mesures de réponse

Le scénario établit l’exposition de la sauvegarde, sans démontrer la possession de clés par un tiers. Le risque porte sur la confidentialité des secrets si un attaquant parvient à obtenir le matériel de déchiffrement. La décision de rotation constitue ici une mesure de précaution, accompagnée du renouvellement de l’identifiant stocké.

L’isolement de la sauvegarde exposée doit limiter les accès supplémentaires tout en préservant les éléments utiles à l’enquête. Cette action est hors du périmètre exécuté. Les rotations seules ne corrigent pas la cause de la copie non autorisée.

## 5. Rotation de la clé interne

Avant la rotation, `vault operator key-status` indiquait `Key Term: 1`. La commande `vault operator rotate` a renvoyé `Success! Rotated key` et `Key Term: 2`. Une nouvelle lecture de l’état a confirmé la génération 2, installée le 2 novembre 2026 à 17 h 12 UTC.

La lecture de `secret/prod/db` après rotation a confirmé que la version 1 était toujours accessible. Ce contrôle vérifie que le changement de clé n’a pas empêché la lecture de cette donnée existante.

La nouvelle clé sert aux écritures futures ; les anciennes restent dans le trousseau pour déchiffrer les données antérieures. La rotation ne révoque donc pas rétroactivement la capacité de déchiffrer une sauvegarde déjà volée si l’attaquant en possède aussi les clés nécessaires.

Preuve : [rotation et lecture du secret](../screenshots/07-rotation-cle.png).

## 6. Modification du mot de passe stocké

Le champ `password` a été mis à jour avec une seconde valeur de test. La lecture finale montre la version 2, créée le 2 novembre 2026 à 17 h 30 min 52 s UTC.

Ce contrôle confirme le changement dans Vault. Il ne démontre pas une rotation d’identifiant dans une base réelle, car aucune base n’a été configurée. KV v2 conserve la version 1 ; le changement de valeur ne signifie pas sa destruction.

Preuves : [commande d’écriture et version 2 créée](../screenshots/08a-ecriture-mot-de-passe.png), puis [lecture de la version 2](../screenshots/08-rotation-mot-de-passe.png). Les deux sorties présentent la même date de création et confirment la modification attendue.

## 7. Vérification finale

Les commandes `vault status` et `vault operator key-status` confirment un coffre-fort initialisé, déverrouillé et utilisant la génération 2. Les lectures avant et après les changements ont réussi. La procédure n’a nécessité aucun redémarrage. Ces contrôles ponctuels ne mesurent pas la disponibilité en continu ni le fonctionnement d’une application cliente.

Preuve : [état final](../screenshots/09-verification-finale.png).

## 8. Correspondance avec les phases NIST

La révision 2 de NIST SP 800-61 regroupe le confinement, l’éradication et le rétablissement dans une même phase. Le tableau associe chaque action du scénario à sa phase.

| Étape du TP | Phase NIST SP 800-61 rév. 2 | Justification |
|---|---|---|
| Configuration initiale | Préparation | Préparer le service, les secrets de test et l’audit. |
| Analyse des journaux | Détection et analyse | Rechercher et interpréter l’indicateur simulé. |
| Évaluation du risque et décision de rotation | Détection et analyse | Évaluer le risque de divulgation et choisir les mesures de réponse. |
| Isolement logique | Confinement, éradication et rétablissement | Limiter l’exposition ; action confiée à une autre équipe. |
| Rotation de la clé interne | Confinement, éradication et rétablissement | Renouveler la clé utilisée pour les nouvelles écritures. |
| Rotation du mot de passe DB | Confinement, éradication et rétablissement | Remplacer le secret stocké ; rotation réelle de la base hors périmètre. |
| Vérification et test post-rotation | Confinement, éradication et rétablissement | Vérifier l’accès aux secrets et l’état du service. |
| Documentation et recommandations | Activités post-incident | Formaliser les résultats et les améliorations. |

La documentation se fait également pendant l’incident ; le rapport final et les leçons apprises relèvent ici des activités post-incident.

## 9. Leçons apprises et recommandations

- Une sauvegarde chiffrée exposée reste un incident : il faut déterminer qui a pu y accéder et si les clés ou identifiants ont aussi été exposés.
- La rotation interne protège les nouvelles écritures ; elle ne remplace pas le confinement, l’investigation et la correction des permissions de sauvegarde.
- Il faut coordonner les changements entre Vault, la base et les applications, puis tester que l’ancien identifiant est effectivement refusé.
- L’historique KV v2 doit avoir une politique de conservation adaptée. Toute destruction d’ancienne version doit tenir compte des besoins de preuve et de restauration.
- Les sauvegardes doivent être accessibles aux seuls comptes nécessaires, chiffrées et surveillées. Le matériel de déverrouillage doit être protégé séparément.
- En production, utiliser TLS, un compte de service dédié, des politiques de moindre privilège et une gestion des clés adaptée. Le token root ne doit pas servir aux opérations courantes.
- Centraliser les journaux et protéger leur intégrité permet de réduire le risque de modification locale. L’ajout manuel effectué ici est uniquement une simulation pédagogique.

## Références

- Énoncé du TP1, cours 8INF882, fourni par l’étudiant.
- [NIST SP 800-61 révision 2, sections 3.1 à 3.4](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf). Cadre demandé par le TP, remplacé par la révision 3 en avril 2025.
- [Documentation de la rotation Vault](https://developer.hashicorp.com/vault/docs/commands/operator/rotate).
- [Documentation KV v2](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2).
