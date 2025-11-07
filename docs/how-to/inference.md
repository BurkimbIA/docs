# Documentation API BurkimbIA - Services d'Inférence

## Repositories Importants

**Pour modifier nos API, vous devez connaître ces deux repositories :**

- **`inference-endpoints`** (prioritaire) : Contient les modèles et le code de déploiement sur RunPod
- **`bia-backend`** : API Gateway déployé sur Fly.io

⚠️ **Important** : Toute modification dans `inference-endpoints` doit être répercutée dans `bia-backend` si nécessaire.

---

## Architecture

<!-- L'infrastructure repose sur deux composants principaux :
- inference-endpoints : Repository contenant les modèles et le code pour les servir sur RunPod (serverless)
- bia-backend : Application FastAPI sur Fly.io qui gère l'authentification, la journalisation et le rate limiting -->

```mermaid
flowchart TD
    %% Define styles for different components
    %% Bleu pour le Client, Violet pour bia-backend, Vert pour inference-endpoints
    classDef userStyle fill:#e1f5fe,stroke:#0277bd,stroke-width:3px,color:#000
    classDef backendStyle fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000
    classDef inferenceStyle fill:#e8f5e8,stroke:#388e3c,stroke-width:3px,color:#000
    
    %% Components
    U[👤 Client<br/>Applications tierces]
    A[🔐 Auth Service<br/>bia-backend<br/>JWT / API Key]
    B[⚙️ API Gateway<br/>bia-backend<br/>Rate Limiting & Logging]
    T[🌐 Translation<br/>inference-endpoints<br/>RunPod]
    S[🎤 Transcription<br/>inference-endpoints<br/>RunPod]
    V[🔊 TTS<br/>inference-endpoints<br/>RunPod]
    
    %% Relationships - Flux numéroté de 1 à 6
    U -.->|"1️⃣ Authenticate"| A
    U -->|"2️⃣ Request"| B
    B -.->|"3️⃣ Verify"| A
    B ==>|"4️⃣ Forward"| T
    B ==>|"4️⃣ Forward"| S
    B ==>|"4️⃣ Forward"| V
    T ==>|"5️⃣ Response"| B
    S ==>|"5️⃣ Response"| B
    V ==>|"5️⃣ Response"| B
    B -->|"6️⃣ Result"| U
    
    %% Apply styles
    class U userStyle
    class A,B backendStyle
    class T,S,V inferenceStyle
```

![Architecture Backend](assets/backend.png)

---

## Services et Modèles

<!-- Liste des modèles ML utilisés par service -->

| Service       | Modèle                               | Taille | Fonction                    |
|---------------|--------------------------------------|--------|-----------------------------|
| Translation   | `burkimbia/BIA-NLLB-600M-5E`         | 600M   | Traduction Français↔Mooré   |
| Translation   | `burkimbia/BIA-MISTRAL-7B-SACHI`     | 7B     | Traduction instruction-tuned|
| Transcription | `burkimbia/BIA-WHISPER-LARGE-SACHI_V2`| 1.5B  | Audio→Texte                 |
| TTS           | `burkimbia/BIA-SPARK-TTS-V2`         | 0.5B   | Texte→Audio                 |

---

## Déploiement sur RunPod

### Procédure

<!-- Étapes du déploiement manuel via l'interface RunPod -->

1. **Déployer l'image Docker** via l'interface RunPod

![Étape 1 - Déploiement](assets/deploy_1.png)

2. **Sélectionner l'option de service** appropriée

![Étape 2 - Options](assets/deploy_2.png)

3. **Lancer la CI** et attendre le succès du build

![Étape 3 - CI](assets/deploy_3.png)

4. **Vérifier le statut du worker** (passe de `initializing` à `idle`/`running`)

![Étape 4 - Statut](assets/deploy_4.png)

5. **Récupérer le service ID** depuis l'interface RunPod ou les logs CI

![Étape 5 - Service ID](assets/deploy_5.png)

![Endpoints sur RunPod](assets/endpoints_on_rundpods.png)

### Configuration

<!-- Le fichier runpod-config.json définit les endpoints, images Docker et ressources GPU -->

⚠️ **Après chaque déploiement** : Le service ID change. Mettre à jour les variables d'environnement dans `bia-backend`.

### Appel direct possible (mais non utilisé)

<!-- Il est techniquement possible d'appeler directement les endpoints RunPod -->

```bash
# Exemple d'appel direct RunPod (pour référence uniquement)
curl -X POST https://api.runpod.ai/v2/{endpoint_id}/run \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer TOKEN_RUNPOD' \
    -d '{
      "input": {
        "text": "Bonjour le monde",
        "model_id": "burkimbia/BIA-NLLB-1.3B-7E",
        "src_lang": "fr_Latn",
        "tgt_lang": "moor_Latn"
      }
    }'
```

**Problème** : RunPod fournit une seule API key pour tous les services, sans branding ni contrôle d'accès personnalisé.

**Solution** : C'est pourquoi nous utilisons `bia-backend` comme API Gateway, qui permet :
- Gestion multi-utilisateurs avec authentification
- Branding personnalisé (`api.burkimbia.com`)
- Rate limiting par utilisateur
- Logs et monitoring centralisés

---

## Utilisation via bia-backend

<!-- L'API Gateway est le point d'entrée recommandé pour tous les services -->

```bash
curl -X POST https://api.burkimbia.com/v1/translate \
  -H "Authorization: Bearer BIA_VOTRE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Bonjour",
    "src_lang": "fr",
    "tgt_lang": "moore"
  }'
```

<!-- Les tokens JWT ou clés API sont obtenus via le backend -->

---

## Debugging

### Backend (bia-backend)

<!-- Utiliser Grafana pour les logs détaillés -->

Utiliser les logs **Grafana** sur Fly.io :
- URL : https://fly.io/apps/bia-backend/monitoring
- ⏱️ Le chargement prend du temps mais fournit des informations détaillées

### Services d'inférence (RunPod)

<!-- Aller dans Manage Serverless puis sélectionner le service -->

**Playground** : Tester les requêtes interactivement

![Playground RunPod](assets/playground.png)

**Logs** : Consulter les logs d'exécution

![Logs RunPod](assets/logs.png)

**Workers Status** : Vérifier l'état des workers

---

## Erreurs Courantes (GitHub Actions CI/CD)

### 1. Limite de workers dépassée

```bash
Checking for existing endpoint...
Creating endpoint...
❌ Error creating endpoint: 500 Server Error: Internal Server Error for url: https://rest.runpod.io/v1/endpoints
Response: {"error":"create endpoint: create endpoint: graphql: Max workers across all endpoints must not exceed your workers quota (5). Reduce the max workers for other endpoints or lower the max worker count for this endpoint to at most 0.","status":500}
```

**Cause** : Pour les comptes tiers comme le nôtre, nous sommes limités à 5 workers maximum.

**Solution** : Supprimer quelques workers existants sur RunPod avant de redéployer.

### 2. Template introuvable (service non prêt)

```bash
Checking for existing endpoint...
Creating endpoint...
❌ Error creating endpoint: 500 Server Error: Internal Server Error for url: https://rest.runpod.io/v1/endpoints
Response: {"error":"create endpoint: create endpoint: graphql: Unable to find template","status":500}
```

**Cause** : Le service n'est pas encore prêt sur RunPod.

**Solution** : Relancer simplement la CI de redéploiement.

---

## Authentification

<!-- Le backend supporte plusieurs modes d'authentification -->

- Email/mot de passe
- OAuth Google
- Clés API pour applications tierces