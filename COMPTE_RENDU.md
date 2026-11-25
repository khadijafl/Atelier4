# Compte rendu : conteneuriser et déployer LogiAI avec Docker

**Nom :** Flija **Groupe :** 4

## 1. Premier constat : ça ne marche que chez moi
| Ce que le projet suppose | 
|---|---|
| Une version de Python et des bibliothèques compatibles avec `model.joblib` 
| Le fichier `.env` avec les deux clés |
| Un modèle entraîné dans `models/` | 
| Deux terminaux : uvicorn, puis `python -m http.server 5500` pour la page |



## 2. Mes fichiers Docker
**Dockerfile (API)**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY logiai/ ./logiai/
COPY data/ ./data/
COPY models/ ./models/
ENV LOGIAI_HOME=/app
EXPOSE 8000
CMD ["uvicorn", "logiai.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**web/Dockerfile (page)**
```dockerfile
FROM nginx:alpine
COPY web/index.html /usr/share/nginx/html/index.html
COPY nginx/default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

**nginx/default.conf**
```nginx
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://api:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

**docker-compose.yml**
```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: logiai-api
    ports:
      - '8001:8000'
    env_file:
      - .env
    environment:
      LOGIAI_HOME: /app
    volumes:
      - ./models:/app/models
      - ./data:/app/data
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

  web:
    build:
      context: .
      dockerfile: web/Dockerfile
    container_name: logiai-web
    ports:
      - '8080:80'
    depends_on:
      api:
        condition: service_healthy
```

## 3. Trois expériences pour casser le conteneur (activité 1)
| Cas | Commande | Prévu | Obtenu | Explication |
|---|---|---|---|---|
| Sans `--env-file` | `docker run -d --name logiai-api-test -p 8001:8000 logiai:1.0` | L'API démarre, mais elle ne connaît aucune clé : 401 même avec la bonne clé. | `/health` : `auth_configured: false` ; `/v1/predict` avec la bonne clé : 401 « Clé API invalide ». | `.env` est exclu de l'image par `.dockerignore` et le conteneur ne voit pas mon dossier Windows : sans `--env-file`, l'API ne connaît aucune clé. |
| Sans publier le port | `docker run -d --name logiai-api-test --env-file .env logiai:1.0` | `curl` échoue : le port n'est pas accessible depuis Windows. | `docker ps` : `8000/tcp` sans `->` ; `curl` : « Failed to connect to localhost:8001 » ; `docker exec` : `/health` répond `ok`. | `EXPOSE` ne fait que documenter le port : sans `-p 8001:8000`, Windows n'a aucun chemin vers le port du conteneur, alors que l'API tourne bien à l'intérieur. |
| Écoute sur 127.0.0.1 | `docker run -d --name logiai-api-test -p 8001:8000 --env-file .env logiai:test` | Ça devrait marcher, puisque le port est publié. | Port publié (`0.0.0.0:8001->8000/tcp`) ; logs : `Uvicorn running on http://127.0.0.1:8000` ; `curl` : « Empty reply from server » ; `docker exec` : `/health` répond `ok`. | Le port est publié, mais uvicorn n'écoute que sur la boucle locale du conteneur : seul l'intérieur peut lui parler. Il faut `--host 0.0.0.0`. |

## 4. Les quatre codes via Nginx (étape 5)
| Requête (port 8080) | Code obtenu |
|---|---|
| `/api/health` | 200 |
| `/api/v1/predict` sans clé | 401 (« Clé API manquante ») |
| `/api/v1/predict` fausse clé | 401 (« Clé API invalide ») |
| `/api/v1/predict` bonne clé `<masquée>` | 200 (probabilité 0.732, alerte true) |

## 5. F12, onglet Network (étape 6)
- Adresse de la requête : `http://localhost:8080/api/v1/predict`, en POST, réponse 200 en 409 ms.
- Requête OPTIONS : non, une seule requête `fetch`.
- Explication : Nginx sert la page et relaie `/api/` vers l'API, donc page et API ont la même origine. Il n'y a pas de CORS ni de requête préliminaire. À l'Atelier 4, la page (port 5500) et l'API (port 8000) avaient deux origines, d'où l'OPTIONS.
- La clé `X-API-Key` reste visible dans les en-têtes : le reverse proxy relaie, il ne protège pas.

## 6. Observer (étape 7)
- Logs de l'API : un `GET /health` depuis `127.0.0.1` toutes les 30 s (le healthcheck). Pour une prédiction passée par Nginx :
  `logiai-api  | INFO:     172.21.0.3:57460 - "POST /v1/predict HTTP/1.1" 200 OK`
  

| Conteneur | CPU | Mémoire | Réseau (reçu / envoyé) | PIDS |
|---|---|---|---|---|
| logiai-api | 0.17 % | 124.2 MiB | 18.7 kB / 8.39 kB | 12 |
| logiai-web | 0.00 % | 4.918 MiB | 33.1 kB / 42.1 kB | 5 |



## 7. Arrêt de l'API (étape 8)
- `docker compose stop api` : seul `logiai-web` reste `Up`.
- `curl /api/health` : **502 Bad Gateway** renvoyé par Nginx.
- La page affiche : **API injoignable ou bloquée (serveur arrêté ? CORS ?) : Unexpected token '<', "<html> <h"... is not valid JSON** 
- Explication : Nginx tourne toujours mais ne peut plus joindre `api:8000`, il répond 502 lui-même. Après `docker compose start api`, l'API passe par `(health: starting)` puis `healthy`, et tout remarche sans reconstruire.

## 8. Le volume (étape 9)
- `docker compose exec api ls /app/models` : `model.joblib model_info.json`, les fichiers de mon disque.
- Preuve : un fichier `preuve.txt` créé dans `models/` côté Windows est apparu dans le conteneur, puis a disparu quand je l'ai supprimé, sans rien reconstruire.
- Nouveau `model.joblib` : le bind mount `./models:/app/models` relie mon dossier Windows au conteneur à l'exécution, donc le conteneur lit directement mon disque et voit le nouveau fichier sans reconstruire l'image.
- Changement dans `logiai/` : je ne monte pas `./logiai`, le code a été copié dans l'image par `COPY` au build, donc il est figé. Il faut `docker compose up -d --build` pour qu'il soit pris en compte.

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