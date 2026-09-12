# Juice Shop Security Pipeline

Projet d’évaluation et d’automatisation de la sécurité de l’application web **OWASP Juice Shop** dans le cadre de l’examen pratique de sécurité des données.

## 1. Objectifs

Ce projet permet de :

- identifier des vulnérabilités de sécurité sur Juice Shop ;
- automatiser plusieurs contrôles de sécurité ;
- intégrer ces contrôles dans une pipeline Jenkins ;
- générer et archiver automatiquement les rapports ;
- notifier le résultat de chaque exécution par e-mail.

## 2. Outils utilisés

- **Jenkins** : automatisation de la pipeline CI/CD ;
- **GitHub** : hébergement du dépôt et déclenchement de la pipeline ;
- **npm audit** : analyse SCA des dépendances ;
- **OWASP ZAP** : analyse DAST de l’application ;
- **Docker** : exécution de Juice Shop et de ZAP ;
- **ngrok** : exposition temporaire de Jenkins pour recevoir le webhook GitHub.

## 3. Structure du projet

```text
project/
├── README.md
├── Jenkinsfile
├── reports/
│   ├── dast.html
│   ├── dast.json
│   ├── sca.json
│   ├── security-summary.txt
│   └── zap.yaml
├── screenshots/
├── security-config/
└── remediation/
```

Le répertoire contient également le code source de l'application Juice Shop utilisée pour les analyses.

## 4. Lancer l'application

Juice Shop peut être lancé avec Docker :

```bash
docker run -d \
  --name juice-shop \
  -p 3000:3000 \
  bkimminich/juice-shop
```

L'application est ensuite accessible à :

```text
http://127.0.0.1:3000
```

## 5. Lancer les analyses manuellement

### SCA avec npm audit

Depuis le répertoire du projet :

```bash
npm audit --json > reports/sca.json
```

Cette analyse recherche les vulnérabilités connues dans les dépendances du projet.

### DAST avec OWASP ZAP

Exemple d'exécution :

```bash
docker run --rm \
  --network host \
  -v "$PWD/reports:/zap/wrk/:rw" \
  ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py \
  --autooff \
  -m 1 \
  -T 5 \
  -z "-silent" \
  -t http://127.0.0.1:3000 \
  -J dast.json \
  -r dast.html
```

Les rapports générés sont enregistrés dans `reports/`.

## 6. Exécution de la pipeline Jenkins

La pipeline est définie dans `Jenkinsfile`.

Elle comprend les étapes principales suivantes :

1. Checkout du dépôt ;
2. Build / préparation ;
3. Analyse de sécurité SCA ;
4. Analyse DAST avec OWASP ZAP ;
5. Génération des rapports ;
6. Notification par e-mail.

La pipeline est déclenchée automatiquement lors d'un push sur le dépôt GitHub via un webhook.

## 7. Résultats du Build de référence

Le **Build #8** constitue l'exécution de référence du projet.

### Analyse SCA

`npm audit` a détecté :

* **7 vulnérabilités critiques**
* **19 vulnérabilités élevées**
* **16 vulnérabilités modérées**
* **3 vulnérabilités faibles**

Soit un total de **45 vulnérabilités liées aux dépendances**.

### Analyse DAST

OWASP ZAP a analysé **88 URLs** :

* **0 FAIL-NEW**
* **3 WARN-NEW**
* **57 PASS**

Les rapports détaillés sont disponibles dans `reports/`.

## 8. Rapports et preuves

Les résultats de la pipeline sont conservés dans le répertoire `reports/`.

Les captures d'écran utilisées comme preuves de l'exécution, des résultats et des notifications sont disponibles dans `screenshots/`.

Les éléments de configuration de sécurité sont regroupés dans `security-config/`.

Les propositions de remédiation sont regroupées dans `remediation/`.

## 9. Limites

Les résultats de `npm audit` concernent principalement les dépendances utilisées par l'application et ne constituent pas à eux seuls une analyse complète de la sécurité applicative.

OWASP ZAP réalise une analyse automatisée dynamique et peut produire des alertes nécessitant une validation manuelle.

Les vulnérabilités applicatives identifiées manuellement sont donc à considérer conjointement avec les résultats automatisés.

## 10. Auteur

**Serigne Saliou Mbacké Seck**
Licence 3 SIMAC
Université Numérique Cheikh Hamidou Kane

