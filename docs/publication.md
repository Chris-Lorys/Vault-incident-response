# Publier le projet sur GitHub

1. Décompresser l’archive dans un dossier de travail.
2. Sur GitHub, créer un dépôt vide nommé `vault-incident-response` et choisir sa visibilité. Laisser les options de génération du README et du `.gitignore` désactivées.
3. Ouvrir un terminal dans le dossier décompressé qui contient `README.md`, puis exécuter les commandes suivantes en remplaçant `VOTRE_COMPTE` par le nom du compte GitHub.

```bash
git init -b main
git add README.md .gitignore config docs screenshots
git diff --cached --stat
git commit -m "Documenter la réponse à incident sur Vault"
git remote add origin https://github.com/VOTRE_COMPTE/vault-incident-response.git
git push -u origin main
```

GitHub affiche le README à la racine du dépôt. Vérifier que les captures s’affichent et que les liens vers la procédure et le rapport s’ouvrent.

**Description du dépôt** — Analyse des journaux d’audit Vault, rotation de la clé de chiffrement et mise à jour d’un secret KV v2 dans un scénario de réponse à incident.
