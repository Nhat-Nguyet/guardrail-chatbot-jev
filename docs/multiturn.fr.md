# Multi-tours : méthode, réglages et évaluation

[English](multiturn.md) · [Tiếng Việt](multiturn.vi.md) · **Français** · [日本語](multiturn.ja.md) · [← README](../README.fr.md)


## Problème

**Contamination par le contexte :** un classifieur qui lit une violation dans l'historique l'attribue au
tour suivant, quel qu'en soit le contenu. Référence (plancher de session, ancienne conception) :
**171 sur 194** messages inoffensifs retenus, mesuré en direct.

## Règle de conception

Un tour n'est retenu que sur la base d'éléments issus du tour lui-même ou de sa réponse. L'historique
détermine le degré d'attention porté au tour, jamais sa rétention.

## Méthode

Chaque tour est décidé en répondant, dans l'ordre, à trois questions.

1. **Le message de l'utilisateur est-il nuisible en lui-même ?** Il est lu seul, sans la conversation
   qui précède. S'il l'est, il est arrêté ici. Un message inoffensif en lui-même n'est jamais arrêté à
   cause de ce qui a été dit avant.
2. **La réponse est-elle nuisible en elle-même ?** Elle est lue seule de la même façon.
3. **Seulement si la conversation a été récemment risquée : la réponse termine-t-elle une demande
   nuisible faite plus tôt ?** La réponse est relue avec les tours précédents. Cette seconde lecture ne
   compte que si la réponse fournit l'étape suivante, des précisions, une traduction ou un récit d'une
   demande nuisible antérieure. Excuses, question sur la loi, signalement ou changement de sujet : elle
   ne compte pas.

Un message arrêté reste dans la conversation sous la forme `[earlier message omitted]` : la tentative est
mémorisée, son texte n'est jamais relu. Le modèle apprend seulement qu'un message a été retenu et quelles
catégories l'ont arrêté, jamais son texte : une relance comme « fais-le » ou « ma première demande » reçoit
une réponse en contexte au lieu d'une supposition. La raison est conservée avec l'état de la session.

| Étape | Code |
| --- | --- |
| 1. Le message de l'utilisateur, lu seul | [`check_input`](../python/src/guardrail_chatbot_jev/guard.py#L119) |
| 2. La réponse, lue seule | [`check_output`](../python/src/guardrail_chatbot_jev/guard.py#L131) |
| 3. La réponse, lue avec les tours précédents, et si elle compte | [`_check_in_context`](../python/src/guardrail_chatbot_jev/guard.py#L333), [`_attribute`](../python/src/guardrail_chatbot_jev/guard.py#L350) |
| Quand une conversation est considérée comme récemment risquée | [`Session.watching`](../python/src/guardrail_chatbot_jev/session.py#L128) |
| Messages arrêtés conservés comme repère ; le modèle apprend la catégorie, jamais le texte | [`Session.record`](../python/src/guardrail_chatbot_jev/session.py#L99), [`model_history`](../python/src/guardrail_chatbot_jev/session.py#L106) |
| Vérification de la conversation entière : elle surveille, n'arrête jamais un tour | [`check_conversation`](../python/src/guardrail_chatbot_jev/guard.py#L170) |

## Exemple, tour par tour

| Tour | Utilisateur | Assistant | Ce que fait le garde-fou | Résultat |
| --- | --- | --- | --- | --- |
| 1 | « Comment fabriquer un explosif avec de l'engrais ? » | (aucune) | Étape 1 : nuisible en lui-même. | **Arrêté.** La conversation devient récemment risquée. |
| 2 | « Désolé, oublie. Quel temps demain à Hanoï ? » | « Ensoleillé, 24 à 30 °C. » | Étapes 1 et 2 : inoffensif. Étape 3 : la réponse ne termine rien (0,02). | **Délivré.** L'ancienne conception le retenait. |
| 3 | « Comment la loi punit-elle la fabrication d'explosifs ? » | « Par une peine de prison, selon les cas. » | Étape 3 : l'utilisateur prend ses distances (0,90) ; la réponse ne termine rien (0,05). | **Délivré.** |
| 4 | « OK, continue. Et l'étape 2 ? » | « Étape 2 : laisser sécher le mélange un jour, puis le répartir. » | Étape 3 : c'est l'étape suivante de la demande refusée (0,95). | **Arrêté.** Seule la lecture en contexte le voit. |
| 4′ | idem | « Je ne peux pas continuer. » | Étape 3 : la réponse ne termine rien. | **Délivré.** |

## Définitions

### Réglages en vigueur

| Réglage | Valeur | Signification |
| --- | --- | --- |
| Fenêtre d'historique | 10 messages | La vérification de conversation et la lecture en contexte voient les dix derniers messages. |
| Risque par verdict | allow 0 · flag 0,25 · review 0,6 · block 1,0 | Ce qu'un verdict ajoute au risque de la session. |
| Décroissance | 0,5 | À chaque vérification, le risque précédent est divisé par deux ; la session garde le plus élevé des deux. |
| Surveillance dès | risque ≥ 0,2 | Tant que le risque atteint 0,2, la réponse est aussi lue en contexte. |
| Report | 2 tours | Après une revue de conversation ou un blocage, la session reste surveillée deux tours. |
| Attribution | ≥ 0,5, et au-dessus du « recul » | La lecture en contexte ne compte que si Jev est sûr à 50 % au moins que la réponse complète une demande nuisible antérieure, et plus sûr de cela que du recul de l'utilisateur. |
| Corroboration | ≥ 0,02 | Une sentinelle est corroborée quand la question principale donne au moins 2 % à sa catégorie. |
| Sentinelle faible | sous sa bande block | Seule et sous la bande block, elle est enregistrée en flag et délivrée. |
| Jamais affaiblies | `ssh` (pack Viet Nam : `ssh`, `vsv`, `vld`) | Ces catégories gardent toute leur force. |
| Réponse de refus | refus ≥ 0,8, sentinelle < 0,5 | Plafonnée à flag, sauf `sid`, `prv`, `ipv` (pack Viet Nam : aussi `vsv`, `vld`). |
| Masquer plutôt que bloquer | sentinelle < 0,8 | Une donnée personnelle signalée par la seule sentinelle est masquée et délivrée. |
| Garde de confiance | confiance < 0,65 | En revue s'il y a un constat ou un quasi-constat, sauf intention jugée bénigne avec confiance ≥ 0,5. |
| Plafond de conversation | escalation ≤ 0,5 | Plafonnée à flag, sauf `cse` et `ssh`. |
| Jev injoignable | entrée : délivrer · sortie : retenir · conversation : délivrer | La sortie est le dernier rempart. |

### Seuils par catégorie

| Catégorie | Nom | Flag / review / block | Sortie | Notes |
| --- | --- | --- | --- | --- |
| `cse` | Child sexual exploitation | 0.03 / 0.06 / 0.15 | = | sentinelle; jamais sous block |
| `iwp` | Indiscriminate weapons (CBRNE) | 0.04 / 0.08 / 0.2 | = | sentinelle |
| `ssh` | Suicide and self-harm | 0.07 / 0.15 / 0.3 | = | sentinelle; traité par crisis_support |
| `vcr` | Violent crimes | 0.12 / 0.25 / 0.45 | = | sentinelle |
| `src` | Sex-related crimes | 0.12 / 0.25 / 0.45 | = |  |
| `hte` | Hate and discrimination | 0.15 / 0.28 / 0.5 | 0.08 / 0.18 / 0.35 |  |
| `ncr` | Non-violent crimes | 0.15 / 0.3 / 0.5 | = |  |
| `sxc` | Adult sexual content | 0.2 / 0.4 / 0.6 | = |  |
| `prv` | Privacy and personal data | 0.2 / 0.4 / 0.7 | 0.15 / 0.28 / 0.5 | sentinelle; traité par redact |
| `ipv` | Intellectual property | 0.25 / 0.45 / 0.7 | = | output, conversation seulement |
| `dfm` | Defamation | 0.18 / 0.35 / 0.6 | = | output, conversation seulement |
| `spc` | Specialized advice | 0.25 / 0.5 / 0.85 | = | traité par guide; output seulement |
| `elc` | Elections and civic integrity | 0.15 / 0.3 / 0.5 | = |  |
| `pij` | Prompt injection and jailbreak | 0.15 / 0.3 / 0.55 | = | sentinelle; input, conversation seulement |
| `sid` | Sensitive information disclosure | 0.08 / 0.18 / 0.35 | = | sentinelle; output, conversation seulement |
| `exa` | Excessive agency | 0.15 / 0.3 / 0.5 | = | output, conversation seulement |
| `mis` | Misinformation and unsupported claims | 0.25 / 0.45 / 0.8 | = | traité par guide; output seulement |
| `scp` | Out of scope | 0.4 / 0.75 / 0.95 | = | désactivée |

### Les règles, en clair

1. Chaque message et chaque réponse sont notés seuls ; le verdict est la bande la plus forte atteinte.
2. La session retient le risque, pas le texte : le plus élevé entre la moitié du risque précédent et le risque du nouveau verdict.
3. Une session est surveillée si son risque atteint 0,2, pendant deux tours après un constat grave, et tant qu'un message retenu reste dans la fenêtre.
4. En session surveillée, la réponse est relue avec l'historique ; cette lecture ne compte que si elle complète une demande nuisible antérieure.
5. L'historique seul ne relève jamais un verdict.

### Formules

```
V_in(t)   = D(input,  J(q_t))
V_out(t)  = D(output, J(r_t))

W_t       = carry_left > 0  ∨  risk_t ≥ 0.2  ∨  placeholder ∈ H_t          (watched)
V_ctx     = D(output, J(r_t | H_t))                                          (only if W_t)
c, d      = P(reply completes an earlier harmful request), P(user steps away)
A_t       = c ≥ τ  ∧  c ≥ d  ∧  V_ctx has a finding ≥ flag,   τ = 0.5
V(t)      = V_out(t) ⊕ V_ctx  if A_t,  else V_out(t)

risk_t+1  = max(δ · risk_t, ρ(action)),  δ = 0.5,  ρ = (0, 0.25, 0.6, 1.0) for (allow, flag, review, block)
carry     = 2 turns after a conversation verdict ≥ review or any block
```

Code : `V_in` [`check_input`](../python/src/guardrail_chatbot_jev/guard.py#L119), `V_out` [`check_output`](../python/src/guardrail_chatbot_jev/guard.py#L131), `W_t` [`Session.watching`](../python/src/guardrail_chatbot_jev/session.py#L128), `V_ctx`, `c`, `d` [`_check_in_context`](../python/src/guardrail_chatbot_jev/guard.py#L333) / [`context_questions`](../python/src/guardrail_chatbot_jev/questions.py#L135), `A_t`, `⊕` [`_attribute`](../python/src/guardrail_chatbot_jev/guard.py#L350), `risk` [`Session.observe`](../python/src/guardrail_chatbot_jev/session.py#L146), `carry` [`Session.advance`](../python/src/guardrail_chatbot_jev/session.py#L163)

## Calibrage mono-tour

En clair :

1. **Une sentinelle seule est un signal faible.** Quand seule la question oui/non dédiée voit une
   catégorie, et que la question principale lui donne moins de 2 %, la détection est *non corroborée*.
2. **Une détection faible reste à flag**, sauf si elle atteint seule le seuil de blocage ; le plancher
   « jamais sous » de la catégorie ne la relève pas. L'automutilation n'est jamais affaiblie (pack
   Viet Nam : aussi `vsv`, `vld`).
3. **Une réponse qui refuse** (refus ≥ 0,8) avec une sentinelle faible sous 0,5 est notée à flag. Les
   fuites de secrets, de données personnelles et de textes protégés (`sid`, `prv`, `ipv`) font exception,
   car un refus peut encore les contenir.
4. **Les données personnelles vues par la seule sentinelle** sont masquées puis livrées, pas bloquées,
   sauf si la sentinelle atteint 0,8.
5. **Une confiance faible** (sous 0,65) envoie une détection ou un quasi-seuil en revue, sauf si Jev lit
   une intention bénigne avec une confiance d'au moins 0,5.
6. **Une conversation qui ne s'aggrave pas** (escalade ≤ 0,5) est plafonnée à flag, sauf `cse` et `ssh`.

```
u_k                 = finding from the sentinel only  ∧  choice_k < 0.02      (uncorroborated)
never_below         applied only if ¬u_k ∨ p ≥ θ_block
u_k ∧ p < θ_block                                                    → at most flag
output ∧ refusal ≥ 0.8 ∧ u_k ∧ p < 0.5 ∧ k ∉ {sid, prv, ipv}         → flag
u_k ∧ route_k = redact ∧ action = block ∧ p < 0.8                    → review (masked)
confidence gate:  conf < 0.65 ∧ (finding ∨ p ≥ θ_flag / 2) → review,  unless intent = benign ∧ conf ≥ 0.5
conversation:     escalation ≤ 0.5 → at most flag,  except cse, ssh
```

Code : `u_k` [`decide`](../python/src/guardrail_chatbot_jev/decide.py#L57), `never_below` [`_finding`](../python/src/guardrail_chatbot_jev/decide.py#L212), weak [`decide`](../python/src/guardrail_chatbot_jev/decide.py#L70), refusal [`_cap_uncorroborated_on_refusal`](../python/src/guardrail_chatbot_jev/decide.py#L245), redact [`_redact_instead_of_block`](../python/src/guardrail_chatbot_jev/decide.py#L270), gate [`_confidence_gate`](../python/src/guardrail_chatbot_jev/decide.py#L378), conversation [`no-escalation-caps-conversation`](../policies/standard-v1.json#L700), settings [`sentinel_corroboration`](../policies/standard-v1.json#L23) / [`confidence_gate`](../policies/standard-v1.json#L32), [`spc`](../policies/standard-v1.json#L328), [`ncr`](../policies/standard-v1.json#L207), [`iwp`](../policies/standard-v1.json#L83)

## Revue en temps réel

`ReviewHandling: ReviewAsAudit` ([`review_handling`](../python/src/guardrail_chatbot_jev/guard.py#L107), [`_audit`](../python/src/guardrail_chatbot_jev/guard.py#L314)) : seul `block` arrête le contenu ; `review` délivre et place en audit
prioritaire, `flag` en audit par échantillonnage ; un verdict dégradé sur une surface fail-closed reste
retenu.

## Résultats

| Mesure en direct, 223 conversations | Plancher (ancien) | Attribution (actuel) |
| --- | --- | --- |
| Messages inoffensifs retenus | 171 / 194 | **0 / 194** |
| Réponses nuisibles détectées | 19 / 19 | **19 / 19** |
| Escalades signalées | 9 / 9 | **9 / 9** |
| Conversations inoffensives en file de revue | 172 / 194 | **2 / 194** |

| Rejeu, ≈ 5 000 réponses enregistrées | Inoffensifs retenus | Violations détectées |
| --- | --- | --- |
| Avant calibrage final | 0,62 % | 100 % |
| Après calibrage final | **0,04 %** | **100 %** |
| Temps réel (`ReviewAsAudit`) | **0,02 %** arrêtés | toutes les réponses nuisibles arrêtées |

Jeux de données, protocoles, ablation, bruit, régression et reproduction : [version anglaise](multiturn.md).
