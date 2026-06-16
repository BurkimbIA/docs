# Cookbook API BurkimbIA

Recettes pour appeler les API BurkimbIA (traduction, transcription, synthèse
vocale) en `curl`, en Python et en Java.

!!! warning "Mode serverless et cold start (à lire avant d'intégrer)"
    Les API tournent en **serverless / scale-to-zero** : les modèles s'éteignent
    quand ils sont inactifs et se rallument à la demande. Latence moyenne
    observée :

    - **Sans cold start** (modèle déjà chaud) : environ **1 à 3 s**.
    - **Avec cold start** (premier appel après une période d'inactivité) :
      environ **30 à 50 s**, le temps de réveiller le worker GPU et de charger le
      modèle en VRAM. Ça peut dépasser une minute en cas de forte pénurie de GPU.

    En pratique : prévoyez un timeout client généreux (≥ 60 s). Le premier appel
    d'une session est lent, les suivants sont rapides tant que le modèle reste
    chaud.

## Base et authentification

- **Base URL** : `https://api.burkimbia.com/api/v1`
- **Authentification** : en-tête `X-API-Key: <votre_clé>` (les clés commencent par `bia_`).
- **Obtenir une clé** : depuis la plateforme [Rɛɛm-doogo](https://platform.burkimbia.com) (section *API Tokens*).
- **Documentation interactive** : [Swagger](https://api.burkimbia.com/docs) ou [ReDoc](https://api.burkimbia.com/redoc).

!!! note "Codes de langue"
    Utilisez les mots `french` et `moore` (pas `fr` ni `mos`).

## Modèles disponibles

On appelle les modèles par leur **alias public** via le champ `model`. Les
identifiants internes ne sont pas exposés.

| Alias | Tâche | Note |
|-------|-------|------|
| `bia-translation-v1` | Traduction FR ↔ Mooré | par défaut, rapide |
| `bia-translation-v2` | Traduction FR ↔ Mooré | meilleure qualité, contextes complexes |
| `bia-transcription-v1` | Transcription audio → texte | |
| `bia-tts-v1` | Synthèse vocale texte → audio | |

## Traduction

```bash
curl -X POST https://api.burkimbia.com/api/v1/translate \
  -H "X-API-Key: $BIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Bonjour, comment vas-tu ?",
    "src_lang": "french",
    "tgt_lang": "moore",
    "model": "bia-translation-v1"
  }'
```

Réponse :

```json
{ "output": "Yibeoogo, laafi laafi??", "src_lang": "french", "tgt_lang": "moore" }
```

En Python :

```python
import os, requests

resp = requests.post(
    "https://api.burkimbia.com/api/v1/translate",
    headers={"X-API-Key": os.environ["BIA_API_KEY"]},
    json={
        "text": "Bonjour, comment vas-tu ?",
        "src_lang": "french",
        "tgt_lang": "moore",
        "model": "bia-translation-v1",
    },
    timeout=120,
)
print(resp.json()["output"])
```

En Java (`java.net.http`, JDK 17+) :

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

var body = """
    {"text":"Bonjour, comment vas-tu ?","src_lang":"french","tgt_lang":"moore","model":"bia-translation-v1"}""";
var req = HttpRequest.newBuilder(URI.create("https://api.burkimbia.com/api/v1/translate"))
    .header("X-API-Key", System.getenv("BIA_API_KEY"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(body))
    .build();
var resp = HttpClient.newHttpClient().send(req, HttpResponse.BodyHandlers.ofString());
System.out.println(resp.body());
```

!!! tip "Traduire une liste"
    `text` accepte aussi une liste (max 20 éléments, 256 caractères chacun) ;
    la réponse garde le même type (liste → liste).

## Transcription

Trois façons d'envoyer l'audio. Choisissez-en une.

**1. Fichier en multipart (le plus simple)** : route `/transcribe/file`.

```bash
curl -X POST https://api.burkimbia.com/api/v1/transcribe/file \
  -H "X-API-Key: $BIA_API_KEY" \
  -F "file=@audio.wav;type=audio/wav" \
  -F "language=mos" \
  -F "model=bia-transcription-v1"
```

**2. URL audio publique** : route `/transcribe`.

```bash
curl -X POST https://api.burkimbia.com/api/v1/transcribe \
  -H "X-API-Key: $BIA_API_KEY" -H "Content-Type: application/json" \
  -d '{"audio_url": "https://exemple.com/audio.wav", "language": "mos", "model": "bia-transcription-v1"}'
```

**3. Audio encodé en base64** : route `/transcribe`.

```bash
curl -X POST https://api.burkimbia.com/api/v1/transcribe \
  -H "X-API-Key: $BIA_API_KEY" -H "Content-Type: application/json" \
  -d "{\"audio_base64\": \"$(base64 -w0 audio.wav)\", \"language\": \"mos\", \"model\": \"bia-transcription-v1\"}"
```

Réponse :

```json
{ "text": "ninsaal pa tõe n yã n yɩɩg a yãab ye.", "chunks": [], "language": "mos", "model": "bia-transcription-v1" }
```

En Python (multipart) :

```python
import os, requests

with open("audio.wav", "rb") as f:
    resp = requests.post(
        "https://api.burkimbia.com/api/v1/transcribe/file",
        headers={"X-API-Key": os.environ["BIA_API_KEY"]},
        files={"file": ("audio.wav", f, "audio/wav")},
        data={"language": "mos", "model": "bia-transcription-v1"},
        timeout=200,
    )
print(resp.json()["text"])
```

En Java (méthode base64, sans gestion multipart) :

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Base64;

var b64 = Base64.getEncoder().encodeToString(Files.readAllBytes(Path.of("audio.wav")));
var body = "{\"audio_base64\":\"" + b64 + "\",\"language\":\"mos\",\"model\":\"bia-transcription-v1\"}";
var req = HttpRequest.newBuilder(URI.create("https://api.burkimbia.com/api/v1/transcribe"))
    .header("X-API-Key", System.getenv("BIA_API_KEY"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(body))
    .build();
var resp = HttpClient.newHttpClient().send(req, HttpResponse.BodyHandlers.ofString());
System.out.println(resp.body());
```

## Synthèse vocale (TTS)

```bash
curl -X POST https://api.burkimbia.com/api/v1/tts \
  -H "X-API-Key: $BIA_API_KEY" -H "Content-Type: application/json" \
  -d '{"text": "Ne y windga", "gender": "male", "model": "bia-tts-v1"}'
```

La réponse contient l'audio WAV encodé en base64 dans le champ `wav` :

```python
import os, base64, requests

resp = requests.post(
    "https://api.burkimbia.com/api/v1/tts",
    headers={"X-API-Key": os.environ["BIA_API_KEY"]},
    json={"text": "Ne y windga", "gender": "male", "model": "bia-tts-v1"},
    timeout=200,
)
with open("sortie.wav", "wb") as out:
    out.write(base64.b64decode(resp.json()["wav"]))
```

En Java :

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Base64;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

var body = """
    {"text":"Ne y windga","gender":"male","model":"bia-tts-v1"}""";
var req = HttpRequest.newBuilder(URI.create("https://api.burkimbia.com/api/v1/tts"))
    .header("X-API-Key", System.getenv("BIA_API_KEY"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(body))
    .build();
var resp = HttpClient.newHttpClient().send(req, HttpResponse.BodyHandlers.ofString());
// Extraire le champ "wav" (en production, préférez un parseur JSON comme Jackson)
Matcher m = Pattern.compile("\"wav\"\\s*:\\s*\"([^\"]+)\"").matcher(resp.body());
if (m.find()) {
    Files.write(Path.of("sortie.wav"), Base64.getDecoder().decode(m.group(1)));
}
```

## Bon à savoir

- **Cold start** : voir l'encadré en haut de page (latence du premier appel).
- **Quota** : un dépassement du quota journalier renvoie `429`.
- **Erreurs courantes** : `401` (clé absente ou invalide), `422` (entrée
  invalide, par exemple une langue autre que `french`/`moore` ou un `model`
  inconnu), `500`/timeout côté inférence.
