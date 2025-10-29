# runpod serverless llm tutoriel

Ce tutoriel detaille la sequence pour livrer un service `llm` sur `runpod serverless`. Le but est d'obtenir un endpoint resilient et peu couteux pour l'association `burkimbia`.

## prerequis

- une image docker basee sur `nvidia cuda` contenant votre modele quantifie.
- un compte `runpod` avec une cle `RUNPOD_API_KEY`.
- un compte `hugging face` si le modele est prive et necessite `HF_TOKEN`.
- `python 3.10` ou plus avec les dependances du script `deploy_runpod.py`.

Stockez les secrets dans votre shell avant de commencer:

```bash
export RUNPOD_API_KEY="votre_cle"
export HF_TOKEN="votre_token"
export DOCKER_USERNAME="votre_identifiant"
```

## etape 1 construire une image docker legere

1. partez d'une base `nvidia/cuda:12.1.1-runtime-ubuntu22.04`.
2. installez `python`, `pip` et uniquement les bibliotheques necessaires (`torch`, `transformers`, `bitsandbytes`).
3. copiez le modele quantifie dans `/models` pour eviter tout telechargement a chaud.
4. definissez `RUNPOD_INIT_TIMEOUT=800` et `PYTHONUNBUFFERED=1` dans le `Dockerfile`.

Construisez et poussez l'image:

```bash
docker build -t docker.io/votre_repo/llm-service:latest .
docker push docker.io/votre_repo/llm-service:latest
```

## etape 2 parametrer `runpod-config.json`

Créez un fichier a la racine du projet:

```json
{
  "translation_endpoint": {
    "endpoint_name": "burkimbia-translation-service",
    "model_type": "translation",
    "image": "docker.io/votre_repo/llm-service:latest",
    "gpu_types": ["NVIDIA RTX A4000"],
    "idle_timeout": 5,
    "execution_timeout": 60,
    "scaling": {
      "min_workers": 0,
      "max_workers": 1
    },
    "env": {
      "HF_TOKEN": "${HF_TOKEN}",
      "MODEL_PATH": "/models/mistral-7b-int4"
    }
  }
}
```

Selectionnez le plus petit `gpu_types` possible. `execution_timeout` doit couvrir le temps de reponse maximal observe. Gardez `max_workers` a `1` tant que la charge reste modeste.

## etape 3 lancer le script deploiement local

Depuis la racine du depot:

```bash
python deploy_runpod.py --config runpod-config.json --force
```

Le script `RunPodDeployer` va:

- verifier la presence des secrets requis.
- creer ou recreer le template serverless.
- appliquer la configuration `idle_timeout` et `execution_timeout`.
- generer `runpod_deployed_endpoints.json` contenant l'identifiant du service.

Si vous preferez `github actions`, declenchez le workflow `DEPLOY - RunPod AI Services` et cochez l'option `Force recreate template`.

## etape 4 tester l'endpoint

Utilisez le client interne `common/runpod_client.py`.

```python
import os
from common.runpod_client import RunPodClient

client = RunPodClient(api_token=os.environ["RUNPOD_API_KEY"])
result = client.translate(
    text="Bonjour le monde",
    service_id="ENDPOINT_ID",
    src_lang="fra_Latn",
    tgt_lang="moor_Latn",
    model_type="nllb"
)
print(result)
```

Verifiez que `status` vaut `COMPLETED` et que le temps total reste sous votre seuil. Activez `verbose=True` pour inspecter le `cold_start` lors de la premiere requete.

## etape 5 surveiller et durcir

- surveillez les journaux `runpod` pour repérer des `ping` anormaux.
- configurez une alerte `slack` quand `gpu_seconds_total` depasse 80 pour cent du budget mensuel.
- fixez `job_ttl` a une valeur courte (par exemple `600` secondes) pour eviter la retention de jobs termines.
- mettez a jour l'image docker a chaque changement de modele, puis relancez le workflow.

## nettoyage

Quand un endpoint n'est plus necessaire:

```bash
python deploy_runpod.py --config runpod-config.json --delete translation_endpoint
```

Cela supprime l'endpoint et libere le worker. Conservez le `template` uniquement si vous prevez un redeploiement rapide.

En suivant ces etapes, vous obtenez un service `llm` serverless frugal et reproductible, adapte aux contraintes de `burkimbia`.
