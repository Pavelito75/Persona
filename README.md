# Projet Persona : AI Agent & Newsletter Automatisée

Ce projet implémente un chatbot d'accueil intelligent avec persistance de profil et un moteur automatisé de veille d'actualités personnalisées sous forme de newsletter HTML.

L'infrastructure repose sur **n8n** hébergé localement sous Docker, couplé à un métamoteur de recherche **SearXNG** et au modèle de langage **Google Gemini**.

---

## 1. Prérequis & Installation de la Stack Docker

### Fichiers requis
Assurez-vous que le répertoire racine du projet contient :
- `compose.yml` (ou `docker-compose.yml`)
- `.env`
- `searxng-settings.yml`
- `index.html` (Interface web de démonstration intégrant le widget `@n8n/chat`)

### Configuration réseau essentielle (`compose.yml`)
Pour éviter les erreurs de résolution DNS (notamment `EAI_AGAIN`) lors des appels externes vers les API de modèles d'IA et forcer l'IPv4 au sein du conteneur Node.js n8n, appliquez la configuration suivante sur le service `n8n` :

```yaml
services:
  n8n:
    image: docker.io/n8nio/n8n:${N8N_VERSION}
    depends_on:
      sandbox-api:
        condition: service_healthy
    dns:
      - 1.1.1.1
      - 8.8.8.8
    environment:
      - NODE_OPTIONS=--dns-result-order=ipv4first
      - GENERIC_TIMEZONE=Europe/Paris
      - TZ=Europe/Paris
    ports:
      - "5678:5678"
    env_file: .env
    volumes:
      - n8n-data:/home/node/.n8n
```

### Démarrage des conteneurs
```bash
# Lancement de l'ensemble de la stack
docker compose down && docker compose up -d

# Vérification du bon fonctionnement
docker ps
```
Accédez ensuite à l'interface d'administration n8n sur : **`http://localhost:5678`**.

---

## 2. Configuration du Modèle de Langage (IA) & SMTP

Le projet utilise **Google Gemini** (`gemini-3.5-flash-lite` via `Google Gemini Chat Model`) :
- **Quota gratuit élevé :** 1 000 requêtes / jour (RPD) et 30 requêtes / minute (RPM), idéal pour supporter à la fois le chatbot conversationnel et la génération de newsletters multi-utilisateurs.
- **Vitesse de traitement :** Inférence en moins de 500 ms assurant une excellente fluidité lors des démonstrations en direct.
- **Fiabilité :** Prise en charge stricte du formatage JSON via le nœud `Structured Output Parser` et du Function Calling pour les Tools n8n.

### Ajout des Credentials dans n8n
1. **Google Gemini API :**
   - Créez une clé d'API sur [Google AI Studio](https://aistudio.google.com/) (section **API Keys**).
   - Dans n8n, ouvrez le panneau latéral gauche $\rightarrow$ **Credentials** $\rightarrow$ **Add Credential** $\rightarrow$ **Google Gemini(PaLM) Api** et renseignez votre clé.
2. **Serveur d'envoi d'emails (SMTP / Gmail) :**
   - Ajoutez un Credential **SMTP** (ou Gmail OAuth2) utilisé par le nœud `Send an Email` du Workflow 2 pour expédier les newsletters.

> **Sécurité :** Ne committez jamais de clés d'API brutes dans le dépôt Git ou dans les exports JSON de workflows.

---

## 3. Modèle de Données (n8n Data Tables)

Le projet exploite deux tables relationnelles internes dans n8n.

### Table : `users`
Stocke les informations de compte, les préférences de veille et l'état de l'abonnement.

| Colonne | Type | Description |
| :--- | :--- | :--- |
| `email` | String | Identifiant unique de l'utilisateur (clé logique) |
| `password_hash` | String | Empreinte cryptographique **SHA-256** du mot de passe |
| `topics` | String | Sujets de veille sélectionnés (ex: *science et football*) |
| `send_time` | String | Heure d'envoi programmée aux formats `HH:mm` ou `HHhMM` (ex: *17h10*, *20:30*) |
| `custom_prompt` | String | Consignes de ton et de style pour l'IA (ex: *de manière fun*, *sérieux*) |
| `gdpr_consent` | Boolean | Consentement explicite au traitement des données (`true` obligatoire) |
| `is_active` | Boolean | État de l'abonnement (`true` = actif, `false` = en pause) |
| `last_sent_at` | String / Date | Horodatage ISO du dernier envoi de la newsletter (anti-doublon quotidien) |

### Table : `sessions`
Gère la persistance de l'authentification pour aiguiller les interactions dans le chat.

| Colonne | Type | Description |
| :--- | :--- | :--- |
| `session_id` | String | Identifiant de session généré par le chat n8n (`sessionId`) |
| `user_email` | String | Email du compte rattaché à la session |
| `is_authenticated` | Boolean | Statut de validation de la session (`true` ou `false`) |
| `auth_step` | String | Étape du tunnel de connexion (`authenticated`, etc.) |
| `expires_at` | Number | Timestamp d'expiration de la session |

---

## 4. Architecture : Workflow 1 (`auth_workflow` — Chatbot & Authentification)

### Schéma logique
```text
[When chat message received]
             │
             ▼
    [Get row(s): sessions]
   (session_id = sessionId)
             │
             ▼
        [If Node]
  (session_id is not empty)
        │         │
 [True] │         │ [False]
        ▼         ▼
 [Persona Agent] [Auth Agent]
```

### Fonctionnement détaillé

1. **Déclencheur :** Le nœud `When chat message received` (rendu public via `Make Chat Publicly Available`) intercepte chaque message et fournit un identifiant `sessionId` unique.
2. **Vérification de session :** Le nœud Data Table `Get row(s)` (avec l'option `Always Output Data` activée) recherche une ligne dans la table `sessions` où :
   - `session_id` = `{{ $('When chat message received').first().json.sessionId }}`
3. **Aiguillage conditionnel (`If`) :**
   - **Condition :** `{{ $json.session_id }}` $\rightarrow$ **is not empty**.
   - **Branche False $\rightarrow$ `Auth Agent` :**
     - **Inscription (`Create User`) :** Collecte l'email, le mot de passe, les sujets, l'heure d'envoi, le style et le consentement RGPD. Le mot de passe est haché à la volée en **SHA-256** via l'expression :
       ```javascript
       {{ String($fromAI('password_hash', 'Mot de passe de l utilisateur', 'string') || '').hash('sha256') }}
       ```
     - **Connexion (`Get User`) :** Vérifie simultanément l'`email` et la correspondance du hash **SHA-256** du mot de passe fourni :
       ```javascript
       {{ String($fromAI('password', 'Mot de passe fourni par l utilisateur', 'string') || '').hash('sha256') }}
       ```
     - **Ouverture de session (`Save Session`) :** Enchaîné automatiquement après `Create User` ou `Get User`, il insère une ligne dans `sessions` avec `session_id` fixé sur `{{ $('When chat message received').first().json.sessionId }}`.
   - **Branche True $\rightarrow$ `Persona Agent` :**
     - Reconnaît l'utilisateur déjà authentifié et récupère son profil (`Get User Profile`).
     - Appelle le Tool `Update Users` pour modifier `topics`, `send_time`, `custom_prompt` ou `is_active` (en ne ciblant que les colonnes modifiées pour préserver `gdpr_consent` et `is_active`).
     - Gère la déconnexion (`Logout Session` $\rightarrow$ suppression dans `sessions`) et le droit à l'oubli (`Delete Account` $\rightarrow$ suppression dans `users` et `sessions`).

---

## 5. Architecture : Workflow 2 (`Newsletter Workflow` — Pipeline Automatisé)

### Schéma logique
```text
[Schedule Trigger (1 min)] ──► [Get Active Users] ──► [Split Topics] ──► [Search News (SearXNG)]
                                                                                   │
[Update send date] ◄── [Send an Email] ◄── [Check Newsletter] ◄── [Build Newsletter] ◄── [Generate Newsletter (Gemini)] ◄── [Group Articles by User] ◄── [Filter News Results]
```

### Fonctionnement détaillé étape par étape

1. **Déclenchement temporel (`Schedule Trigger`) :** S'exécute automatiquement toutes les minutes en arrière-plan lorsque le workflow est à l'état **Published**.
2. **Récupération des abonnés (`Get Active Users`) :** Sélectionne dans la table `users` tous les profils ayant `is_active = true` et `gdpr_consent = true`.
3. **Filtrage horaire & Découpage des sujets (`Split Topics`) :**
   - Convertit l'heure courante sur le fuseau **`Europe/Paris`** et normalise le champ `send_time` (supporte `17h30`, `17:30`, `8h30`).
   - Vérifie deux conditions : l'heure programmée est atteinte (`currentHHMM >= userTime`) **et** l'utilisateur n'a pas encore reçu sa newsletter aujourd'hui (`last_sent_at` différent de la date du jour `YYYY-MM-DD`).
   - Sépare les sujets multiples via l'expression régulière `/,|;|\s+et\s+|\s+and\s+/i` et génère un item par couple `(utilisateur, sujet)`.
4. **Collecte d'actualités (`Search News` & `Filter News Results`) :**
   - Interroge l'API JSON locale de **SearXNG** (`http://searxng:8080/search`) pour chaque sujet.
   - Nettoie et filtre les résultats pertinents.
5. **Agrégation et Dédoublonnage (`Group Articles by User`) :**
   - Regroupe tous les articles par utilisateur (`user_id` / `email`), élimine les URLs en doublon entre différents sujets et calcule les statistiques de collecte.
6. **Rédaction IA & Mise en page (`Generate Newsletter` & `Build Newsletter`) :**
   - Le nœud `Generate Newsletter` (propulsé par **Google Gemini** et un `Structured Output Parser`) synthétise les articles en respectant le ton défini dans `custom_prompt`.
   - Le nœud `Build Newsletter` assemble le rendu final en **HTML responsive** prêt à l'envoi.
7. **Vérification, Distribution & Traçabilité (`Check Newsletter` $\rightarrow$ `Send an Email` $\rightarrow$ `Update send date`) :**
   - Si la newsletter est valide (`Check Newsletter`), le nœud `Send an Email` l'expédie au destinataire.
   - Le nœud `Update send date` met **uniquement** à jour la colonne `last_sent_at` (`{{ $now.toISO() }}`) dans la table `users`, évitant ainsi tout envoi en doublon sur le reste de la journée sans écraser les booléens `is_active` et `gdpr_consent`.

---

## 6. Règles de Sécurité & Conformité RGPD

- **Hachage systématique des mots de passe :** Aucun mot de passe n'est stocké ni comparé en clair. Le hachage **SHA-256** est appliqué directement au niveau des expressions n8n des Tools `Create User` et `Get User`.
- **Protection des identifiants sensibles :** L'agent Persona n'est pas autorisé à modifier l'adresse email ou le mot de passe dans le chat.
- **Affectation stricte du Session ID :** Le champ `session_id` dans l'outil `Save Session` n'est jamais laissé à l'improvisation du LLM ; il est injecté depuis le déclencheur via `{{ $('When chat message received').first().json.sessionId }}`.
- **Préservation de l'intégrité lors des mises à jour :** Les nœuds `Update` sur la table `users` ne déclarent que les colonnes strictement modifiées afin de ne jamais réinitialiser accidentellement `gdpr_consent` ou `is_active` à `false`.

---

## 7. Interface Web (`index.html`) & Guide de Soutenance

Une page d'accueil documentaire au design inspiré de `react.dev` (`index.html`) embarque le widget officiel `@n8n/chat` connecté au webhook de production du `auth_workflow`.

### Procédure de lancement pour la démonstration
1. Dans n8n, vérifier que **`auth_workflow`** et **`Newsletter Workflow`** sont tous les deux en mode **Published** (bouton en haut à droite).
2. Copier la **Chat URL** de production du nœud `When chat message received` et la renseigner dans la propriété `webhookUrl` du fichier `index.html`.
3. Ouvrir `index.html` dans le navigateur :
   - **Test Inscription :** *"Inscris-moi avec demo@epitech.eu, mot de passe secret123, sujets science et football à [Heure actuelle + 2 min], de manière fun, j'accepte le RGPD"*
   - **Vérification Base de données :** Montrer dans la table `users` le mot de passe haché en SHA-256 et les cases `gdpr_consent` / `is_active` cochées.
   - **Test Modification (`Persona Agent`) :** *"Modifie mon heure d'envoi à [Heure actuelle + 1 min]"*
   - **Réception & Traçabilité :** Suivre l'exécution automatique dans l'onglet **Executions** du `Newsletter Workflow`, présenter l'email HTML reçu et vérifier la mise à jour de `last_sent_at` dans la table `users`.