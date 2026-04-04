# Déployer avec RunPod Model Caching

Ce guide explique comment déployer des modèles HuggingFace sur RunPod Serverless en utilisant le **Model Caching** natif, sans Network Volume.

## Pourquoi Model Caching ?

Les Network Volumes sont liés à une **région spécifique**. Si le stock GPU est épuisé dans cette région, les endpoints sont bloqués. Le Model Caching natif de RunPod résout ce problème : RunPod pré-télécharge le modèle sur la machine hôte **avant** le démarrage du worker, sans contrainte de région.

| | Network Volume | Model Caching |
|---|---|---|
| Contrainte de région | Oui | Non |
| Coût mensuel | ~$1.75/25GB | Gratuit |
| Provisioning | Manuel | Automatique |
| Multi-région | Non | Oui |

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

### Piège 4 : le Model Caching se configure dans l'UI

Le Model Caching **ne se configure pas** dans le template ni via l'API. Il faut aller dans l'UI RunPod :

1. Endpoint → **Manage** → **Edit Endpoint**
2. Section **Model**
3. Entrer le nom exact du modèle HuggingFace
4. Ajouter votre HF token (pour les repos privés)

Sans cette étape, le worker démarre en mode offline mais ne trouve aucun modèle → crash.

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
| Speech SparkTTS | à mesurer | à mesurer |
