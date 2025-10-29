# deploiement llm sans cluster

Notre equipe Burkimbia doit servir des modeles pour des langues africaines sans exploser un budget limite a vingt euros par mois. Ce billet raconte comment nous avons arbitre entre plusieurs fournisseurs et pourquoi le mode serverless reste aujourd'hui notre garde fou. Pas de pas a pas ici, uniquement les choix strategiques et leurs impacts.

## nos contraintes terrain

- usage intermittent: environ trois mille requetes mensuelles, chacune autour de 8.5 secondes de calcul GPU.
- public fragile: trente pour cent des requetes sont offertes pour des apprenantes et apprenants sans moyens.
- equipe reduite: pas de temps pour maintenir un cluster ECS ou EKS.
- transparence: tout cout doit etre justifiable pour nos financeurs.

Ces contraintes rendent toute facturation continue impraticable. Louer un T4 en permanence chez AWS ou Azure revient a plus de 360 dollars mensuels. Avec le mode serverless, seules les secondes effectives sont facturees.

## comparatif rapide des plateformes

| plateforme | gpu | modele de facturation | cout mensuel estime | cout pour 1000 requetes |
|------------|-----|-----------------------|---------------------|-------------------------|
| aws ec2 | g4dn.xlarge | 720 heures payees | 378.72 usd | 126.24 usd |
| azure vm | nc4as_t4_v3 | 720 heures payees | 378.72 usd | 126.24 usd |
| hugging face endpoints | t4 | 720 heures payees | 360.00 usd | 120.00 usd |
| runpod flex | rtx a6000 | a la seconde | 8.67 usd | 2.89 usd |

Les chiffres proviennent des grilles officielles d'octobre 2025. Seul RunPod Flex respecte notre budget tout en offrant une A6000 pour les modeles de 7b quantifies.

## lessons retenues apres un incident couteux

Un oubli de execution timeout a prolonge un worker RunPod pendant plusieurs jours. Plus de cinq mille requetes de ping ont maintenu la session ouverte et genere des frais serverless importants. Depuis, nous appliquons trois regles:

1. fixer idle_timeout et execution_timeout pour chaque endpoint.
2. limiter max_workers a 1 tant que la charge reste faible.
3. monitorer la frequence de ping via nos journaux et couper tout traffic suspect.

## images docker minimalistes

Nos premieres images PyTorch depassaient neuf gigaoctets. Cela rallongeait chaque cold start. Nous sommes passes a une base Nvidia CUDA puis installons seulement les builds necessaires (Torch, BitsAndBytes, SentencePiece). Chaque image contient directement le modele quantifie pour eviter un telechargement au demarrage. Avec cette approche, les cold start descendent sous quinze secondes sur A6000 et les surprises de facturation disparaissent.

## quantification et matrice gpu

Nous ne lancons aucun modele non quantifie sur RunPod. Mistral 7b passe de vingt gigaoctets en FP16 a environ quatre gigaoctets en INT4. Cela rend l'image plus legere et nous autorise a rester sur A6000 ou meme A4000 pour des charges plus modestes. La regle interne: quantifier d'abord, tester ensuite le plus petit GPU disponible, ne monter en gamme que si la latence explose.

## suivi budgetaire en temps reel

Trois indicateurs sont exposes dans notre tableau de bord Metabase:

- cold_start_seconds: objectif sous quinze secondes.
- gpu_seconds_total: converti en cout temps reel en euros.
- cost_per_1000_requests: alerte quand on depasse 3 usd.

Nous declenchons une alerte Slack quand quatre-vingts pour cent du budget mensuel sont consommes. La transparence vis-a-vis des partenaires rend plus faciles les demandes de financement ponctuel.

## stack hybride recommandee

- production: endpoints RunPod Serverless en RTX A6000 avec timeouts serre.
- prototypes publics: Hugging Face Spaces ZeroGPU a dix dollars pour heberger des demos Gradio.
- experiments internes: instances locales consumer GPU pour iterer sur les quantifications.

Cette combinaison maintient les couts sous vingt euros par mois tout en gardant une vitrine publique simple a partager.

## checklist strategique avant tout nouveau modele

1. verifier la taille en FP16 puis en INT4 et mettre a jour le plan de build Docker.
2. recalculer le cout par mille requetes avec les nouvelles latences mesurees.
3. mettre a jour idle_timeout, execution_timeout et les limites max_workers.
4. ajouter le modele au tableau de bord couts et tester les alertes.
5. documenter l'impact pour les financeurs avant le lancement public.

Rester frugal n'est pas un choix ideologique mais une condition de survie pour une organisation a but non lucratif. Cette fiche conceptuelle doit guider toute nouvelle initiative autour des LLM chez Burkimbia.
