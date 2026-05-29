# Plan SOTA pour un LLM Moore

Ce document fixe la stratégie pour passer d'un systeme de traduction FR-Moore a un modele de langue Moore capable de traduire, corriger, expliquer, raisonner et dialoguer.

## Decision centrale

Le projet ne doit pas etre positionne comme un simple modele de traduction. Le bon objectif est un modele instruct Moore multi-taches:

- traduction bidirectionnelle FR <-> Moore;
- correction grammaticale et orthographique du Moore;
- detection de paires de mauvaise qualite;
- explication linguistique courte;
- terminologie et dictionnaire;
- comprehension, question-reponse, reformulation et dialogue.

NLLB reste utile, mais seulement comme baseline forte, teacher de traduction et outil de backtranslation. Le coeur du projet doit etre un LLM adapte au Moore.

## Etat des donnees locales

Mesures locales au 26 mai 2026:

| Fichier | Lignes utiles | Role |
|---|---:|---|
| `output/postprocessing/cleaned/cleaned_data_train.csv` | 123,587 | corpus propre principal FR -> Moore |
| `output/postprocessing/cleaned/cleaned_data_evaluation.csv` | 1,172 | evaluation MT thematique |
| `output/postprocessing/anomalies/anomalies_train.csv` | 2,347 | exemples negatifs pour controle qualite |
| `output/postprocessing/anomalies/cleaned_data_evaluation.csv` | 12 | exemples negatifs eval |

Schemas observes:

- train nettoye: `src`, `trg`, `source`, `source_lang`, `target_lang`;
- eval nettoyee: `id`, `src`, `reference_translation`, `source`, `source_lang`, `target_lang`, `theme`, `weight`;
- annotations Label Studio: `source_text`, `target_text`, `quality`, `modified`, `corrected_source`, `corrected_target`, `keep_annotation`, `annotator`, `created_at`.

Le pipeline a deja une base assez large pour un vrai SFT multi-taches. Le manque principal n'est plus seulement le volume brut, mais la transformation des donnees en exemples instructifs et evaluables.

## Sources a ponderer

Le corpus n'a pas une distribution neutre. Les sources religieuses sont volumineuses et utiles, mais elles ne doivent pas dominer le comportement general du modele.

Priorite haute:

- traductions expertes;
- glossaire numerique;
- agriculture et elevage;
- ministeres et textes administratifs;
- enquetes;
- SPG series;
- sante et noms de maladies;
- dictionnaire Moore-Francais.

Priorite controlee:

- Bible;
- Coran;
- corpus web ou semi-synthetiques;
- paires signalees par les regles d'anomalie.

Regle recommandee: ne jamais laisser Bible + Coran depasser 25-30% du SFT final, meme si ces sources sont propres.

## Dataset cible

Le dataset final doit etre publie comme un dataset instruct versionne, par exemple:

`burkimbia/moore-instruct-v1`

Schema recommande:

| Colonne | Description |
|---|---|
| `id` | identifiant stable |
| `task` | type de tache |
| `input_lang` | `fr`, `moore`, `mixed` |
| `output_lang` | `fr`, `moore`, `label`, `mixed` |
| `input` | prompt utilisateur ou texte source |
| `target` | reponse attendue |
| `source` | provenance |
| `domain` | religious, administrative, health, agriculture, tech, daily_life, dictionary, unknown |
| `quality_tier` | gold, silver, bronze, negative |
| `was_corrected` | bool |
| `correction_type` | source, target, grammar, orthography, alignment, none |
| `synthetic` | bool |
| `generator` | human, rule, model_name, none |

## Taille cible

Base actuelle: environ 123k paires propres.

Version v1 recommandee:

| Bloc | Calcul | Lignes approx. |
|---|---:|---:|
| Traduction FR -> Moore | 123,587 | 123k |
| Traduction Moore -> FR | 123,587 | 123k |
| Correction Moore synthetique, 2 variantes | 123,587 x 2 | 247k |
| Controle qualite et rejet | anomalies + corruptions | 50k |
| Terminologie et dictionnaire | sous-echantillon | 25k |
| Explication grammaticale courte | sous-echantillon | 25k |
| QA, reformulation, dialogue | sous-echantillon | 30k |

Total v1: environ 620k lignes instruct.

Version v2:

- augmenter les corruptions controlees a 3-4 variantes;
- ajouter transcriptions ASR Moore propres;
- ajouter multi-reference MT;
- ajouter preferences DPO.

Total v2: 800k a 1.2M lignes.

La v1 doit privilegier la qualite. Mieux vaut 600k lignes bien typées que 1M lignes melangees sans controle.

## Taches a generer

### 1. Traduction

FR -> Moore:

```text
Traduis en moore naturel et correct:
{src}
```

Moore -> FR:

```text
Traduis en francais:
{trg}
```

Variantes utiles:

- traduction litterale;
- traduction naturelle;
- traduction administrative;
- traduction simple pour grand public.

### 2. Correction grammaticale Moore

Sources:

- corrections humaines Label Studio;
- corruptions synthetiques controlees;
- sorties faibles de modeles existants;
- anomalies corrigeables.

Exemple:

```text
Corrige ce texte moore. Garde le sens et ne rajoute pas d'information:
{moore_corrompu}
```

Target:

```text
{moore_corrige}
```

Corruptions a generer:

- suppression de diacritiques: `ã`, `ẽ`, `ĩ`, `õ`, `ũ`;
- confusion `ɛ/e`, `ɩ/i`, `ʋ/u/v`;
- ponctuation manquante;
- espaces multiples ou mots colles;
- caracteres OCR parasites;
- mots francais injectes dans une phrase Moore;
- casse incoherente;
- guillemets et apostrophes mal normalises.

### 3. Detection qualite

Utiliser `anomalies_train.csv` comme exemples negatifs.

Taches:

```text
Cette paire francais-moore est-elle une bonne paire de traduction?
Francais: {src}
Moore: {trg}
Reponds par: correct, incorrect, ou a verifier. Explique brievement.
```

Targets:

- `incorrect: ratio de longueur anormal`;
- `incorrect: texte source trop long pour la cible`;
- `incorrect: traduction probablement tronquee`;
- `a verifier: possible variation legitime`;
- `correct: paire coherente`.

Cette tache apprend au modele a ne pas accepter tout texte comme bon Moore.

### 4. Explication grammaticale courte

Ne pas generer de longs raisonnements internes. Produire des explications courtes et verifiables.

```text
Explique brievement la structure grammaticale de cette phrase moore:
{moore}
```

Target:

```text
Sujet: ...
Verbe/action: ...
Complement: ...
Note: ...
```

### 5. Terminologie

Sources prioritaires:

- dictionnaire Niggli;
- glossaire numerique;
- noms de maladies;
- textes experts.

Taches:

- donner la traduction d'un terme;
- expliquer un terme;
- choisir la bonne traduction selon le domaine;
- lister synonymes ou variantes si disponibles.

### 6. QA et comprehension

Construire des questions simples a partir des textes longs propres:

```text
Lis ce texte moore puis reponds en moore:
{passage}
Question: {question}
```

Targets:

- reponse courte;
- reponse avec citation reformulee;
- resume en Moore;
- resume en francais.

### 7. Dialogue

Creer des dialogues controles:

- salutation;
- demande de traduction;
- correction d'une phrase;
- explication d'un mot;
- aide administrative;
- sante simple sans conseil medical dangereux;
- agriculture/elevage.

Le dialogue doit rester ancre dans les donnees et eviter de transformer le modele en chatbot hallucinateur.

## Modele recommande

### Baselines obligatoires

- NLLB 600M actuel;
- NLLB 1.3B si disponible;
- Mistral 7B SACHI existant;
- Gemma/TranslateGemma si le pipeline le permet.

### Candidat principal

Premier candidat recherche:

- `Qwen3-4B-Instruct` en QLoRA;
- `Qwen3-8B-Instruct` si GPU suffisant.

Pourquoi:

- bonne capacite instruction-following;
- bon compromis raisonnement/taille;
- adaptation LoRA realiste;
- meilleur candidat pour correction, explication et dialogue que NLLB.

### Candidat alternatif

- `Gemma 3 4B`;
- `TranslateGemma 4B` comme baseline specialisee traduction.

### Modele leger

Ne pas commencer par un modele <1B pour le raisonnement. Un modele <1B peut devenir un student distille apres avoir obtenu un teacher solide.

Trajectoire:

1. teacher 4B ou 8B;
2. dataset instruct v1;
3. evaluation;
4. distillation vers 1.5B ou 0.6B;
5. quantization int4/int8 pour production.

## Plan d'entrainement

### Phase 0: hygiene donnees

Objectifs:

- aligner les chemins S3 legacy `burkimbia` et canonical `burkimbia-store`;
- materialiser `reviewed_data`;
- corriger le schema final;
- ajouter `domain`, `quality_tier`, `task`;
- dedupliquer avec normalisation.

Sortie:

`burkimbia/moore-instruct-v1-raw`

### Phase 1: DAPT

Continued pretraining sur texte Moore monolingue.

Sources:

- colonne Moore du corpus parallele;
- transcriptions ASR Moore propres;
- proverbes;
- textes longs Moore;
- glossaires et dictionnaires transformes en texte naturel.

But:

- apprendre les caracteres Moore;
- stabiliser l'orthographe;
- reduire les sorties pseudo-Moore.

### Phase 2: SFT multi-taches

Melange recommande v1:

| Tache | Poids |
|---|---:|
| traduction bidirectionnelle | 40% |
| correction grammaticale/orthographique | 25% |
| detection qualite/rejet | 10% |
| terminologie/dictionnaire | 10% |
| comprehension/QA/resume | 10% |
| dialogue/reformulation | 5% |

Parametres de depart:

- QLoRA;
- rank 16 ou 32;
- learning rate `1e-4` a `2e-4`;
- 2 a 3 epochs;
- sequence length 512 pour v1;
- evaluation a chaque epoch;
- pas de packing pour garder les conversations propres au debut.

### Phase 3: DPO ou preference tuning

Construire des preferences:

- correction humaine > original fautif;
- traduction humaine > traduction NLLB > traduction faible;
- reponse courte factuelle > reponse longue hallucinee;
- Moore valide > pseudo-Moore;
- rejet correct > correction inventee.

Objectif:

Ameliorer la discipline du modele, surtout pour la correction grammaticale et la qualite linguistique.

### Phase 4: Distillation

Quand le teacher 4B/8B est bon:

- generer sorties sur prompts non vus;
- filtrer par score qualite et validation humaine;
- fine-tuner un modele plus petit;
- quantifier pour production.

## Evaluation

Il faut separer les scores. Un seul score global cache les vrais problemes.

### Traduction

Metrics:

- chrF++;
- BLEU;
- COMET-QE si disponible;
- evaluation humaine multi-reference.

Sets:

- `mt-benchmark-public`;
- eval privee;
- nouveau subset multi-reference.

### Correction grammaticale

Creer un benchmark `moore-grammar-correction-benchmark`.

Types d'erreurs:

- diacritiques;
- orthographe;
- ponctuation;
- spacing;
- OCR;
- melange francais/moore;
- phrase grammaticalement incoherente.

Metrics:

- exact match sur corrections simples;
- chrF entre correction et reference;
- taux de preservation du sens;
- jugement humain: correct, acceptable, incorrect.

### Detection qualite

Metrics:

- accuracy;
- F1 incorrect;
- false accept rate.

Le false accept rate est critique: un modele qui accepte du mauvais Moore est dangereux pour la qualite du corpus.

### Comprehension et dialogue

Evaluer par grille humaine:

- Moore naturel;
- reponse pertinente;
- pas d'hallucination;
- respect de la consigne;
- capacite a dire "je ne sais pas".

## Plan de travail

### Sprint 1: dataset instruct v1

- corriger les chemins S3 de `labelstudio-processing`;
- exporter `reviewed_data`;
- creer un script `build_moore_instruct.py`;
- generer traduction bidirectionnelle;
- generer correction synthetique;
- integrer anomalies comme taches negatives;
- publier un dataset prive v1.

Livrable:

`burkimbia/moore-instruct-v1-private`

### Sprint 2: premier modele Qwen/Gemma

- config Qwen3-4B QLoRA;
- entrainement SFT v1;
- evaluation sur MT + correction;
- comparaison NLLB/Mistral/Qwen.

Livrable:

`burkimbia/BIA-MOORE-LLM-QWEN3-4B-v1`

### Sprint 3: benchmark correction

- selection de 500 a 1,000 phrases Moore;
- generation d'erreurs controlees;
- correction par locuteurs natifs;
- leaderboard correction grammaticale.

Livrable:

`burkimbia/moore-grammar-benchmark-v1`

### Sprint 4: preference tuning

- collecter paires preferencees;
- DPO sur teacher;
- re-evaluation;
- preparation article.

Livrable:

`BIA-MOORE-LLM-v2`

### Sprint 5: article

Titre possible:

`BIA-MOORE-LLM: A Multi-Task Instruction Model for Translation, Grammar Correction and Linguistic Reasoning in Moore`

Contributions:

1. corpus instruct Moore multi-taches;
2. benchmark correction grammaticale Moore;
3. comparaison NLLB vs LLM instruct;
4. methode d'augmentation controlee pour langue faible ressource;
5. analyse humaine des erreurs.

## Risques

### Trop de donnees religieuses

Mitigation:

- plafonner par domaine;
- sampler par poids;
- evaluer sur domaines non religieux.

### Synthese de mauvaise qualite

Mitigation:

- separer `synthetic=true`;
- limiter a 40-50% du SFT v1;
- filtrer par regles + modele juge + validation humaine.

### Pseudo-Moore

Mitigation:

- DAPT monolingue Moore;
- correction grammaticale;
- lexique de caracteres autorises;
- penaliser sorties avec caracteres suspects;
- eval humaine.

### Confusion des codes langue

Mitigation:

- unifier dans les datasets internes: `fr`, `moore`;
- documenter mapping externe: `fra_Latn`, `mos_Latn`, `moo`;
- eviter `moor_Latn` sauf si necessaire pour compatibilite legacy.

### Benchmark trop faible

Mitigation:

- multi-reference;
- themes equilibres;
- evaluation humaine;
- split prive non contamine.

## Priorite immediate

La prochaine action technique doit etre:

1. reparer/aligner `labelstudio-processing` avec les bons buckets;
2. exporter le dataset `reviewed_data`;
3. creer le builder `moore-instruct-v1`;
4. entrainer Qwen3-4B sur 600k lignes maximum;
5. evaluer face a NLLB, Mistral et Gemma.

## References utiles

- LoResLM 2026 French-Moore MT: <https://aclanthology.org/2026.loreslm-1.53/>
- NLLB: <https://ai.meta.com/research/no-language-left-behind/>
- Qwen3 model family: <https://huggingface.co/Qwen>
- TranslateGemma: <https://blog.google/innovation-and-ai/technology/developers-tools/translategemma/>
- AfriNLLB / efficient African MT direction: <https://arxiv.org/abs/2602.09373>
