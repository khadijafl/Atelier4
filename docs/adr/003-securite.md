# ADR 003 : sécurité de l'API LogiAI

## Décision 1 : où transmettre la clé (en-tête, URL ou corps) ?
- Votre choix :les deux. Le Dockerfile copie `models/` dans l'image (`COPY models/`), et le Compose monte `./models:/app/models`, ce qui masque la copie de l'image à l'exécution. Le modèle est donc lu depuis mon disque avec Compose, et reste disponible dans l'image si on la lance seule.
- Ce que l'autre option ferait perdre :
  - Avec un modèle **uniquement copié** : à chaque ré-entraînement (`python -m logiai.train`), il faudrait reconstruire l'image (`docker compose up -d --build`) pour que le conteneur voie le nouveau `model.joblib`.
  - Avec un modèle **uniquement monté** : l'image ne contiendrait plus de modèle. Lancée seule, sans le dossier `models/`, elle démarrerait sans modèle et l'API ne pourrait pas prédire. L'image ne serait plus transportable à elle seule.

## Décision 2 : quelles origines autoriser (CORS) ? Une liste précise ou « * » ?
- Votre choix :les deux. 8080 pour la page et l'API via Nginx, 8001 pour accéder directement à l'API (Swagger sur `/docs`, `/health`, tests avec `curl`).
- Ce que l'autre option ferait perdre :
  - En publiant **8080 seulement** : on perdrait l'accès direct à `/docs` et les tests `curl` sur l'API seule, et on réduirait la surface exposée à un seul point d'entrée.
  - En publiant **aussi 8001** : un visiteur peut contourner Nginx et appeler l'API directement. Il reste bloqué par la clé API (401 sans clé), car Nginx n'est pas une protection : il relaie. Mais il peut atteindre `/docs` et `/health` sans clé, et tout filtre qu'on ajouterait plus tard dans Nginx serait contourné.
- Pour la production : ne publier que le port 8080.
