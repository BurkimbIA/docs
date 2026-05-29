# RunPod MCP launcher pour entrainement LLM

Ce guide documente notre chemin pratique pour piloter RunPod depuis Codex avec MCP, creer un pod GPU, preparer `bia-llms`, lancer un smoke training, puis observer les logs. Il complete le guide manuel [runpod pods setup](runpod_pods_setup.md).

Le cas concret ici est le smoke du modele `Qwen/Qwen3-1.7B` sur le dataset prive `burkimbia/moore-instruct-v1`.

## Objectif

- Creer un pod training depuis Codex via le MCP RunPod.
- Utiliser VS Code Remote-SSH ou `code-server` pour inspecter le pod.
- Installer `bia-llms` et `moore-llm-sota`.
- Lancer un smoke run court avant tout entrainement complet.
- Garder des logs exploitables dans `/workspace`.
- Stopper ou supprimer les pods instables pour eviter les couts inutiles.

## Prerequis

Variables locales cote machine Codex:

```powershell
$env:RUNPOD_API_KEY = "<runpod_api_key>"
$env:GITHUB_ACCESS_TOKEN = "<github_token_repo_private>"
$env:HF_TOKEN = "<huggingface_token_dataset_prive>"
$env:WANDB_API_KEY = "<wandb_key_optional>"
```

Dans l'interface RunPod, il est preferable d'enregistrer les secrets sous ces noms:

```bash
GITHUB_ACCESS_TOKEN
HF_TOKEN
WANDB_API_KEY
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

- `NVIDIA RTX A5000` ou `NVIDIA GeForce RTX 4090` pour smoke 1.7B.
- `NVIDIA L40`, `RTX A6000`, `RTX 6000 Ada` pour plus de marge et eventuellement un run 4B.

Exemple de specification:

```text
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
  WANDB_PROJECT: moore-llm-sota
```

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

Depuis le pod:

```bash
cd /workspace
git clone https://github.com/BurkimbIA/moore-llm-sota.git
cd moore-llm-sota
```

Si le repo est prive et que le clone interactif echoue, utiliser `GITHUB_ACCESS_TOKEN` ou envoyer une archive depuis la machine locale. Le script projet accepte `GITHUB_ACCESS_TOKEN` et `GITHUB_TOKEN`.

Preparation:

```bash
export GITHUB_TOKEN="${GITHUB_ACCESS_TOKEN:-$GITHUB_TOKEN}"
export HF_TOKEN="<hf_token>"
export WANDB_API_KEY=""
bash /workspace/moore-llm-sota/scripts/runpod_prepare.sh
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
nohup bash /workspace/moore-llm-sota/scripts/runpod_train.sh smoke-1.7b > /workspace/train-smoke-1.7b.log 2>&1 &
tail -f /workspace/train-smoke-1.7b.log
```

Le smoke doit prouver:

- chargement du dataset prive `burkimbia/moore-instruct-v1`;
- chargement du modele `Qwen/Qwen3-1.7B`;
- detection CUDA;
- demarrage SFT/QLoRA;
- generation/evaluation aux steps prevus;
- pas de repetition massive des tags XML.

## Observation des logs

Commandes utiles:

```bash
nvidia-smi
tail -120 /workspace/runpod_prepare.log
tail -120 /workspace/train-smoke-1.7b.log
ps -ef | grep -E 'train.py|uv|python' | grep -v grep
du -sh /workspace/*
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
- Preferer `GITHUB_ACCESS_TOKEN`, `HF_TOKEN`, `WANDB_API_KEY` comme noms standards.
- Pour smoke, W&B peut etre desactive.
- Toujours observer `runpod_prepare.log` avant de lancer le training.
- Stopper un pod instable immediatement.
- Terminer un pod inutile si on n'a plus besoin du volume.
