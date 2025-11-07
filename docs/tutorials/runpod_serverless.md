# Guide exhaustif de déploiement RunPod

Ce guide documente toutes les options pratiques pour déployer vos services BurkimbIA sur RunPod, qu'il s'agisse de simples tests manuels depuis le tableau de bord ou d'un pipeline complet Infrastructure-as-Code (IaC). Nous nous concentrons ici sur le service de traduction français <-> mooré afin de garder l'exemple concret et reproductible.

---

## 1. Comprendre le contexte RunPod

RunPod propose trois approches complémentaires :
- **Console RunPod** : interface graphique "Deploy" et "Create Endpoint" pour un démarrage rapide.
- **GitHub + RunPod** : RunPod peut consommer un repository GitHub et déclencher vos builds (RunPod Deploys).
- **API RunPod** : appels REST permettant d'automatiser totalement la création de templates et d'endpoints (IaC).

Vous n'êtes pas obligé d'utiliser notre code Python interne (`RunPodClient`) ni nos workflows GitHub. Ce guide vous décrit chaque méthode pas à pas et liste les problèmes fréquents rencontrés lors de notre retour d'expérience.

---

## 2. Prérequis généraux

- **Image Docker construite et poussée** sur Docker Hub ou un registre privé (taille optimisée < 5 GB si possible). **Crucial** : intégrer le modèle (ici NLLB) dans l'image pour éviter les téléchargements à chaque cold start.
- **Variables sensibles** : `RUNPOD_API_KEY`, `HF_TOKEN` (pour accès modèles privés), identifiants éventuels vers stockage S3/MinIO.
- **Fichiers BurkimbIA** : `runpod-config.json`, `deploy_runpod.py`, `services/translation/test_input.json`.
- **Documentation coûts** : voir `PRICING_REFERENCES.md` et la [liste officielle des GPU RunPod](https://runpod.io/pricing#serverless) pour choisir le profil adapté à votre budget.
- **Tests locaux** : exécuter `docker run ...` en local avec les variables d'environnement clés pour valider démarrage + healthcheck.

---

## 3. Problèmes récurrents à surveiller

1. **Jetons Hugging Face absents** : impossible de télécharger les poids privés -> l'init échoue ou boucle.
2. **Timeouts par défaut trop courts** : RunPod coupe le worker si `RUNPOD_INIT_TIMEOUT` ou `executionTimeout` sont insuffisants.
3. **Images surdimensionnées** : >7 GB = cold starts très longs et coûts réseau élevés.
4. **Modèles téléchargés à chaque démarrage** : sans intégrer les poids dans l'image Docker, chaque cold start re-télécharge le modèle (NLLB = ~2.5 GB), causant des latences de 60-120s.
5. **Secrets non injectés** : différence entre votre `.env` local et l'environnement côté RunPod.
6. **Healthcheck/heartbeat** non implémenté : RunPod pense que le container est down.
7. **Ping storm** : sans `idleTimeout` court, un endpoint reçoit des pings fréquents et consomme du GPU inutilement.

Nous détaillons pour chaque approche comment éviter ces points.

---

## 4. Option A — Déploiement 100% manuel depuis la console RunPod

> Idéal pour tester rapidement un prototype sans pipeline.

1. **Créer un compte RunPod** et récupérer votre clé API (Account > API Keys).
2. **Accéder au dashboard** > Serverless > Deploy Endpoint.
3. **Choisir bring your own container** :
   - Image : `docker.io/sawalle/translation-service:latest` (ou votre tag personnalisé).
   - Entrypoint et CMD : laissez ceux du Dockerfile.
   - GPU : sélectionner NVIDIA RTX A6000 ou NVIDIA L40 (bon équilibre coût/perf pour la traduction NLLB).
4. **Configurer l'onglet environment** :
   - Ajouter `HF_TOKEN`, `MODEL_PATH`, `LANGUAGE_CODE`, etc.
   - Définir `RUNPOD_INIT_TIMEOUT=800` et `RUNPOD_STOP_TIMEOUT=10`.
5. **Définir les timeouts/scaling** :
    - `idleTimeoutSeconds = 5` pour éviter les pings coûteux.
    - `minWorkers = 0`, `maxWorkers = 1` (budget serré BurkimbIA).
   - `executionTimeoutSeconds = 90` pour les requêtes de traduction volumineuses.
6. **Sauvegarder le template** puis cliquer sur `Deploy`.
7. **Tester via la console** :
   - `Serverless` > votre endpoint > `Run`.
   - Coller `services/translation/test_input.json`.
8. **Consulter les logs** (`Logs` tab) pour vérifier chargement du modèle, latence, éventuelles erreurs.

**Diagnostic manuel** :
- Erreur `ModuleNotFoundError` -> vérifier requirements Docker.
- Erreur 401 HF -> vérifier `HF_TOKEN` et les permissions HF (read access).
- Timeout -> augmenter `executionTimeoutSeconds` ou optimiser modèle.

---

## 5. Option B — Déploiement via un repository GitHub + RunPod Deploys

> Adapté si vous souhaitez que RunPod construise automatiquement votre image depuis GitHub et déclenche les mises à jour à chaque commit.

### 5.1 Créer votre repository étape par étape

> Hypothèse : vous partez de zéro et vous voulez une structure simple, reproductible, focalisée sur la traduction.

**Étape 1 — Initialiser le repository**

```bash
mkdir burkimbia-runpod-translation && cd burkimbia-runpod-translation
git init
```

**Étape 2 — Poser l'arborescence minimale**

```
.
├── deployment/
│   ├── deploy_runpod.py
│   └── runpod-config.json
├── services/
│   └── translation/
│       ├── Dockerfile
│       ├── src/
│       │   └── handler.py
│       ├── requirements.txt
│       └── test_input.json
├── scripts/
│   └── runpod_client.py
├── workflows/
│   └── build-and-push.yml
├── requirements.txt
└── README.md
```

- Le service de traduction possède son `Dockerfile`, un `src/handler.py` (RunPod handler) et un `test_input.json` prêt à l'emploi.
- Les éléments d'infrastructure (`deploy_runpod.py`, `runpod-config.json`) vivent dans `deployment/`.
- Les ressources CI/CD sont regroupées dans `workflows/` afin de pouvoir être copiées dans `.github/workflows/` le moment venu.

**Étape 3 — Fournir un payload de test**

Contenu recommandé pour `services/translation/test_input.json` :

```json
{
   "input": {
      "text": ["Bonjour"],
      "src_lang": "fra_Latn",
      "tgt_lang": "moo_Latn"
   }
}
```

**Étape 4 — Créer un Dockerfile optimisé**

Le secret pour éviter les cold starts longs (>60s) est d'**intégrer le modèle NLLB directement dans l'image Docker** plutôt que de le télécharger à chaque démarrage. Voici un exemple pour `services/translation/Dockerfile` basé sur le Dockerfile BurkimbIA en production :

```dockerfile
FROM pytorch/pytorch:2.7.1-cuda11.8-cudnn9-runtime

RUN apt-get update && apt-get install -y \
    build-essential \
    gcc \
    g++ \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install Python dependencies first (better caching)
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install -r requirements.txt

# Setup Hugging Face authentication using official CLI method
ARG HF_TOKEN
RUN huggingface-cli login --token ${HF_TOKEN}

# Download models using huggingface-cli (BEST PRACTICE for RunPod)
# This embeds the models in the Docker image for fastest loading
RUN huggingface-cli download burkimbia/nllb_600M_v0.0.2 --local-dir /app/models/BIA-NLLB-600M-5E && \
    # Clean up any potential .git folders to reduce image size
    find /app/models -name ".git" -type d -exec rm -rf {} + 2>/dev/null || true

# Copy application code
COPY src/ ./src/
COPY test_input.json ./test_input.json

# Create non-root user for security
RUN useradd --create-home --shell /bin/bash app && \
    chown -R app:app /app

USER app

# Set environment variables
ENV MODEL_PATH="/app/models/BIA-NLLB-600M-5E"
ENV PYTHONPATH="/app"

CMD ["python", "-u", "src/handler.py"]
```

> **Gain de performance** : Avec `huggingface-cli download` pendant le build, le modèle NLLB (600M) est intégré dans l'image. Cold start passe de 60-120s à 5-10s !

**Étape 5 — Créer un handler.py avec chargement local**

Contenu recommandé pour `services/translation/src/handler.py` :

```python
import os
import logging
from typing import Dict, List, Any
from transformers import AutoTokenizer, AutoModelForSeq2SeqLM
import runpod

# Configuration des logs
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Variables globales pour le modèle (chargé une seule fois)
MODEL = None
TOKENIZER = None
MODEL_PATH = os.getenv("MODEL_PATH", "/app/models/BIA-NLLB-600M-5E")

def load_model():
    """Charge le modèle NLLB depuis le stockage local."""
    global MODEL, TOKENIZER
    if MODEL is None:
        logger.info(f"Loading NLLB model from {MODEL_PATH}")
        try:
            TOKENIZER = AutoTokenizer.from_pretrained(MODEL_PATH)
            MODEL = AutoModelForSeq2SeqLM.from_pretrained(MODEL_PATH)
            logger.info("Model loaded successfully from local storage")
        except Exception as e:
            logger.error(f"Failed to load model: {str(e)}")
            raise
    return MODEL, TOKENIZER

def translate_text(texts: List[str], src_lang: str, tgt_lang: str) -> List[str]:
    """Traduit une liste de textes."""
    model, tokenizer = load_model()
    
    # Préparation des entrées
    tokenizer.src_lang = src_lang
    inputs = tokenizer(texts, return_tensors="pt", padding=True, truncation=True)
    
    # Génération
    generated_tokens = model.generate(
        **inputs,
        forced_bos_token_id=tokenizer.lang_code_to_id[tgt_lang],
        max_length=512,
        num_beams=5,
        early_stopping=True
    )
    
    # Décodage
    translations = tokenizer.batch_decode(generated_tokens, skip_special_tokens=True)
    return translations

def handler(event: Dict[str, Any]) -> Dict[str, Any]:
    """Handler RunPod pour la traduction."""
    try:
        input_data = event.get("input", {})
        texts = input_data.get("text", [])
        src_lang = input_data.get("src_lang", "fra_Latn")
        tgt_lang = input_data.get("tgt_lang", "moo_Latn")
        
        if not texts:
            return {"error": "No text provided"}
        
        logger.info(f"Translating {len(texts)} texts: {src_lang} -> {tgt_lang}")
        translations = translate_text(texts, src_lang, tgt_lang)
        
        return {
            "translations": translations,
            "source_language": src_lang,
            "target_language": tgt_lang,
            "model_path": MODEL_PATH
        }
        
    except Exception as e:
        logger.error(f"Translation failed: {str(e)}")
        return {"error": str(e)}

if __name__ == "__main__":
    # Préchargement du modèle au démarrage
    load_model()
    logger.info("Translation service ready")
    
    # Démarrer le serveur RunPod
    runpod.serverless.start({"handler": handler})
```

**Étape 6 — Fichier requirements.txt pour le service**

Contenu pour `services/translation/requirements.txt` :

```
torch>=2.0.0
transformers>=4.30.0
sentencepiece>=0.1.99
runpod>=1.6.0
huggingface-hub>=0.17.0
```

**Étape 7 — Définir la configuration RunPod**

Placez le fichier suivant dans `deployment/runpod-config.json`.

#### 5.1.1 Contenu de `deployment/runpod-config.json`

```json
{
   "translation_endpoint": {
      "endpoint_name": "burkimbia-translation-service",
      "description": "Service de traduction fra <-> mooré",
      "image": "docker.io/YOUR_DOCKER_USERNAME/translation-service:latest",
      "gpu_types": ["NVIDIA RTX A4000", "NVIDIA RTX A6000"],
      "idle_timeout": 5,
      "execution_timeout": 90,
      "scaling": {
         "min_workers": 0,
         "max_workers": 1
      },
      "env": [
         {"key": "SERVICE_NAME", "value": "translation"},
         {"key": "RUNPOD_INIT_TIMEOUT", "value": "800"},
         {"key": "HF_TOKEN", "value": "${HF_TOKEN}"},
         {"key": "MODEL_PATH", "value": "/app/models/BIA-NLLB-600M-5E"}
      ],
      "test_payload": "services/translation/test_input.json"
   }
}
```

#### 5.1.2 Contenu de `deployment/deploy_runpod.py`

```python
import argparse
import json
import pathlib
import sys
from typing import Any, Dict

import requests


RUNPOD_API_BASE = "https://api.runpod.ai/v2"


def load_config(path: pathlib.Path) -> Dict[str, Any]:
   try:
      return json.loads(path.read_text(encoding="utf-8"))
   except FileNotFoundError as exc:
      raise SystemExit(f"Config file not found: {path}") from exc


def build_headers(api_key: str) -> Dict[str, str]:
   return {
      "Authorization": api_key,
      "Content-Type": "application/json"
   }


def upsert_template(api_key: str, payload: Dict[str, Any]) -> str:
   response = requests.post(
      f"{RUNPOD_API_BASE}/endpoints",
      headers=build_headers(api_key),
      json=payload,
      timeout=30
   )
   response.raise_for_status()
   data = response.json()
   template_id = data.get("id") or data.get("endpointId")
   if not template_id:
      raise SystemExit(f"Unexpected response: {data}")
   return template_id


def deploy_endpoint(api_key: str, template_id: str) -> Dict[str, Any]:
   response = requests.post(
      f"{RUNPOD_API_BASE}/{template_id}/deploy",
      headers=build_headers(api_key),
      json={},
      timeout=30
   )
   response.raise_for_status()
   return response.json()


def load_payload(raw_config: Dict[str, Any], service: str) -> Dict[str, Any]:
   block = raw_config.get(service)
   if not block:
      raise SystemExit(f"Service '{service}' not found in config")
   payload = {
      "name": block["endpoint_name"],
      "description": block.get("description", ""),
      "imageName": block["image"],
      "gpuTypes": block.get("gpu_types", []),
      "idleTimeout": block.get("idle_timeout", 5),
      "restartPolicy": "OnFailure",
      "scaleConfig": {
         "min": block["scaling"]["min_workers"],
         "max": block["scaling"]["max_workers"],
         "batchSize": 1
      },
      "env": block.get("env", []),
      "workerConfig": {
         "timeout": block.get("execution_timeout", 60)
      }
   }
   return payload


def main() -> None:
   parser = argparse.ArgumentParser(description="Deploy RunPod endpoints from config")
   parser.add_argument("--config", default="runpod-config.json", help="Chemin fichier config")
   parser.add_argument("--service", required=True, help="Identifiant du service à déployer")
   parser.add_argument("--api-key", default=None, help="RunPod API key (sinon RUNPOD_API_KEY)")
   parser.add_argument("--dry-run", action="store_true", help="Afficher la payload sans appeler RunPod")
   args = parser.parse_args()

   api_key = args.api_key or pathlib.os.getenv("RUNPOD_API_KEY")
   if not api_key:
      raise SystemExit("RunPod API key missing. Set RUNPOD_API_KEY or use --api-key")

   raw_config = load_config(pathlib.Path(args.config))
   payload = load_payload(raw_config, args.service)

   if args.dry_run:
      print(json.dumps(payload, indent=2))
      sys.exit(0)

   template_id = upsert_template(api_key, payload)
   result = deploy_endpoint(api_key, template_id)
   print(f"Endpoint deployed: {result}")


if __name__ == "__main__":
   main()
```

#### 5.1.3 Contenu de `scripts/runpod_client.py`

```python
import time
from typing import Any, Dict, Optional

import requests


class RunPodClient:
   def __init__(self, api_token: str, base_url: str = "https://api.runpod.ai/v2") -> None:
      self.api_token = api_token
      self.base_url = base_url

   def _headers(self) -> Dict[str, str]:
      return {
         "Authorization": self.api_token,
         "Content-Type": "application/json"
      }

   def submit(self, endpoint_id: str, payload: Dict[str, Any]) -> str:
      response = requests.post(
         f"{self.base_url}/{endpoint_id}/run",
         headers=self._headers(),
         json=payload,
         timeout=30
      )
      response.raise_for_status()
      data = response.json()
      job_id = data.get("id") or data.get("jobId")
      if not job_id:
         raise RuntimeError(f"Invalid submit response: {data}")
      return job_id

   def poll(self, endpoint_id: str, job_id: str, timeout_s: int = 120, interval_s: int = 5) -> Optional[Dict[str, Any]]:
      deadline = time.time() + timeout_s
      while time.time() < deadline:
         response = requests.get(
            f"{self.base_url}/{endpoint_id}/status/{job_id}",
            headers=self._headers(),
            timeout=15
         )
         response.raise_for_status()
         data = response.json()
         status = data.get("status")
         if status in {"COMPLETED", "FAILED", "CANCELLED"}:
            return data
         time.sleep(interval_s)
      return None

   def run_and_wait(self, endpoint_id: str, payload: Dict[str, Any], timeout_s: int = 120) -> Dict[str, Any]:
      job_id = self.submit(endpoint_id, payload)
      result = self.poll(endpoint_id, job_id, timeout_s=timeout_s)
      if result is None:
         raise TimeoutError(f"Job {job_id} timed out")
      return result
```

#### 5.1.4 Contenu de `requirements.txt`

```
requests>=2.32.0
```

> Ajoutez les dépendances translation (transformers, sentencepiece, sacrebleu, etc.) directement dans le Dockerfile pour garder ce fichier léger.

Copiez ensuite `workflows/build-and-push.yml` vers `.github/workflows/build-and-push.yml` et utilisez ce contenu :

```yaml
name: Build and push
on:
   workflow_dispatch:
   push:
      branches: [ main ]

jobs:
   build:
      runs-on: ubuntu-latest
      steps:
         - uses: actions/checkout@v4
         - name: Log in to Docker Hub
            run: echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
            env:
               DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
               DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
         - name: Build image
            run: |
               docker build \
                  --build-arg HF_TOKEN=${{ secrets.HF_TOKEN }} \
                  -t ${{ secrets.DOCKER_USERNAME }}/translation-service:latest \
                  services/translation/
         - name: Push image
            run: docker push ${{ secrets.DOCKER_USERNAME }}/translation-service:latest
```

> **Important** : Passer le `HF_TOKEN` comme build argument permet de télécharger le modèle pendant la construction de l'image, éliminant ainsi les cold starts longs.

4. Déclarer les secrets GitHub suivants : `DOCKER_USERNAME`, `DOCKER_PASSWORD`, `HF_TOKEN`, `RUNPOD_API_KEY`.

### 5.2 Connecter RunPod à GitHub

1. Dans RunPod, cliquer sur `Deploy` > `Deploy from GitHub`.
2. Autoriser RunPod à accéder à votre organisation/repo.
3. Sélectionner le repository, la branche et le dossier contenant le Dockerfile.
4. Configurer le type de GPU, les variables d'environnement, timeouts et scaling directement dans l'assistant.
5. Activer `Autodeploy on push` si souhaité.

### 5.3 Problèmes courants

- RunPod ne voit pas votre Dockerfile -> vérifier `Context path`.
- Build échoue -> taille de l'image ou dépendances (utiliser base `runpod/base:...` si besoin).
- Secrets Docker manquants -> configurer `Build secrets` dans RunPod.

---

## 6. Option C — Infrastructure as Code via API RunPod

> Approche privilégiée par BurkimbIA pour industrialiser les déploiements et documenter la configuration.

### 6.1 Structure des fichiers

- `runpod-config.json` : référence centrale décrivant l'endpoint de traduction.
- `deploy_runpod.py` : script Python qui lit la config et appelle l'API RunPod.
- `scripts/runpod_client.py` : client Python optionnel pour consommer l'endpoint déployé.

### 6.2 Variables d'environnement

```bash
export RUNPOD_API_KEY="sk_live_xxxxxx"
export HF_TOKEN="hf_xxxxxx"
export RUNPOD_ACCOUNT_ID="your_account"   # optionnel si multi-org
```

### 6.3 Lancer le déploiement

```bash
python deployment/deploy_runpod.py \
   --service translation_endpoint \
   --config deployment/runpod-config.json
```

Options utiles :
- `--dry-run` : affiche la payload envoyée à RunPod sans exécuter la requête.
- `--api-key` : passer la clé directement en argument (sinon variable d'environnement `RUNPOD_API_KEY`).

### 6.4 Exemple d'appel API brut

Pour créer un template via `curl` :

```bash
curl -X POST https://api.runpod.ai/v2/endpoints \
   -H "Authorization: ${RUNPOD_API_KEY}" \
   -H "Content-Type: application/json" \
   -d @payload.json
```

`payload.json` peut être généré depuis `runpod-config.json`. Exemple minimal :

```json
{
   "name": "burkimbia-translation-service",
   "imageName": "docker.io/sawalle/translation-service:latest",
   "gpuTypes": ["NVIDIA RTX A4000"],
   "minWorkers": 0,
   "maxWorkers": 1,
   "idleTimeout": 5,
   "env": [
      {"key": "HF_TOKEN", "value": "${HF_TOKEN}"},
      {"key": "RUNPOD_INIT_TIMEOUT", "value": "800"}
   ],
   "ports": [],
   "volumeMounts": []
}
```

### 6.5 Avantages de l'IaC

- Historique Git complet des changements d'infrastructure.
- Reproductibilité (staging vs production).
- Possibilité d'intégrer des tests (ex. déclencher une requête de santé après déploiement).
- Compatible avec d'autres orchestrateurs (Terraform via provider HTTP, Pulumi via Python).

---

## 7. Diagnostic et support

### 7.1 Logs et métriques

- RunPod Console > Endpoint > `Logs` pour les journaux temps réel.
- `Metrics` fournit le nombre de requêtes, latence, cold starts.
- Ajouter des logs structurés JSON côté application pour tracer le coût par requête (cf. `monitoring.py`).

### 7.2 Tests d'inférence

Si vous n'utilisez pas `RunPodClient`, voici un `curl` générique :

```bash
curl -X POST https://api.runpod.ai/v2/<ENDPOINT_ID>/run \
   -H "Authorization: ${RUNPOD_API_KEY}" \
   -H "Content-Type: application/json" \
   -d @services/translation/test_input.json
```

Réponse : un `jobId` à poller.

```bash
curl -X GET https://api.runpod.ai/v2/<ENDPOINT_ID>/status/<JOB_ID> \
   -H "Authorization: ${RUNPOD_API_KEY}"
```

### 7.3 Erreurs typiques et remèdes

| Symptom | Cause probable | Correction |
|---------|----------------|-----------|
| 404 template | Mauvais endpoint ID | Vérifier `runpod_deployed_endpoints.json` |
| Status `FAILED_JOB` | Exception Python | Lire logs, vérifier dépendances |
| Load > 90s | Image trop lourde | Utiliser base + quantification 4-bit |
| Load > 90s (modèle) | Téléchargement NLLB à chaque start | Intégrer modèle dans Dockerfile avec ARG HF_TOKEN |
| GPU indisponible | Rupture de stock RunPod | Ajouter fallback `gpuTypes` (A4000, A6000) |

---

## 8. Monitoring, coûts et gouvernance

- Coûts réels BurkimbIA : voir `PRICING_REFERENCES.md` (A6000 serverless = $8.67/mois pour 3000 requêtes de traduction fra->mooré).
- Activer les alertes budgétaires RunPod (`Settings` > `Billing alerts`).
- Exporter les logs vers un stockage externe pour audit (S3/Backblaze).
- Mettre en place un `cron` qui arrête les endpoints inactifs (`/endpoint/<id>/pause`).

---

## 9. Roadmap d'amélioration continue

1. **Automatiser les tests d'acceptation** : script qui déploie, envoie une requête, vérifie la qualité de traduction.
2. **Ajout Terraform/Pulumi** : générer les payloads RunPod depuis IaC mainstream.
3. **Gestion multi-modèles** : templating Jinja dans `runpod-config.json` pour décliner les GPU selon la paire de langues.
4. **Mécanisme de rollback** : conserver les `templateId` précédents pour repasser à une version stable.

---

## 10. Checklist rapide (à coller dans vos PR)

- [ ] Docker image construite localement et scannée (`docker scan`).
- [ ] Variables d'environnement listées et documentées.
- [ ] `runpod-config.json` mis à jour et validé avec `jsonschema`.
- [ ] Tests d'inférence exécutés (curl + script Python).
- [ ] Cost impact évalué (GPU, temps moyen, budget BurkimbIA).
- [ ] Documentation interne mise à jour (`RUNPOD_DEPLOYMENT_GUIDE.md`, `PRICING_REFERENCES.md`).

---

**Avec ces trois options (console, GitHub, API), vous pouvez démarrer rapidement et évoluer vers une approche IaC robuste en fonction de votre maturité et de vos contraintes.**