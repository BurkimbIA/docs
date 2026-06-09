# RunPod MCP launcher pour entrainement LLM

Ce guide documente notre chemin pratique pour piloter RunPod depuis Codex avec MCP, creer un pod GPU, preparer `bia-llms`, lancer un smoke training, puis observer les logs. Il complete le guide manuel [runpod pods setup](runpod_pods_setup.md).

Le cas concret ici est le smoke du modele `Qwen/Qwen3-1.7B` sur le dataset prive `burkimbia/moore-instruct-v1`.
La branche projet pour `moore-llm-sota` est toujours `main`, et le workflow training la checkout explicitement.

## Objectif

- Creer un pod training depuis Codex via le MCP RunPod.
- Utiliser VS Code Remote-SSH ou `code-server` pour inspecter le pod.
- Installer `bia-llms` et `moore-llm-sota`.
- Lancer un smoke run court avant tout entrainement complet.
- Rendre visibles W&B, epochs, checkpoints et S3 dans le template.
- Garder des logs exploitables dans `/workspace`.
- Stopper ou supprimer les pods instables pour eviter les couts inutiles.

## Prerequis

Variables locales cote machine Codex:

```powershell
$env:RUNPOD_API_KEY = "<runpod_api_key>"
$env:GITHUB_ACCESS_TOKEN = "<github_token_repo_private>"
$env:HF_TOKEN = "<huggingface_token_dataset_prive>"
$env:WANDB_API_KEY = "<wandb_key>"
```

Dans l'interface RunPod, il est preferable d'enregistrer les secrets sous ces noms:

```bash
GITHUB_ACCESS_TOKEN
HF_TOKEN
WANDB_API_KEY
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_DEFAULT_REGION
AWS_ENDPOINT_URL_S3
S3_CHECKPOINT_URI
S3_MODEL_URI
S3_LOG_URI
```

Important: les secrets enregistres dans l'interface RunPod ne sont pas toujours lisibles directement par MCP. Ils sont utilisables quand ils sont injectes comme variables d'environnement dans un pod ou un template. Ne pas supposer que MCP peut les relire.

## Configuration MCP Codex

Ajouter les MCP:

```powershell
codex mcp add runpod-docs --url https://docs.runpod.io/mcp
codex mcp add runpod --env "RUNPOD_API_KEY=$env:RUNPOD_API_KEY" -- npx -y @runpod/mcp-server@latest
```

Verifier:

```powershell
codex mcp list
codex mcp get runpod
```

Si RunPod renvoie `401`, verifier que la config ne contient pas le litteral casse:

```text
RUNPOD_API_KEY = ":RUNPOD_API_KEY"
```

Si c'est le cas:

```powershell
codex mcp remove runpod
codex mcp add runpod --env "RUNPOD_API_KEY=$env:RUNPOD_API_KEY" -- npx -y @runpod/mcp-server@latest
```

Redemarrer Codex apres modification MCP. Les serveurs MCP sont charges au demarrage de la session.

## Creation du pod par MCP

Pour un smoke `Qwen3-1.7B`, demander au MCP un GPU 24 GB minimum. Les meilleurs choix pratiques:

- Debug: local/CPU d'abord, puis GPU 16-24 GB seulement si CUDA doit etre teste. Chercher A4000, A4500, RTX 4000 Ada, RTX 2000 Ada, A5000, L4 ou 3090 selon le prix live.
- `NVIDIA RTX A5000` ou `NVIDIA GeForce RTX 4090` pour smoke 1.7B apres validation locale.
- `NVIDIA L40`, `RTX A6000`, `RTX 6000 Ada` pour plus de marge et eventuellement un run 4B.

Ne pas utiliser A6000/L40/48 GB pour setup, docs, template edits ou debug scripts. Ces GPUs sont reserves aux full runs approuves ou aux smoke finaux.

Exemple de specification:

```text
templateId: bgqn0opi39
templateName: moore-llm-sota-training-template
cloudType: COMMUNITY ou SECURE selon stock
imageName: runpod/pytorch:2.8.0-py3.11-cuda12.8.1-cudnn-devel-ubuntu22.04
gpuCount: 1
containerDiskInGb: 50
volumeInGb: 50
volumeMountPath: /workspace
ports:
  - 8888/http
  - 8080/http
  - 22/tcp
env:
  RUNPOD_INIT_TIMEOUT: "600"
  HF_HOME: /workspace/.cache/huggingface
  TRANSFORMERS_CACHE: /workspace/.cache/huggingface
  XDG_CACHE_HOME: /workspace/.cache
  UV_CACHE_DIR: /workspace/.cache/uv
  UV_PROJECT_ENVIRONMENT: /workspace/.venv/bia-llms
  UV_LINK_MODE: copy
  WANDB_PROJECT: moore-llm-sota
  GITHUB_ACCESS_TOKEN: "{{ RUNPOD_SECRET_GITHUB_ACCESS_TOKEN }}"
  GITHUB_TOKEN: "{{ RUNPOD_SECRET_GITHUB_ACCESS_TOKEN }}"
  HF_TOKEN: "{{ RUNPOD_SECRET_HF_TOKEN }}"
  WANDB_API_KEY: "{{ RUNPOD_SECRET_WANDB_API_KEY }}"
  AWS_ACCESS_KEY_ID: "{{ RUNPOD_SECRET_AWS_ACCESS_KEY_ID }}"
  AWS_SECRET_ACCESS_KEY: "{{ RUNPOD_SECRET_AWS_SECRET_ACCESS_KEY }}"
  AWS_DEFAULT_REGION: "auto"
  AWS_REGION: "auto"
  AWS_ENDPOINT_URL_S3: "https://fly.storage.tigris.dev"
  S3_CHECKPOINT_URI: "s3://burkimbia-store/moore-llm-sota/checkpoints"
  S3_MODEL_URI: "s3://burkimbia-store/moore-llm-sota/models"
  S3_LOG_URI: "s3://burkimbia-store/moore-llm-sota/logs"
  RUNPOD_S3_SYNC_INTERVAL_SECONDS: "300"
  RUNPOD_AUDIT_INTERVAL_SECONDS: "60"
  RUNPOD_SKIP_UV_SYNC: "false"
  REQUIRE_S3_SYNC: "true"
  BIA_LLMS_BRANCH: "moore-multitask-xml"
  BIA_LLMS_FALLBACK_BRANCH: "main"
  MOORE_BRANCH: "main"
```

Pour un smoke non officiel sans S3, mettre temporairement `REQUIRE_S3_SYNC=false`. Pour un full run, garder `REQUIRE_S3_SYNC=true`.

Le premier `uv sync --extra cu128 --all-groups` sur volume neuf peut prendre plusieurs minutes, meme avec un petit GPU, parce qu'il telecharge PyTorch, CUDA, xFormers, bitsandbytes et Unsloth. Garder les caches sous `/workspace/.cache` et l'environnement sous `/workspace/.venv/bia-llms`. Si une image ou un volume contient deja l'environnement synchronise, mettre `RUNPOD_SKIP_UV_SYNC=true`; sinon le premier prepare est la phase lente a surveiller et a stopper si elle bloque.

Le `stockStatus=Low` ne garantit pas que la creation passera. RunPod peut retourner:

```text
create pod: This machine does not have the resources to deploy your pod
create pod: There are no instances currently available
```

Dans ce cas, changer de GPU ou de cloud type.

## Connexion VS Code ou code-server

VS Code Remote-SSH est le choix le plus propre pour developper et lancer les commandes longues. Recuperer l'IP et le port TCP exposes, puis:

```bash
ssh root@<PUBLIC_IP> -p <TCP_PORT> -i ~/.ssh/id_ed25519
```

Dans VS Code:

1. Installer `Remote - SSH`.
2. `Remote-SSH: Connect to Host`.
3. Ouvrir `/workspace`.

`code-server` est utile comme fallback navigateur. Exposer `8080/http`, puis dans le pod:

```bash
curl -fsSL https://code-server.dev/install.sh | sh
nohup code-server --bind-addr 0.0.0.0:8080 --auth none /workspace > /workspace/code-server.log 2>&1 &
```

URL:

```text
https://<POD_ID>-8080.proxy.runpod.net
```

## Preparation training

Sur un volume vide, le chemin recommande pour un agent est de pousser le bootstrap versionne depuis un checkout local:

```powershell
scp -P <TCP_PORT> scripts\runpod_agent_bootstrap.sh root@<PUBLIC_IP>:/tmp/runpod_agent_bootstrap.sh
ssh -p <TCP_PORT> root@<PUBLIC_IP> "sed -i 's/\r$//' /tmp/runpod_agent_bootstrap.sh && bash /tmp/runpod_agent_bootstrap.sh prepare-bg"
ssh -p <TCP_PORT> root@<PUBLIC_IP> 'bash /workspace/moore-llm-sota/scripts/runpod_agent.sh status'
```

Sous Windows PowerShell, ne pas envoyer directement `Get-Content -Raw` vers
Bash: les fins de ligne CRLF peuvent interrompre le script avant le bootstrap.

Le bootstrap recupere les variables RunPod depuis `/proc/1/environ`, clone `moore-llm-sota` branche `main`, puis delegue a `scripts/runpod_agent.sh`.

Depuis le pod, quand le repo existe:

```bash
cd /workspace
git clone --branch main https://github.com/BurkimbIA/moore-llm-sota.git
cd moore-llm-sota
```

Si le repo est prive et que le clone interactif echoue, utiliser `GITHUB_ACCESS_TOKEN` ou envoyer une archive depuis la machine locale. Le script projet accepte `GITHUB_ACCESS_TOKEN` et `GITHUB_TOKEN`.

Preparation:

```bash
export GITHUB_TOKEN="${GITHUB_ACCESS_TOKEN:-$GITHUB_TOKEN}"
export HF_TOKEN="<hf_token>"
bash /workspace/moore-llm-sota/scripts/runpod_agent.sh prepare-bg
bash /workspace/moore-llm-sota/scripts/runpod_agent.sh status
```

Logs recommandés:

```bash
nohup bash /workspace/moore-llm-sota/scripts/runpod_prepare.sh > /workspace/runpod_prepare.log 2>&1 &
tail -f /workspace/runpod_prepare.log
```

Signaux attendus dans le log:

```text
Git LFS initialized.
Cloning into '/workspace/bia-llms'...
bia-llms==0.23.0
torch==2.8.0+cu128
unsloth==...
HF login OK: <user>
RunPod setup complete
```

## Smoke training

Ne pas lancer directement un full run. Commencer par:

```bash
bash /workspace/moore-llm-sota/scripts/runpod_agent.sh doctor
bash /workspace/moore-llm-sota/scripts/runpod_agent.sh smoke-1.7b
bash /workspace/moore-llm-sota/scripts/runpod_agent.sh follow
```

Le smoke doit prouver:

- chargement du dataset prive `burkimbia/moore-instruct-v1`;
- chargement du modele `Qwen/Qwen3-1.7B`;
- detection CUDA;
- demarrage SFT/QLoRA;
- cap exact a `max_steps=30`;
- W&B actif dans le projet `moore-llm-sota`;
- pas de repetition massive des tags XML.

## Full training

Le chemin officiel utilise le launcher du repo, pas des overrides `--cli_config.*`.

```bash
bash /workspace/moore-llm-sota/scripts/runpod_agent.sh full-1.7b
bash /workspace/moore-llm-sota/scripts/runpod_agent.sh status
```

Puis, seulement si le 1.7B est bon qualitativement:

```bash
bash /workspace/moore-llm-sota/scripts/runpod_agent.sh full-4b
```

Politique actuelle:

| Run | Modele | Longueur | Checkpoints | W&B | S3 |
| --- | --- | --- | --- | --- | --- |
| `smoke-1.7b` | `Qwen/Qwen3-1.7B` | `max_steps=30` | non | oui | sync finale si configure |
| `full-1.7b` | `Qwen/Qwen3-1.7B` | `2 epochs` | toutes les `1000` steps, garder `3` | oui | sync periodique + finale |
| `full-4b` | `Qwen/Qwen3-4B` | `2 epochs` | toutes les `1000` steps, garder `3` | oui | sync periodique + finale |

Le launcher imprime cette politique au debut du log. Pour un full run, on doit voir `num_epochs=2`, `save_steps=1000`, `wandb_enabled=True`, puis les URI S3 configurees.

Chaque run cree aussi un dossier d'audit autonome:

```text
/workspace/audit/moore-llm-sota/<run_id>/
  run.log
  progress.log
  manifest.txt
  resolved_config.yaml
  train_command.txt
  python_packages.txt
  status.txt
```

`run.log` capture les sorties meme si la commande externe n'a pas redirige stdout. `manifest.txt` contient le run kind, les commits Git, le GPU, l'environnement masque et le chemin de la config resolue. `progress.log` ajoute un snapshot GPU/disque/checkpoints/process toutes les 60 secondes.

## Observation des logs

Commandes utiles:

```bash
nvidia-smi
tail -120 /workspace/runpod_prepare.log
bash /workspace/moore-llm-sota/scripts/runpod_agent.sh logs
bash /workspace/moore-llm-sota/scripts/runpod_agent.sh progress
ps -ef | grep -E 'train.py|uv|python' | grep -v grep
du -sh /workspace/*
ls -lah /workspace/audit/moore-llm-sota/latest
tail -120 /workspace/audit/moore-llm-sota/latest/run.log
tail -80 /workspace/audit/moore-llm-sota/latest/progress.log
cat /workspace/audit/moore-llm-sota/latest/manifest.txt
```

Pour verifier S3 sans afficher de secret:

```bash
grep -E 'S3 sync|resolved run_kind|wandb_enabled|save_steps|num_epochs' /workspace/train-full-1.7b.log
```

Si SSH tombe mais MCP voit encore le pod `RUNNING`, verifier la fiche machine. Une note comme celle-ci indique une machine instable:

```text
This server has recently suffered a network outage and may have spotty network connectivity.
```

Dans ce cas, stopper le pod puis recreer ailleurs. Ne pas continuer a debuguer longuement une machine marquee en outage.

## Retour d'experience du 29 mai 2026

Essai effectue:

- Pod MCP cree: `moore-qwen3-1-7b-smoke`
- GPU: `NVIDIA L40`, 48 GB VRAM
- Cout: `$0.69/h`
- Pod id: `pdkrf6lwy1opc5`
- SSH teste OK au debut: `NVIDIA L40, 46068 MiB`
- `code-server` installe et lance sur `8080`
- `uv sync --extra cu128 --all-groups` a termine et installe `bia-llms==0.23.0`, `torch==2.8.0+cu128`, `unsloth`
- Blocage initial: `huggingface-cli` absent dans l'environnement recent. Correction: ecrire le token HF dans `HF_HOME/token` et verifier via `huggingface_hub.whoami()`.
- Blocage GitHub: utiliser `GITHUB_ACCESS_TOKEN` et pas seulement `GITHUB_TOKEN`.
- Incident final: la machine L40 a ete marquee par RunPod comme ayant une panne reseau, puis SSH a expire. Le pod a ete stoppe pour arreter le cout GPU.

Conclusion: le flux MCP fonctionne, mais il faut automatiser les fallbacks GPU et traiter les machines avec note d'outage comme non fiables.

## Regles de securite et couts

- Ne jamais commiter les tokens.
- Ne pas mettre de tokens bruts dans les templates; utiliser `{{ RUNPOD_SECRET_* }}`.
- Preferer `GITHUB_ACCESS_TOKEN`, `HF_TOKEN`, `WANDB_API_KEY` comme noms standards.
- Pour les full runs, W&B doit rester actif.
- Pour les full runs, S3 doit etre configure avec `REQUIRE_S3_SYNC=true`.
- Toujours observer `runpod_prepare.log` avant de lancer le training.
- Stopper un pod instable immediatement.
- Terminer un pod inutile si on n'a plus besoin du volume.
