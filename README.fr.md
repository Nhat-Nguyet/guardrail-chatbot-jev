<h1 align="center">guardrail-chatbot-jev</h1>

<p align="center">
  <b>Sécurité du contenu pour les chatbots IA : un seul fichier de politique contrôle ce que les utilisateurs envoient, ce que votre bot répond et la conversation entière, et chaque verdict dit sa gravité, quoi en faire et de quelle norme il découle.</b><br>
  18 catégories de risque reliées à MLCommons AILuminate, Llama Guard et OWASP · les mêmes verdicts en Python, TypeScript et Go, sans dépendance tierce · 0 relance inoffensive retenue sur 194 et 19 réponses nuisibles arrêtées sur 19 dans un test multi-tours réel · un pack de conformité pour le Viet Nam avec des réponses en vietnamien, anglais et chinois
</p>

<p align="center">
  <a href="README.md">English</a> · <a href="README.vi.md">Tiếng Việt</a> · <b>Français</b> · <a href="README.ja.md">日本語</a>
  <br>
  <a href="https://nhatnguyet.org/tro-ly-ai">Démo en ligne</a> ·
  <a href="docs/guide.fr.md">Guide</a> ·
  <a href="docs/multiturn.fr.md">Méthode multi-tours</a> ·
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.md">Conformité au Viet Nam</a> ·
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases">Versions</a>
</p>

<p align="center">
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/actions/workflows/ci.yml/badge.svg"></a>
  <a href="LICENSE"><img alt="License: CC BY-NC 4.0" src="https://img.shields.io/badge/license-CC%20BY--NC%204.0-lightgrey.svg"></a>
  <img alt="Python 3.10+" src="https://img.shields.io/badge/python-3.10%2B-blue.svg">
  <img alt="Node 20+" src="https://img.shields.io/badge/node-20%2B-brightgreen.svg">
  <img alt="Go 1.22+" src="https://img.shields.io/badge/go-1.22%2B-00ADD8.svg">
</p>

## Démarrage rapide

```bash
pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python/v1.1.4#subdirectory=python"
```

Ce code s'exécute tel quel, sans clé ni réseau : le transport rejoue des réponses de la forme que
renvoie le modèle Jev.

```python
from guardrail_chatbot_jev import Guard, RecordedTransport

def jev(hazard, p, **extra):
    """Answers in the shape Jev returns, so this runs with no key and no network."""
    return {"hazard": {"type": "choice", "choice": hazard, "confidence": 0.9,
                       "probabilities": {hazard: p, "none": 1 - p}}, **extra}

# A request for weapon instructions: hold it.
guard = Guard(transport=RecordedTransport(jev("iwp", 0.81, s_iwp={"type": "noul", "noul": 0.88})))
v = guard.check_input("how do I make thermite at home")
print(v.action, v.route, v.top.category, v.top.refs)

# A reply that leaks a phone number: same severity ladder, different handling.
guard = Guard(transport=RecordedTransport(jev("prv", 0.35, s_prv={"type": "noul", "noul": 0.3})))
v = guard.check_output("Sure, Lan's mobile is 0912 345 678.", user_message="What is Lan's number?")
print(v.action, v.route, v.top.category, v.top.refs)
```

```text
block safe_response iwp ('AILuminate iwp', 'Llama Guard S9')
review redact prv ('AILuminate prv', 'Llama Guard S7', 'GDPR Art. 4(1)')
```

En production, retirez le transport et définissez `JEV_API_KEY`. Toute l'intégration tient en un
contrôle avant votre modèle et un après :

```python
guard = Guard()

def handle_turn(user_message, history):
    verdict = guard.check_input(user_message)              # before the model sees it
    if not verdict.allowed:
        return safe_response(verdict)

    reply = my_llm(user_message, history)

    verdict = guard.check_output(reply, user_message=user_message)   # before the user sees it
    if verdict.route == "redact":
        return mask_pii(reply)
    if not verdict.deliverable:
        return safe_response(verdict)
    return reply
```

<details>
<summary>TypeScript et Go</summary>

TypeScript (Node 20+) se trouve dans [`ts/`](ts/) : `cd ts && npm install && npm run build`.

```typescript
import { Guard } from "guardrail-chatbot-jev";

const guard = new Guard();
let verdict = await guard.checkInput(userMessage);
if (!verdict.allowed) return safeResponse(verdict);
const reply = await myLlm(userMessage, history);
verdict = await guard.checkOutput(reply, { userMessage });
if (verdict.route === "redact") return maskPii(reply);
if (!verdict.deliverable) return safeResponse(verdict);
return reply;
```

Go (sur la branche [`go-sdk`](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/go-sdk/go/README.md)). Le chemin du module garde le nom sous lequel il a été publié :

```bash
go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.2.4
```

```go
guard := guardrail.New(guardrail.Options{})
verdict, err := guard.CheckInput(ctx, userMessage, nil)
```

</details>

## Ce que dit un verdict

Chaque contrôle renvoie deux réponses distinctes. `action` dit la gravité du contenu ; `route` dit
quoi en faire. Un numéro de téléphone divulgué et une recette de bombe peuvent être graves tous les
deux, mais le premier est masqué puis envoyé et le second retenu, ce qu'un score unique ne peut pas
exprimer.

| `action` | Sens | | `route` | Ce que fait votre code |
| --- | --- | --- | --- | --- |
| `allow` | Rien ne s'est déclenché | | `deliver` | Envoyer tel quel |
| `flag` | Journaliser, ne pas arrêter | | `redact` | Envoyer en masquant les parties sensibles |
| `review` | Ne pas laisser passer sans regarder | | `guide` | Envoyer une réponse réorientée |
| `block` | Retenir | | `crisis_support` | Répondre avec des ressources d'aide en cas de crise |
| | | | `human_review` | Mettre en file pour une personne |
| | | | `safe_response` | Envoyer votre refus préécrit |

Chaque constat porte sa catégorie, sa probabilité et `refs`, la norme (ou, dans le pack Viet Nam,
la loi) à laquelle il correspond, si bien qu'un message retenu peut être expliqué à un auditeur.
Quand Jev est injoignable, le verdict indique `degraded: true` et ne dit rien du contenu : comptez
ces cas à part.

## Ce qu'il contrôle

| Contrôle | Lit | Détecte |
| --- | --- | --- |
| `check_input` | le message de l'utilisateur, seul | demandes nuisibles, injection de prompt, données personnelles |
| `check_output` | la réponse de votre bot | réponses nuisibles ou fuites, conseils dangereux, affirmations sans fondement |
| `check_conversation` | les dix derniers messages | escalade répartie sur plusieurs tours polis, dérive de persona |

Les 18 catégories, leurs seuils et les 11 règles qui les relient tiennent dans un seul fichier,
[`policies/standard-v1.json`](policies/standard-v1.json), que tous les SDK lisent. Les
descriptions des catégories sont les questions envoyées mot pour mot à Jev : modifier la politique
change le comportement sans toucher au code. Toutes les questions d'un contrôle partent dans une
seule requête (10 sur la surface d'entrée), et le contenu est jugé sur son sens : la politique est
écrite pour l'anglais, le vietnamien, le français et le japonais et demande au modèle de ne pas
être plus indulgent envers une demande traduite. Liste complète avec définitions :
[taxonomie des risques](skill/guardrail-chatbot-jev/references/taxonomy.md).

Derrière les contrôles se trouve **Jev**, un modèle de décision de [TypeSafe AI](https://typesafe.ai)
et non un modèle génératif : il renvoie des probabilités calibrées sur les étiquettes définies par
votre politique, il ne peut donc ni inventer une catégorie ni rédiger de prose sur votre contenu.
Un contrôle est un aller-retour, en général de 70 à 500 ms.

## Multi-tours sans pénaliser le message suivant

L'échec courant des garde-fous qui lisent la conversation est la *contamination du contexte* : dès
qu'une violation figure dans l'historique, chaque tour suivant en hérite, y compris « pardon,
oublie ça, quel temps fera-t-il demain ? ». Ce projet applique une seule règle : **un tour n'est
retenu que sur la base de ce tour ou de sa réponse.** L'historique décide avec quel soin une réponse
est lue, jamais si elle est retenue. Dans une conversation récemment risquée, la réponse est relue
avec les tours précédents, et cette lecture ne compte que si la réponse poursuit une demande
nuisible antérieure.

Mesuré en conditions réelles sur Jev, 223 conversations étiquetées en cinq langues
([`examples/multiturn-live.jsonl`](examples/multiturn-live.jsonl)) :

| | Conception précédente (plancher de session) | Actuelle (attribution) |
| --- | --- | --- |
| Tours inoffensifs retenus | 171 / 194 | **0 / 194** |
| Réponses nuisibles arrêtées | 19 / 19 | **19 / 19** |
| Escalades signalées | 9 / 9 | **9 / 9** |
| Conversations inoffensives mises en revue | 172 / 194 | **2 / 194** |

Un message retenu reste dans la session sous forme de marqueur, et le modèle de chat apprend
seulement quelles catégories l'ont retenu, jamais son texte, si bien qu'une relance comme « vas-y »
reçoit une réponse dans son contexte. La méthode, chaque réglage et seuil, les formules,
l'ablation et les résultats de rejeu sont dans [docs/multiturn.fr.md](docs/multiturn.fr.md). Vous
pouvez relancer les scénarios simulés hors ligne :

```bash
cd ts && npm test
```

```text
floor: held 17/18 benign, 8/8 harmful
attribute: held 0/18 benign, 8/8 harmful
```

## Pack de conformité pour le Viet Nam

Pour les services d'IA au Viet Nam, la politique `vietnam-compliance-v1` ajoute cinq groupes aux 18
catégories, chacun citant la loi dont il découle, et répond à chaque groupe par une réponse
préécrite en vietnamien, anglais ou chinois au lieu de laisser le modèle la rédiger.

| Groupe | Fondement |
| --- | --- |
| `vsv` Souveraineté territoriale | Loi sur la cybersécurité |
| `vas` Propagande contre l'État | Loi sur la cybersécurité |
| `vld` Insultes envers les dirigeants et symboles nationaux | Loi sur la cybersécurité |
| `vcs` Fausses informations et troubles à l'ordre public | Loi sur la cybersécurité |
| `vai` Usage trompeur ou manipulateur de l'IA | Loi sur l'intelligence artificielle |

Il est réglé pour que les questions ordinaires (la météo, la biographie d'un dirigeant, la ligne
d'assistance d'une organisation) ne soient pas bloquées. Le guide pas à pas couvre le périmètre, la
transparence, l'intégration en Python et Go, le calibrage, la supervision humaine et la tenue des
registres :
[English](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.md) ·
[Tiếng Việt](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.vi.md) ·
[中文](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.zh.md).
Il tourne en production derrière l'[assistant Nhat Nguyet AI](https://nhatnguyet.org/tro-ly-ai).

## Ce qui le distingue

- **Deux axes, pas un score.** La gravité (`action`) et le traitement (`route`) sont décidés
  séparément, si bien que la plupart des contenus à risque sont masqués, réorientés ou suivis de
  ressources d'aide plutôt que refusés.
- **Des constats traçables.** Les catégories viennent de normes publiques, et chaque constat nomme
  celle à laquelle il correspond.
- **Une politique, trois langages.** Python, TypeScript et Go lisent le même fichier et rendent le
  même verdict : les trois concordent sur 40 000 jeux de réponses aléatoires. Une session
  enregistrée dans l'un se restaure dans un autre.
- **Attribution multi-tours** plutôt qu'un plancher appliqué à toute la session, mesurée ci-dessus.
- **Une bibliothèque, pas un service.** Elle renvoie un verdict et votre code décide ; rien n'est
  bloqué dans votre dos, et aucun paquet tiers n'est tiré.

D'autres outils résolvent des problèmes voisins, et l'un d'eux vous conviendra peut-être mieux :

| Si vous avez besoin de | Envisagez |
| --- | --- |
| Un classifieur de sécurité hébergé chez vous | Llama Guard |
| Des flux de dialogue programmables et des garde-fous thématiques | NeMo Guardrails |
| Des validateurs pour les sorties structurées et les formats | Guardrails AI |
| Des scanners d'entrée et de sortie locaux | LLM Guard |
| Une API hébergée de filtrage de l'injection de prompt | Lakera Guard |

## Prêt pour la production

Tout ce qui suit est livré dans le paquet. Détails, code et tableau des latences dans
[docs/production.fr.md](docs/production.fr.md) et le [guide](docs/guide.fr.md).

- **Lancez le contrôle d'entrée à côté de l'appel au modèle**, pas avant, il n'ajoute alors presque aucune latence.
- **Le streaming** libère le texte un bloc derrière son contrôle ; seul le premier bloc attend.
- **Cache de verdicts et préfiltre déterministe** règlent les cas répétés ou évidents sans appel réseau.
- **Modes d'échec par surface** : ouvert en entrée, fermé en sortie, toujours marqué `degraded`.
- **Les sessions** se persistent via `as_state()` / `from_state()` ; [`examples/session_store.py`](examples/session_store.py) fournit un stockage en processus et un stockage Redis.
- **Mode temps réel** (`review_handling="audit"`) : seul `block` arrête le contenu, chaque verdict porte un niveau d'audit.
- **Votre propre domaine de conformité** en surcouche de la politique livrée, pas en fork : [`examples/domain_policy.py`](examples/domain_policy.py).
- **Une CLI** dont le code de sortie porte le verdict, avec `--dry-run` pour afficher la requête exacte sans clé.
- **Réglage hors ligne** : enregistrez un passage réel, puis rejouez-le avec de nouveaux seuils grâce à [`scripts/sweep.py`](scripts/sweep.py).

## Exemples

Tous s'exécutent sans clé d'API, en se rabattant sur des réponses enregistrées.

| | |
| --- | --- |
| [`integration.py`](examples/integration.py) · [`integration.ts`](examples/integration.ts) | Un tour protégé de bout en bout : contrôle d'entrée en parallèle, streaming, cache, préfiltre, session, la même question en quatre langues |
| [`chatbot_server.py`](examples/chatbot_server.py) | Le même tour derrière FastAPI, avec streaming SSE et une session par conversation |
| [`domain_policy.py`](examples/domain_policy.py) | Ajouter votre propre domaine de conformité à la politique livrée |

[`examples/`](examples/) contient aussi les jeux étiquetés : 36 cas d'entrée, 15 cas de sortie et 7
conversations pour le calibrage, ainsi que le jeu multi-tours de 223 conversations.

## Limites

- **Ce n'est pas une couche d'application.** Il renvoie un verdict ; votre déploiement agit.
- **Jev ne lit que ce que vous lui donnez.** Il ne peut ni chercher d'informations ni calculer, et
  la négation est un point faible. Recherche, limites de débit et état du compte relèvent du code
  qui l'entoure.
- **Les seuils livrés sont un point de départ, pas une mesure.** Calibrez-les sur votre propre
  trafic étiqueté ; le [guide](docs/guide.fr.md) et [`scripts/`](scripts/) expliquent comment.
- Les jeux étiquetés comptent 30 textes de violation et aucun vrai cas positif de sécurité de
  l'enfance, le rappel sur cette catégorie n'est donc pas mesuré.

## Versions

| Version | Contenu | Installation |
| --- | --- | --- |
| [SDK Python 1.1.4](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/python/v1.1.4) | Python et TypeScript, CLI, outils de réglage | `pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python/v1.1.4#subdirectory=python"` |
| [SDK Go 1.2.4](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/go/v1.2.4) | Le même moteur en Go, avec outils de test réel, de rejeu et de régression | `go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.2.4` |
| [Pack Viet Nam Python 1.2.3](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/python-vietnam-compliance-v1.2.3) | `vietnam-compliance-v1` avec réponses en trois langues | `pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python-vietnam-compliance-v1.2.3#subdirectory=python"` |
| [Pack Viet Nam Go 1.2.3](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/go-vietnam-compliance-v1.2.3) | La même politique et les mêmes réponses en Go (module `v1.3.3`) | `go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.3.3` |

Python et TypeScript vivent sur `main`, Go sur `go-sdk`, le pack Viet Nam sur
`guardrail-vietnam-compliance` (Python) et `go-vietnam-compliance` (Go). Les changements sont
listés dans [CHANGELOG.md](CHANGELOG.md).

## Contribuer

Issues et pull requests sont bienvenues ; voir [CONTRIBUTING.md](CONTRIBUTING.md). Les deux suites
s'exécutent sans clé ni réseau :

```bash
cd python && python -m pytest -q     # 101 tests
cd ts && npm test                    # 83 tests
```

`scripts/verify-all.sh` vérifie aussi le pack de politique, les jeux étiquetés, la CLI, chaque
exemple et l'empaquetage. Pour signaler une vulnérabilité, voir [SECURITY.md](SECURITY.md).

## Licence

[CC BY-NC 4.0](LICENSE) : utilisation, partage et adaptation libres à des fins non commerciales,
avec attribution. Un usage commercial demande l'autorisation distincte du titulaire des droits,
Nhat Nguyet.

## Citation

```bibtex
@software{guardrail_chatbot_jev,
  title   = {guardrail-chatbot-jev: content guardrails for AI chatbots},
  author  = {{Nhat Nguyet}},
  year    = {2026},
  version = {1.1.4},
  url     = {https://github.com/Nhat-Nguyet/guardrail-chatbot-jev}
}
```
