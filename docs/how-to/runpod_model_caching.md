# Déployer avec RunPod Model Caching

Ce guide explique comment déployer des modèles HuggingFace sur RunPod Serverless en utilisant le **Model Caching** natif. Il documente les pièges qu'on a rencontrés et qu'on n'a trouvés nulle part dans la documentation officielle.

## Notre parcours : des images lourdes au Model Caching

### Phase 1 : modèles embarqués dans les images Docker

Au début, on embarquait les poids des modèles directement dans les images Docker. Simple, mais les images faisaient **12-15 GB** :

| Service | Modèles dans l'image | Taille image |
|---------|---------------------|-------------|
| Speech | BiCodec (1.77 GB) + wav2vec2 + LLM (0.96 GB) | ~12 GB |
| Transcription | Whisper V2 + V3 (2.88 GB chacun) | ~13 GB |
| Translation | Mistral-7B 4bit (3.85 GB) + NLLB (2.33 GB) | ~15 GB |

Résultat : des **cold starts de 2-5 minutes**. L'inférence elle-même prenait 3 secondes, mais les utilisateurs attendaient des minutes parce que RunPod devait puller 15 GB à chaque démarrage de worker.

### Phase 2 : Network Volumes

On a migré vers les [Network Volumes](https://docs.runpod.io/serverless/endpoints/manage-volumes) — les modèles sur un volume partagé, les images Docker ne contiennent que le code (~4-5 GB). Cold starts réduits à **40-60 secondes**.

Mais un nouveau problème est apparu : les volumes sont liés à une **région spécifique**. Quand le stock de GPU RTX A4000 s'est épuisé en EUR-NL-1 (notre région), tous nos endpoints étaient bloqués. Workers throttled, jobs en queue infinie, impossible de servir les utilisateurs.

### Phase 3 : Model Caching (actuel)

Le [Model Caching](https://docs.runpod.io/serverless/endpoints/model-caching) natif de RunPod résout ce problème : RunPod pré-télécharge le modèle sur la machine hôte **avant** le démarrage du worker, sans contrainte de région. Plus de volume à gérer, plus de blocage régional.

| | Images lourdes | Network Volume | Model Caching |
|---|---|---|---|
| Cold start | 2-5 min | 40-60s | **10-30s** |
| Contrainte de région | Non | **Oui** | Non |
| Coût mensuel | $0 | ~$1.75/25GB | $0 |
| Provisioning | Automatique | Manuel | Automatique |
| Taille image Docker | 12-15 GB | 4-5 GB | **3-4 GB** |

## Architecture : 1 endpoint = 1 modèle

Le Model Caching RunPod ne supporte qu'**un seul modèle HuggingFace par endpoint**. Si vous avez plusieurs modèles, créez un endpoint par modèle :

| Endpoint | Modèle HF | Service |
|----------|-----------|---------|
| burkimbia-speech | `burkimbia/BIA-SPARKTTS-V4` | TTS |
| burkimbia-transcription | `burkimbia/BIA-WHISPER-LARGE-SACHI_V2` | ASR |
| burkimbia-translation-mistral | `burkimbia/BIA-MISTRAL-7B-SACHI_4bit` | Traduction |
| burkimbia-translation-nllb | `burkimbia/BIA-NLLB-600M-10E-CD-CRCL` | Traduction |

## Le handler : charger le modèle au startup

Le point le plus important : le modèle doit être chargé **une seule fois au démarrage du module**, pas à chaque requête.

!!! danger "Anti-pattern : chargement par requête"
    ```python
    def handler(job):
        # MAL — recharge le modèle à chaque requête
        model = load_model(model_path)
        result = model.inference(job["input"]["text"])
        return result
    ```

!!! success "Pattern correct : chargement au startup"
    ```python
    # Chargé UNE FOIS au démarrage du worker
    MODEL_ID = os.environ.get("MODEL_ID", "burkimbia/BIA-SPARKTTS-V4")
    model_path = resolve_snapshot_path(MODEL_ID)
    model = load_model(model_path)

    def handler(job):
        # Utilise le modèle déjà en mémoire GPU
        result = model.inference(job["input"]["text"])
        return result
    ```

## Le chemin du cache

RunPod Model Caching télécharge les modèles dans une structure HuggingFace cache standard :

```
/runpod-model-cache/hub/
└── models--burkimbia--bia-sparktts-v4/
    ├── refs/
    │   └── main          # contient le hash du commit
    └── snapshots/
        └── 38dc4b62.../  # fichiers du modèle
```

Votre handler doit résoudre ce chemin. Voici la fonction `resolve_snapshot_path` :

```python
HF_CACHE_ROOTS = [
    "/runpod-model-cache/hub",
    "/runpod-model-cache",
]

def resolve_snapshot_path(model_id: str) -> str:
    org, name = model_id.split("/", 1)
    folder_name = f"models--{org}--{name}"

    for cache_root in HF_CACHE_ROOTS:
        model_root = os.path.join(cache_root, folder_name)
        if not os.path.isdir(model_root):
            continue

        refs_main = os.path.join(model_root, "refs", "main")
        snapshots_dir = os.path.join(model_root, "snapshots")

        if os.path.isfile(refs_main):
            with open(refs_main) as f:
                snapshot_hash = f.read().strip()
            candidate = os.path.join(snapshots_dir, snapshot_hash)
            if os.path.isdir(candidate):
                return candidate

    raise RuntimeError(f"Model '{model_id}' not found in cache")
```

## Le mode offline

Activez le mode offline pour empêcher les téléchargements accidentels au runtime :

```python
os.environ["HF_HUB_OFFLINE"] = "1"
os.environ["TRANSFORMERS_OFFLINE"] = "1"
```

Ces variables doivent aussi être dans votre **template RunPod** (section Environment Variables).

---

## Les pièges

### Piège 1 : le case mismatch (MODEL_NAME vs MODEL_ID)

C'est le piège le plus vicieux. RunPod injecte deux variables d'environnement :

| Variable | Valeur | Source |
|----------|--------|--------|
| `MODEL_ID` | `burkimbia/BIA-SPARKTTS-V4` | Votre template |
| `MODEL_NAME` | `burkimbia/bia-sparktts-v4` | RunPod (lowercase) |

Le problème : RunPod utilise `MODEL_NAME` (lowercase) pour télécharger le modèle. Le dossier cache sera donc :

```
models--burkimbia--bia-sparktts-v4   # lowercase
```

Mais si votre code construit le chemin avec `MODEL_ID` :

```
models--burkimbia--BIA-SPARKTTS-V4   # original case → pas trouvé !
```

**Résultat** : le modèle n'est pas trouvé, le handler crash, le worker est marqué **throttled**.

**Solution** : cherchez aussi avec `MODEL_NAME`, ou faites un scan case-insensitive :

```python
# Essayer MODEL_ID et MODEL_NAME (injecté par RunPod)
model_name_env = os.environ.get("MODEL_NAME", "")
ids_to_try = {model_id}
if model_name_env and "/" in model_name_env:
    ids_to_try.add(model_name_env)

for mid in ids_to_try:
    org, name = mid.split("/", 1)
    folder = f"models--{org}--{name}"
    # ... chercher dans les cache roots
```

### Piège 2 : "throttled" ne veut pas dire "pas de GPU"

Quand vous voyez vos workers marqués **throttled** dans l'UI RunPod, votre premier réflexe est de penser que les GPU sont en rupture de stock. En réalité, **throttled = le worker crash en boucle**.

RunPod essaie de relancer le worker → il crash → RunPod le throttle pour ne pas gaspiller des ressources.

Les causes courantes :

- Le modèle n'est pas trouvé dans le cache (piège 1)
- Une dépendance Python manquante (ex: `psutil`, `ffmpeg`)
- OOM — le modèle ne tient pas dans la VRAM du GPU

**Comment diagnostiquer** : allez dans l'UI → Workers → cliquez sur le worker → Logs. Cherchez les erreurs Python (traceback, ModuleNotFoundError, RuntimeError).

### Piège 3 : le container registry privé

Si votre image Docker est sur un registry **privé** (Docker Hub privé, GHCR privé), vous devez configurer les **Container Registry Credentials** dans votre template RunPod. Sinon :

```
error creating container: pull access denied for burkimbia/burkimbia-ai-services,
repository does not exist or may require 'docker login': denied
```

Allez dans RunPod → Templates → votre template → **Container Registry Credentials** → sélectionnez vos credentials.

### Piège 4 : le Model Caching se configure UNIQUEMENT dans l'UI

C'est la limitation la plus frustrante. Le Model Caching **ne se configure pas** via l'API REST RunPod, ni via le MCP, ni dans le template. Il n'existe **aucun moyen de l'automatiser dans une CI/CD**. Il faut aller manuellement dans l'UI RunPod :

1. Endpoint → **Manage** → **Edit Endpoint**
2. Section **Model**
3. Entrer le nom exact du modèle HuggingFace
4. Ajouter votre HF token (pour les repos privés)
5. Sauvegarder

!!! warning "Impact sur le workflow de déploiement"
    Cela signifie que **chaque fois que vous supprimez et recréez un endpoint** (par exemple pour forcer le pull d'une nouvelle image Docker), vous devez **refaire cette configuration manuellement**. C'est une étape qu'il est facile d'oublier, et si vous l'oubliez : le worker démarre en mode offline, ne trouve aucun modèle dans le cache, crash silencieusement, et est marqué "throttled".

    On espère que RunPod exposera ce paramètre dans l'API à l'avenir. En attendant, ajoutez-le à votre checklist de déploiement.

---

## Debugging avec le MCP RunPod dans Claude Code

Vous pouvez inspecter et gérer vos endpoints RunPod directement depuis Claude Code grâce au [MCP Server RunPod](https://docs.runpod.io/get-started/mcp-servers).

### Installation

```bash
claude mcp add runpod --scope user \
  -e RUNPOD_API_KEY=votre_clé_api \
  -- npx -y @runpod/mcp-server@latest
```

Relancez Claude Code, puis vérifiez avec `/mcp`.

### Ce que vous pouvez faire

**Lister les endpoints et leurs workers :**

Demandez simplement à Claude : *"montre-moi l'état de mes endpoints RunPod"*. Claude utilisera le MCP pour appeler `list-endpoints` et vous afficher le statut, les GPU, les workers.

**Inspecter un worker throttled :**

```
"inspecte l'endpoint speech, je veux voir les env vars du worker"
```

Claude appelle `get-endpoint` avec `includeWorkers: true` et vous montre les variables d'environnement injectées par RunPod — y compris `MODEL_NAME` (pour vérifier le case mismatch).

**Recréer un endpoint :**

```
"supprime l'endpoint speech et recrée-le avec le même template"
```

Utile après un rebuild Docker pour forcer le pull de la nouvelle image (FlashBoot cache l'ancienne).

**Diagnostiquer le case mismatch :**

En inspectant les env vars du worker, vous pouvez voir :

```json
{
  "MODEL_ID": "burkimbia/BIA-SPARKTTS-V4",
  "MODEL_NAME": "burkimbia/bia-sparktts-v4"
}
```

Si `MODEL_ID` ≠ `MODEL_NAME` en termes de casse, c'est la source du problème.

---

## Déployer une nouvelle release avec FlashBoot

Quand vous rebuild votre image Docker avec du nouveau code, les workers FlashBoot continuent d'utiliser l'**ancien snapshot en cache**. C'est le comportement attendu — FlashBoot est conçu pour restaurer l'état rapidement, pas pour vérifier si l'image a changé.

### Le problème

```
1. Vous fixez un bug dans le code
2. Vous rebuild et push speech-latest sur Docker Hub
3. Vous lancez une requête → le worker FlashBoot démarre en ~1s
4. ... avec l'ancien code
```

### La solution : modifier le template

Pour invalider le cache FlashBoot, il faut **modifier le template** de l'endpoint. Tout changement dans la configuration du template (image, env vars) force RunPod à recréer un snapshot frais.

**Option 1 : via le MCP RunPod** (recommandé)

```
"mets à jour le template hznc9i3vuz avec IMAGE_VERSION=3"
```

Claude appelle `update-template` avec la nouvelle env var, ce qui invalide les snapshots.

**Option 2 : via l'UI RunPod**

1. Templates → votre template → **Edit**
2. Ajoutez ou incrémentez une variable : `IMAGE_VERSION=3`
3. Save

**Option 3 : via l'API GraphQL**

```python
mutation = """
mutation saveTemplate($input: SaveTemplateInput!) {
    saveTemplate(input: $input) { id }
}"""
variables = {
    "input": {
        "id": "votre_template_id",
        "env": [
            {"key": "IMAGE_VERSION", "value": "3"},
            # ... autres env vars existantes
        ]
    }
}
```

### Workflow complet de release

```
1. git push → CI build → Docker push (speech-latest)
2. Mettre à jour le template (IMAGE_VERSION++)
3. Première requête → cold start (~40-60s, nouvelle image)
4. FlashBoot crée un nouveau snapshot
5. Requêtes suivantes → ~1-5s (nouveau snapshot)
```

!!! tip "Pas besoin de rolling update"
    Contrairement aux déploiements Kubernetes, il n'y a pas de stratégie de rolling update à configurer. Avec FlashBoot + `min_workers: 0`, le cycle est naturel : pas de trafic → worker spin down → prochain démarrage = nouvelle image.

---

## Checklist de déploiement

Avant de déployer un nouvel endpoint, vérifiez :

- [ ] **Handler** : le modèle est chargé au **startup**, pas dans la fonction handler
- [ ] **Mode offline** : `HF_HUB_OFFLINE=1` et `TRANSFORMERS_OFFLINE=1` dans le template
- [ ] **Model Caching** : configuré dans l'**UI** de l'endpoint (pas dans le template)
- [ ] **Container Registry** : credentials configurées dans le template (si image privée)
- [ ] **Case mismatch** : `resolve_snapshot_path` gère les noms lowercase
- [ ] **Dépendances** : toutes les dépendances transitives sont dans `pyproject.toml` (ex: `psutil`)
- [ ] **Container disk** : assez grand pour l'image + le modèle caché (≥20 GB)
- [ ] **GPU VRAM** : suffisante pour le modèle (vérifier la taille en bfloat16/float16)

## Résultats

Avec cette configuration, nos cold starts sont passés de **2-5 minutes** (Network Volume) à **10-30 secondes** (Model Caching), et l'inférence prend **1-5 secondes** selon le service :

| Service | Cold start | Inference |
|---------|-----------|-----------|
| Translation NLLB | ~28s | ~1s |
| Translation Mistral | ~11s | ~1.2s |
| Transcription Whisper | ~1.4s (warm) | ~5s/12s audio |
| Speech SparkTTS | ~126s (1er), puis ~30s | ~24s |
