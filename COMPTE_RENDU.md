# Compte rendu : tester, sécuriser et appeler l'API LogiAI

**Nom :** Flija **Groupe :** 4

## 1. Casser son API (activité 1)
| Cas | Requête (corps ou URL) | Prévu | Obtenu |
|---|---|---|---|
| champ manquant | livraison valide sans poids_kg | 422 | 422 |
| valeur absurde | pluie_mm: 500 (ou poids_kg: 99999 dans le test) | 422 | 422 |
| texte au lieu d'un nombre | distance_km: "beaucoup" | 422 | 422 |
| champ en trop | ajout de "couleur": "rouge" | 422 | 422 |
| mauvaise route | POST /v1/predictt | 404 | 404 |

- Une requête acceptée alors qu'elle devait être refusée ? Laquelle ?
  Non, les 5 requêtes ont été refusées comme prévu. Les requêtes acceptées à tort se trouvaient aux bornes (activité 2).

## 2. Les 2 bugs (activité 2)
| Bug | Test qui l'a révélé | Comportement avant | Correction |
|---|---|---|---|
| 1 | heure_depart=24 | attendu 422, obtenu 200 : l'heure 24 n'existe pas (0 à 23) mais l'API l'acceptait | `le=24` remplacé par `le=23` |
| 2 | nb_arrets=-1 | attendu 422, obtenu 200 : un nombre d'arrêts négatif était accepté | ajout de `ge=0` |

- Sortie de `python -m pytest` à la fin de l'activité 2 :
  41 passed, 1 warning

## 3. La clé API (sprint 1)
- Ma fonction `verify_api_key()` :

```python
def verify_api_key(x_api_key: str | None = Header(default=None)) -> str:
    if x_api_key is None:
        raise HTTPException(
            status_code=401,
            detail="Clé API manquante",
            headers={"WWW-Authenticate": "ApiKey"},
        )
    if _egal(x_api_key, config.ADMIN_KEY):
        return "admin"
    if _egal(x_api_key, config.API_KEY):
        return "user"
    raise HTTPException(
        status_code=401,
        detail="Clé API invalide",
        headers={"WWW-Authenticate": "ApiKey"},
    )
```

| Requête | Code obtenu |
|---|---|
| `/v1/predict` sans clé | 401 |
| `/v1/predict` fausse clé | 401 |
| `/v1/predict` clé utilisateur | 200 |
| `/v1/model-info` clé utilisateur | 403 |
| `/v1/model-info` clé administrateur | 200 |
| `/health` sans clé | 200 |

- Preuve que `.env` n'est pas dans Git (sortie de `git status`) :
  À COMPLÉTER : colle ici la sortie de `git status` (le fichier `.env` n'y apparaît pas).

## 4. Le client web (sprint 2)
- Mon `fetch` :

```javascript
async function appelerApi(livraison, cle) {
  const reponse = await fetch(API_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json", "X-API-Key": cle },
    body: JSON.stringify(livraison),
  });
  const corps = await reponse.json();
  return { status: reponse.status, corps };
}
```

- Ce qui s'affiche :
  - livraison normale : « Probabilité de retard : 73.2 %. Alerte : livraison à risque. »
  - pluie de 500 mm : « pluie_mm : Input should be less than or equal to 100 »
  - fausse clé : À COMPLÉTER (recopie le texte affiché par la page)
- Avant l'ajout du CORS, la page affichait : « API injoignable ou bloquée (serveur arrêté ? CORS ?) : Failed to fetch ».
- Expérience CORS (page servie sur le port 5501) : la page affiche « API injoignable ou bloquée (serveur arrêté ? CORS ?) : Failed to fetch ». Le navigateur bloque la réponse parce que http://localhost:5501 n'est pas dans `config.ALLOWED_ORIGINS`. `curl` marche quand même parce que CORS est une règle du navigateur : `curl` n'est pas un navigateur, seule la clé compte.
- Où voyez-vous la clé dans F12 ? À COMPLÉTER (Network, requête `predict`, en-têtes de la requête : `X-API-Key`). Si la clé était écrite dans le JavaScript de la page, n'importe quel visiteur pourrait la lire dans le code source.

## 5. Questions de compréhension
1. Quelle différence entre 401, 403 et 422 ?
   401 : l'API ne sait pas qui appelle (clé absente ou fausse). 403 : elle sait qui appelle, mais l'action n'est pas permise (clé utilisateur sur /v1/model-info). 422 : l'appelant est reconnu, mais les données envoyées sont invalides. La clé est vérifiée avant les données, donc une pluie de 500 mm sans clé donne 401, pas 422.
2. Pourquoi la clé n'est-elle ni dans le code ni dans Git ?
   Une clé commitée reste dans l'historique de Git pour toujours, même si on supprime le fichier ensuite, et elle est lisible par tous ceux qui ont accès au dépôt. Elle est dans `.env`, ignoré par Git ; `.env.example` ne contient que de fausses valeurs.
3. Pourquoi comparer avec `compare_digest` plutôt qu'avec `==` ?
   `==` s'arrête au premier caractère différent, donc la durée de la réponse révèle combien de caractères sont justes (attaque par temps de réponse). `compare_digest` prend un temps constant.
4. Pourquoi `curl` fonctionne-t-il alors que la page du port 5501 est bloquée ?
   Le blocage vient du navigateur, qui applique CORS pour protéger l'utilisateur. `curl` n'est pas un navigateur et n'applique pas CORS : seule la clé API compte.
5. Pourquoi la validation faite par le navigateur ne remplace-t-elle pas celle de l'API ?
   N'importe quel client (curl, script, autre programme) peut envoyer une requête sans passer par la page. L'API doit donc valider elle-même toutes les données reçues.

## 6. Lien de la pull request
À COMPLÉTER (lien de la pull request GitHub, une fois créée)