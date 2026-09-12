# Remédiations de sécurité

Ce dossier présente des exemples de remédiation des vulnérabilités identifiées sur OWASP Juice Shop.

Les correctifs présentés ci-dessous sont basés sur les mécanismes de correction proposés directement par les challenges de sécurité de Juice Shop.

---

## 1. Injection SQL (SQL Injection)

### Vulnérabilité concernée

La vulnérabilité concerne la fonction d'authentification de l'application.

Le code vulnérable construit directement une requête SQL à partir des paramètres fournis par l'utilisateur :

```javascript
return models.sequelize.query(
  `SELECT * FROM Users WHERE email = '${req.body.email}' AND password = '${security.hash(req.body.password)}'`
)
```

Les valeurs `email` et `password` sont directement intégrées dans la requête SQL.

Cette construction permet à une entrée spécialement construite par un attaquant de modifier la logique de la requête SQL.

### Impact

Une injection SQL peut notamment permettre :

* de contourner l'authentification ;
* de modifier la logique de recherche dans la base de données ;
* d'accéder à des données qui ne devraient pas être accessibles.

### Correctif retenu

Dans le **Coding Challenge: Login Admin**, Juice Shop indique que le **Fix 3** permet de prévenir la vulnérabilité.

Le correctif remplace la concaténation directe des données utilisateur par une requête paramétrée avec le mécanisme `bind` de Sequelize :

```javascript
models.sequelize.query(
  'SELECT * FROM Users WHERE email = $1 AND password = $2 AND deletedAt IS NULL',
  {
    bind: [req.body.email, security.hash(req.body.password)],
    model: models.User,
    plain: true
  }
)
```

### Pourquoi ce correctif fonctionne

Les données fournies par l'utilisateur ne sont plus directement incorporées dans la chaîne SQL.

Elles sont transmises séparément grâce au mécanisme de liaison :

```javascript
bind: [req.body.email, security.hash(req.body.password)]
```

La structure de la requête SQL reste ainsi définie indépendamment des valeurs fournies par l'utilisateur.

Le message de validation du challenge précise que le mécanisme de binding de Sequelize est équivalent à l'utilisation d'une requête préparée et empêche la modification de la syntaxe de la requête par une entrée malveillante.

### Validation

Le challenge **Coding Challenge: Login Admin** permet de comparer plusieurs propositions de correction.

Le **Fix 3** a été sélectionné et validé par Juice Shop, comme l'indique l'icône de validation verte affichée dans le challenge.

La capture de cette validation est conservée dans :

```text
screenshots/sqli-remediation-juice-shop.png
```

### Vérification recommandée

Pour une validation complète de la remédiation :

1. reproduire l'injection avant correction ;
2. appliquer le correctif ;
3. reproduire la même entrée après correction ;
4. vérifier que l'entrée ne permet plus de modifier la logique de la requête SQL ;
5. compléter cette vérification par une analyse automatisée lorsque cela est pertinent.

---

## 2. Bonnes pratiques générales contre les injections SQL

Pour prévenir les injections SQL :

* utiliser des requêtes préparées ou paramétrées ;
* éviter la concaténation directe des entrées utilisateur dans les requêtes SQL ;
* utiliser les mécanismes sécurisés fournis par l'ORM ou le framework ;
* valider les données reçues lorsque cela est nécessaire ;
* appliquer le principe du moindre privilège aux comptes utilisés pour accéder à la base de données.

