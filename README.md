# veille-ia

Digest IA bi-hebdomadaire — veille interne, publiée sur GitHub Pages.

## Confidentialité — à lire avant toute modification

Ce dépôt est **public** (GitHub Pages n'est pas disponible sur dépôt privé en offre gratuite, et un site Pages reste public même avec GitHub Pro — la visibilité privée est réservée à Enterprise). Tout ce qui est commité ici est lisible par n'importe qui, y compris les métadonnées.

Quatre règles en découlent :

1. **Rien d'identifiant dans le contenu publié.** Le prompt décrit la cible du digest en termes génériques (taille et secteur) et interdit explicitement au modèle de nommer l'entreprise, ses outils ou ses projets. Le nœud `Preparer Publication` applique en plus un filet de sécurité qui neutralise toute occurrence résiduelle avant publication.
   *Motif :* le modèle n'a aucune donnée interne — ce qu'il écrit sur l'entreprise est spéculatif, mais se lit de l'extérieur comme une roadmap, et engage la marque sans mention de non-officialité.

2. **Pas d'identifiants d'infrastructure dans le dépôt.** Pas d'ID de workflow, d'URL d'instance, de nom de credential, d'adresse e-mail ni d'ID de salon de messagerie — ni dans le code, ni dans la documentation. Ces valeurs vivent dans l'outil d'orchestration, pas ici.

3. **Pas d'indexation.** Chaque page publiée porte `<meta name="robots" content="noindex,nofollow,noarchive,nosnippet">`, injecté à la publication. Le `robots.txt` présent à la racine du dépôt est **inopérant** sur un *project page* (les crawlers ne lisent `robots.txt` qu'à la racine du domaine) ; il n'est là que pour le jour où le site passerait sur un domaine dédié. Le meta est le seul blocage réel.

4. **Mention de non-officialité** dans le pied de page de chaque digest et sur la page d'accueil.

Les digests antérieurs à l'application de ces règles ont été retirés du dépôt et de l'historique git (ils restaient sinon lisibles via `git log`). Une copie complète est conservée hors dépôt.

> Point de vigilance : l'auteur des commits (nom et e-mail) est public sur chaque commit, y compris ceux poussés par l'automatisation. Configurer `user.name` / `user.email` du dépôt avec des valeurs neutres, et vérifier ce que porte le compte utilisé par le jeton d'écriture.

## Fonctionnement

Un workflow d'automatisation tourne chaque lundi et jeudi à 9h :

1. Récupère les flux RSS de plusieurs sources IA (Anthropic, OpenAI, Google AI, Hugging Face, The Decoder, TechCrunch, The Verge, Siècle Digital).
2. Filtre les articles des dernières 72h et déduplique.
3. Sélectionne et rédige les 5 actualités les plus pertinentes via un LLM.
4. Envoie le digest par e-mail à l'équipe — version interne, qui peut être nominative.
5. Publie une **version neutralisée** du digest sur ce dépôt (`digests/digest-AAAA-MM-JJ.html`) via l'API Contents de GitHub, servie ensuite par GitHub Pages.
6. Vérifie que la page publiée répond bien en HTTP 200 avant d'envoyer le lien sur la messagerie d'équipe.

La neutralisation (étape 5) a lieu dans `Preparer Publication` : c'est le seul point où l'e-mail interne et la page publique divergent.

## Déploiement (GitHub Pages)

Le site est publié via **GitHub Actions** (`.github/workflows/pages.yml`), et non via le builder « legacy » historique.

Historique : le builder legacy est déjà resté bloqué en `building` pendant ~50 minutes, rendant le digest du jour inaccessible (404) alors qu'il était bien commité. Le déploiement via Actions est plus rapide (~15-20 s) et son statut est vérifiable (onglet *Actions* du dépôt), contrairement au builder legacy qui pouvait rester bloqué silencieusement.

La page d'accueil liste les digests en interrogeant l'API Contents de GitHub depuis le navigateur (`/repos/:owner/:repo/contents/digests`), sans authentification ni build. **Cela suppose que le dépôt reste public** : s'il repassait en privé, l'API répondrait 404 et la liste afficherait son message de repli — les digests resteraient joignables par lien direct. Le quota anonyme est de 60 requêtes/h par IP, largement suffisant à cette échelle.

Autre incident constaté : passer le dépôt en *private* **dépublie le site** (Pages indisponible sur dépôt privé en offre gratuite). Symptôme : `has_pages: false` et 404 sur l'URL, alors que le dernier workflow a réussi. Correctif : repasser le dépôt en public, puis réactiver Pages en mode Actions (`POST /repos/:owner/:repo/pages` avec `build_type=workflow`) — la réactivation n'est pas automatique.

## Fiabilité du lien de notification

À l'origine, le workflow attendait une durée fixe (1 minute) après le commit avant d'envoyer le lien du digest sur la messagerie d'équipe — sans vérifier que la page était réellement en ligne. En cas de déploiement lent ou bloqué côté GitHub Pages, le lien envoyé pouvait renvoyer une 404.

Le workflow vérifie désormais la disponibilité de la page avant de notifier :

- Attente initiale de 15 s après le commit.
- Vérification HTTP de la page publiée (`Verifier page en ligne`).
- Si la page répond 200 → envoi immédiat du message.
- Sinon → nouvelle tentative toutes les 15 s, jusqu'à 10 fois (~2,5 minutes max).
- Au-delà, le message est envoyé quand même (pour ne jamais rester silencieux), la situation étant alors anormale et à investiguer manuellement.
