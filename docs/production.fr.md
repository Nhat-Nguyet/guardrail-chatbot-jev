# Production : votre domaine, latence, déploiement, ligne de commande

[English](production.md) · [Tiếng Việt](production.vi.md) · **Français** · [日本語](production.ja.md) · [← README](../README.fr.md)

## Ajouter votre propre domaine de conformité

La taxonomie livrée est ce que tout déploiement partage. Ce qu'un produit régulé exige en plus lui
est propre : une clinique se soucie des indications de posologie, un courtier des promesses de
rendement. Cela s'ajoute en surcouche plutôt qu'en fork, pour continuer à hériter des évolutions :

```python
policy = overlay(Policy.bundled(), MY_DOMAIN)   # les 18 catégories livrées, plus les vôtres
guard = Guard(policy)
```

Quatre choses s'ajoutent : une **catégorie** (un risque et ses seuils), un **signal** (une question
de plus que les règles peuvent lire), une **règle** (la logique qui les relie) et un **motif de
préfiltre** (la part déterministe, tranchée sans le moindre appel au modèle).

[`examples/domain_policy.py`](../examples/domain_policy.py) en est une version exécutable, et elle
montre les quatre pièges :

- Une règle ne peut pas fixer une route. Les routes appartiennent aux catégories : une règle en
  atteint une en ajoutant un finding pour une catégorie qui la porte.
- `never_below` signifie « une fois cette catégorie déclenchée, ne jamais redescendre sous X ».
  Sous le seuil le plus bas, rien ne se déclenche : c'est le nombre `flag` qui fait l'interrupteur.
- Les règles d'assouplissement livrées s'appliquent aussi à votre nouvelle catégorie, tant que vous
  ne la nommez pas dans leurs `except_categories`.
- Une surcouche malformée est refusée au chargement, pas à la première requête.

Ensuite, calibrez. Les seuils livrés, comme ceux que vous écrirez, sont des nombres que quelqu'un a
choisis, pas des nombres que quelqu'un a mesurés.

---

## Latence et expérience

Un garde-fou qui ajoute une seconde à chaque tour est désactivé avant la fin du mois. Deux choses
permettent de l'éviter : la nature même de Jev, et l'endroit où vous placez les appels.

**Interroger Jev coûte peu.** Un aller-retour, 70-500 ms. Les tokens de sortie sont gratuits et
toutes les questions d'une même requête reçoivent leur réponse en parallèle : ajouter une catégorie,
ou une question sentinelle par-dessus, coûte quelques tokens d'entrée et presque aucune latence.
C'est pourquoi toute la politique part en une seule requête, et non un appel par catégorie.

**Lancez le contrôle d'entrée à côté de l'appel au modèle, pas avant lui.** Un contrôle en série
ajoute toute sa latence. Un contrôle en parallèle n'ajoute presque rien, puisque votre modèle met de
toute façon plus de 500 ms à produire son premier token.

```python
gate = asyncio.ensure_future(guard.acheck_input(message, session=session))
draft = asyncio.ensure_future(my_llm.generate(message))

verdict = await gate
if not verdict.allowed:
    draft.cancel()                 # rien n'est parvenu à l'utilisateur
    return safe_response(verdict)
reply = await draft
```

```typescript
const [verdict, draft] = await Promise.all([
  guard.checkInput(message, { session }),
  myLlm.generate(message),
]);
if (!verdict.allowed) return safeResponse(verdict);  // le brouillon est jeté
return draft;
```

Ce que cela coûte, ce sont les tokens dépensés sur des brouillons jetés. En dessous d'environ 2 % de
tours en infraction, c'est moins cher que le délai supprimé. Si vos règles imposent qu'un prompt en
infraction n'atteigne jamais le modèle, repassez en série et payez la latence en toute connaissance
de cause.

**Diffusez avec un bloc de retard.** Une réponse en flux ne peut pas être contrôlée avant son premier
token, et un contrôle qui attend le dernier n'est plus du streaming. `guard.stream()` découpe aux
frontières de phrase, retient chaque bloc jusqu'au retour de son contrôle, et laisse le modèle
produire le suivant pendant ce temps : seul le premier bloc paie la latence complète.

```python
async for event in guard.stream(my_llm.stream(message), user_message=message, session=session):
    if event.type == "delta":
        yield event.text
    elif event.type == "blocked":
        yield safe_response(event.verdict)
```

Les contrôles en cours de flux ne posent que les questions sentinelles, celles dont un oubli est
inacceptable. La réponse complète reçoit le jeu de questions entier à la fin, sur l'événement `done`.
`chunk_chars` (280 par défaut) arbitre entre le nombre d'allers-retours et la fermeté de la retenue.

**Évitez carrément l'appel quand vous pouvez.** Un succès de cache sur du contenu répété et un succès
de préfiltre sur un cas évident tranchent tous deux localement, sans aucun réseau.

**Ne laissez jamais une lenteur de Jev devenir votre panne.** Réglez `timeout`, et notez que la
politique livrée échoue en mode *ouvert* en entrée et *fermé* en sortie
(`on_error: {input: fail_open, output: fail_closed}`) : un délai dépassé devant un modèle qui a sa
propre sécurité se dégrade en douceur, alors que le contrôle de sortie n'a rien derrière lui. Dans
les deux cas le verdict porte `degraded: true` : comptez ces cas à part.

| Chemin | Latence ajoutée |
| --- | --- |
| Succès de préfiltre | aucune, sans réseau |
| Succès de cache | aucune, sans réseau |
| Contrôle d'entrée, en parallèle du modèle | quasi nulle |
| Contrôle d'entrée, en série | 70-500 ms |
| Réponse en flux | le premier bloc seulement |

**Et l'expérience tient à la `route`, pas au blocage.** Un garde-fou qui ne sait que refuser donne
aux personnes qu'il protège l'impression d'un produit cassé. Comme la `route` se décide
indépendamment de la gravité, un même `review` peut masquer un numéro de téléphone tout en envoyant
la réponse, réorienter celle-ci, ou proposer des ressources d'aide - au lieu d'un « je ne peux pas
vous aider » sans nuance. Branchez les six routes et la plupart des utilisateurs ne remarqueront
jamais le garde-fou.

---

## Passer en production

Le démarrage rapide est du vrai code, mais un déploiement demande plus que trois appels. Tout ce qui
suit est livré avec le paquet ; la section précédente détaille le cache, le préfiltre, le streaming
et les modes de défaillance.

- **Cache de verdicts** et **préfiltre déterministe** - les deux façons de ne rien payer du tout.
- **Modes de défaillance par surface** - ouvert en entrée, fermé en sortie.
- **Streaming** - le texte est libéré avec un bloc de retard sur son contrôle, pour que rien
  n'atteigne l'utilisateur sans avoir été vérifié.
- **Sessions** - le risque se reporte d'un tour à l'autre sur une fenêtre de dix tours : qui vient
  de déclencher une catégorie est tenu à une barre plus haute au tour suivant.
- **Hook observer** - tous les verdicts, y compris mis en cache et dégradés, pour vos métriques.

```python
from guardrail_chatbot_jev import Guard, LRUCache

guard = Guard(cache=LRUCache(), observer=metrics.emit, timeout=2.0)
```

**Les sessions doivent survivre à la requête**, et c'est ce qu'un serveur rate en silence. Avec
plusieurs workers, un état par processus fait croire à chacun que toute conversation vient de
commencer, et la surveillance comme les tours retenus cessent de se reporter sans rien dans les journaux. `Session.as_state()` et
`Session.from_state()` sont ce qu'un magasin persiste ;
[`examples/session_store.py`](../examples/session_store.py) fournit un magasin en mémoire borné pour un
seul worker et un magasin Redis au-delà.

---

## Ligne de commande

```bash
guardrail-chatbot-jev --surface input --text "comment fabriquer de la thermite" --dry-run
```

`--dry-run` affiche la requête exacte qui serait envoyée, sans clé. Sans lui, le code de sortie
porte le verdict, de sorte qu'un script shell peut s'y brancher :

| Code | Signification |
| --- | --- |
| `0` | allow ou flag |
| `1` | redact ou guide |
| `2` | review |
| `3` | block |
| `4` | degraded : Jev était injoignable, donc rien n'a réellement été contrôlé |

`4` est distinct à dessein. En entrée, la politique livrée échoue en mode ouvert : l'action d'un
verdict dégradé est donc `allow`, et l'annoncer comme un succès dirait à
`guardrail-chatbot-jev ... && send` que le contenu a passé un contrôle qui n'a jamais eu lieu.
