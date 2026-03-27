# Partie 4-5-6: Pitch + Deep Dive Technique

## Objectif du document

Ce document est fait pour TA partie de presentation:

- 4. Connaissances du cours appliquees
- 5. Outils non couverts par le cours
- 6. Comment la solution s'evalue

Tu as 3 niveaux d'usage:

1. Version orale simple (si stress)
2. Version technique (si jury creuse)
3. Mapping direct vers le code (pour prouver)

---

## A) Pitch oral pret a dire (environ 4 min)

### A1. Connaissances du cours appliquees (1 min 30)

Sur cette partie, on a applique les concepts du cours de facon tres operationnelle.

Premier concept: decomposition agentique.
On a decoupe la solution en blocs specialises: extraction, recherche API, scoring, recommendation et evaluation.
Ce choix rend le systeme plus lisible et plus facile a debugger qu'un agent unique qui ferait tout.

Deuxieme concept: prompt engineering structure.
On impose des sorties JSON au LLM au lieu de texte libre.
C'est essentiel pour brancher ensuite des controles automatiques.

Troisieme concept: grounding factuel.
On ne fait pas confiance aveuglement au LLM.
La recommandation est confrontee a la source de verite externe via l'ID produit.

Quatrieme concept: approche hybride.
Le deterministic gere le controle metier et le ranking,
le generatif gere la comprehension langage naturel et l'explication utilisateur.

Cinquieme concept: evaluation continue.
Chaque interaction est tracee et transformee en KPI.
On pilote donc la qualite par la mesure.

### A2. Outils hors cours utilises (1 min)

On a utilise des outils externes pour operationaliser les concepts:

- Gradio pour exposer le systeme et le dashboard en direct.
- DummyJSON comme catalogue public reproductible.
- Matplotlib pour les tendances temporelles.
- Journalisation JSON custom pour garder les traces complete.

Ces outils ne changent pas la logique scientifique,
ils la rendent demonstrable, observable et auditable.

### A3. Evaluation de la solution (1 min 30)

L'evaluation se fait sur deux boucles.

Online:
a chaque requete, on sauvegarde contraintes, resultats API, ranking, recommendation, et provenance.
Le dashboard lit ensuite ces traces toutes les 5 secondes et recalcule les KPI.

Offline:
un benchmark rejoue un dataset de requetes fixes,
calcule des metriques de qualite,
et exporte un rapport detaille.

La metrique cle est provenance_ok.
Regle simple: l'ID recommande doit appartenir aux IDs retournes par l'API.
Si ce n'est pas le cas, on signale un risque d'hallucination.

Conclusion de cette partie:
notre evaluation n'est pas decorative,
elle est integree au coeur du pipeline.

---

## B) Version technique (si le jury demande des details)

## B1. Connaissances appliquees avec preuves dans le code

1. Decomposition agentique

- preference_agent.py: extraction contraintes.
- api_search_agent.py: acquisition source de verite.
- search_agent.py: filtres et scoring explicables.
- recommendation_agent.py: choix final et justification.
- gui.py + analytics_dashboard.py + benchmark_dummyjson.py: evaluation/observabilite.

2. Prompt engineering structure

- extraction: contrainte JSON a 3 champs.
- recommendation structuree: JSON avec best_product_id.

3. Grounding

- verification de provenance par appartenance d'ID.

4. Hybridation deterministic/generatif

- deterministic: filtres budget/tags/score.
- generatif: interpretation du besoin et formulation.

5. Evaluation continue

- ecriture trace par requete,
- dashboard auto-refresh,
- benchmark batch dataset.

## B2. Outils hors cours et justification technique

1. Gradio

- avantage: prototypage rapide, interaction + monitoring sans backend lourd.

2. DummyJSON

- avantage: API publique stable, facilite reproduction des resultats.

3. Matplotlib

- avantage: vue tendance, pas seulement snapshot.

4. JSON logging custom

- avantage: audit complet par requete,
- utile pour debug et post-mortem.

## B3. Evaluation: logique exacte des metriques

1. provenance_rate

- definition: proportion de requetes avec provenance_ok = True.
- interpretation: robustesse factuelle du systeme.

2. budget_adherence

- definition operationnelle: prix recommande <= budget \* 1.3.
- interpretation: adequation economique, avec tolerance.

3. api_coverage

- definition: proportion de requetes qui retournent au moins un produit API.
- interpretation: capacite du backend de recherche a couvrir les intentions.

4. offline metrics

- category_match_rate,
- budget_respected_rate,
- provenance_ok_rate,
- with_at_least_one_candidate.

---

## C) Mapping code a montrer en live (si besoin)

1. Extraction contrainte

- Montrer extract_constraints dans preference_agent.py.
- Phrase utile: "ici on force un JSON machine-readable".

2. Source de verite API

- Montrer search_products_api dans api_search_agent.py.
- Phrase utile: "on choisit endpoint categorie ou search selon le besoin".

3. Ranking local

- Montrer search_products dans search_agent.py.
- Phrase utile: "le score est explicable, ce n'est pas une boite noire".

4. Recommendation structuree

- Montrer build_recommendation_structured dans recommendation_agent.py.
- Phrase utile: "on exige un best_product_id pour verification automatique".

5. Benchmark online

- Montrer record_to_benchmark dans gui.py.
- Phrase utile: "chaque requete devient une trace complete".

6. Dashboard KPI

- Montrer calculate_metrics et get_trend_data dans analytics_dashboard.py.
- Phrase utile: "on transforme les traces en indicateurs suivables dans le temps".

7. Benchmark offline

- Montrer run_benchmark dans benchmark_dummyjson.py.
- Phrase utile: "on valide la qualite sur un dataset stable et rejouable".

---

## D) Reponses courtes aux questions difficiles

Question: Pourquoi une tolerance budget de 30%?
Reponse: Pour eviter un filtre trop dur et conserver des candidats proches du budget, tout en gardant une limite explicite.

Question: Votre evaluation prouve-t-elle zero hallucination?
Reponse: Non, elle ne prouve pas zero hallucination au sens absolu, mais elle detecte automatiquement le cas critique: recommendation hors source API.

Question: Pourquoi ne pas tout faire dans le LLM?
Reponse: Parce qu'on perdrait la maitrise metrique et la tracabilite. Le deterministic apporte controle et reproductibilite.

Question: Qu'est-ce qui est le plus important dans votre architecture?
Reponse: Le couple grounding + evaluation continue. C'est ce qui transforme une demo LLM en systeme auditable.

---

## E) Version 30 secondes (ultra resume)

On applique les concepts du cours avec une architecture multi-agent,
un LLM contraint en JSON,
une verification de verite sur source API,
et une evaluation continue online et offline.

Les outils externes (Gradio, DummyJSON, matplotlib) servent a rendre cette logique demonstrable et mesurable.

Notre point fort: chaque recommendation est tracee, verifiee et convertie en KPI.

---

## F) Benchmark explique a fond (niveau debutant total)

## F1. Question centrale du benchmark

Le benchmark sert a repondre a 3 questions critiques:

1. Est-ce que la recommendation finale est basee sur de vrais produits API?
2. Est-ce que la recommendation respecte globalement le budget/categorie?
3. Est-ce qu'on peut mesurer ces points de facon reproductible?

En clair: le benchmark transforme une demo "ca a l'air bien" en preuve mesurable.

## F2. Ce que le benchmark verifie exactement dans ce projet

### Verification A: ancrage API (anti-invention)

- Donnees connues: la liste api_products retournee par DummyJSON.
- Sortie LLM: best_product_id.
- Regle: provenance_ok = (best_product_id appartient aux ids de api_products).

Interpretation:

- True: la recommendation est ancree dans les resultats API.
- False: la recommendation est non verifiable / hallucinee.

### Verification B: adherence budget

- On compare le prix recommande au budget utilisateur.
- Dans le dashboard online, la regle est prix <= budget \* 1.3.
- Dans le benchmark offline (dataset), la regle est intervalle attendu [min, max] par cas.

### Verification C: coherence categorie (offline)

- On compare la categorie du meilleur candidat au expected_category du dataset.

## F3. Flux exact du benchmark online (GUI)

1. L'utilisateur envoie une requete dans l'interface.
2. Le systeme extrait les contraintes (product, budget, usage).
3. Le systeme appelle l'API DummyJSON et recupere api_products.
4. Le systeme applique le scoring local et produit ranked_products.
5. Le LLM fournit une recommendation structuree avec best_product_id.
6. La fonction record_to_benchmark ecrit une trace complete dans benchmark_results.json.
7. Le dashboard relit ce fichier (polling) et recalcule les KPI.

Important:
le benchmark online ne stocke pas juste une note finale, il stocke tout le chemin de decision.

## F4. Flux exact du benchmark offline (dataset)

1. Charger dataset_dummyjson.json.
2. Pour chaque exemple:
   - extraire contraintes,
   - appeler API,
   - scorer,
   - demander recommendation structuree,
   - verifier provenance/categorie/budget,
   - stocker resultat detaille.
3. Calculer les taux globaux.
4. Exporter un rapport JSON complet.

Resultat:
on peut rejouer les memes tests et comparer des versions du systeme.

## F5. Ce que le benchmark PROUVE et ce qu'il NE PROUVE PAS

### Ce qu'il prouve bien

1. La recommendation finale cite bien (ou non) un produit present dans la source API.
2. La logique de classement respecte des contraintes explicites (budget, tags, score).
3. La qualite est mesurable dans le temps (dashboard) et sur dataset fixe (offline).

### Ce qu'il ne prouve pas totalement

1. Il ne prouve pas "zero hallucination semantique".
   Exemple: id valide API, mais justification texte un peu exageree.
2. Il ne prouve pas que "le LLM a lui-meme appele l'API".
   Dans cette architecture, c'est le code Python qui appelle l'API, puis passe les resultats au LLM.
3. Il ne garantit pas une couverture metier parfaite, car dataset encore petit.

Message a dire au jury:
"Notre benchmark prouve l'ancrage factuel de la recommendation via l'ID API, mais on reste transparents sur les limites: ce n'est pas une preuve mathematique de zero hallucination semantique."

## F6. Reponse precise a "est-ce qu'on teste l'utilisation des APIs et outils souhaites?"

Oui, mais avec un niveau de preuve specifique:

1. Utilisation API DummyJSON:

- Verifiee indirectement car on stocke api_products a chaque run.
- Sans reponse API, pas d'IDs de reference pour provenance_ok.

2. Utilisation des outils internes de pipeline:

- Verifiee par la trace complete (constraints, api_products, ranked_products, recommendation).
- On peut auditer chaque etape apres execution.

3. Anti-invention:

- Verifiee par provenance_ok (ID recommande present dans api_products).

Formulation claire:
"On ne demande pas au LLM d'inventer ou d'aller chercher seul. On lui fournit des candidats API et on valide son choix par verification d'ID."

## F7. Explication code par code (ultra simple)

### 1) preference_agent.py

- Role: comprendre le texte utilisateur.
- Sortie: UserConstraints.
- Si LLM echoue: fallback regex.

Ce que tu dis:
"Ici, on transforme une phrase libre en donnees structurees pour le reste du pipeline."

### 2) api_search_agent.py

- Role: interroger DummyJSON.
- Fait: detecte categorie ou lance une recherche texte.
- Renvoie une liste standardisee de produits.

Ce que tu dis:
"Ici, on construit notre source de verite."

### 3) search_agent.py

- Role: filtrer et scorer localement.
- Budget, tags, mots-clés, bonus prix.
- Produit ranked_products.

Ce que tu dis:
"Ici, on garde une logique explicable et reproductible."

### 4) recommendation_agent.py

- Role: choisir le meilleur produit et expliquer.
- Mode structure: impose best_product_id.

Ce que tu dis:
"Ici, on force un format qui permet la verification automatique."

### 5) gui.py

- Role: execution online + enregistrement benchmark.
- record_to_benchmark ecrit la trace complete.

Ce que tu dis:
"Ici, chaque interaction devient une evidence exploitable."

### 6) analytics_dashboard.py

- Role: calcul KPI et tendances.
- Lit benchmark_results.json en continu.

Ce que tu dis:
"Ici, on transforme les traces en pilotage qualite."

### 7) benchmark_dummyjson.py

- Role: benchmark offline reproductible.
- Rejoue les exemples, calcule les taux, exporte rapport.

Ce que tu dis:
"Ici, on sort du mode demo et on passe en mode evaluation systematique."

---

## G) Script oral mot a mot (ta partie 4-5-6)

Tu peux lire ce bloc exactement tel quel.

"Je prends maintenant les points 4, 5 et 6: connaissances appliquees, outils hors cours, et evaluation.

Sur les connaissances du cours, on a applique cinq principes.
Premier principe: decomposition agentique.
Au lieu d'un seul agent monolithique, on a separe extraction, appel API, scoring local, recommendation et evaluation.
Ce choix rend le systeme lisible et surtout debuggable.

Deuxieme principe: prompt engineering structure.
On impose des sorties JSON au LLM.
Ce n'est pas du confort, c'est ce qui permet des controles automatiques ensuite.

Troisieme principe: grounding factuel.
La recommendation n'est pas acceptee aveuglement.
On la verifie contre la source DummyJSON avec la regle d'appartenance d'ID.

Quatrieme principe: approche hybride.
Le deterministic gere les regles metier et le ranking.
Le generatif sert a comprendre la requete et expliquer la decision.

Cinquieme principe: evaluation continue.
Chaque execution est tracee, puis transformee en KPI.

Sur les outils hors cours,
on a utilise Gradio pour l'interface et le dashboard,
DummyJSON comme source publique reproductible,
matplotlib pour les tendances,
et une journalisation JSON custom pour l'audit.

Maintenant, comment la solution s'evalue.
On a deux boucles.

Premiere boucle, online:
a chaque requete, on stocke contraintes, produits API, ranking, recommendation et statut de provenance.
Le dashboard relit ces traces toutes les 5 secondes et recalcule les KPI.

Deuxieme boucle, offline:
on rejoue un dataset fixe,
on calcule category match, budget respected et provenance,
et on exporte un rapport complet.

Je precise ici un point important techniquement: oui, le benchmark passe bien par un dataset.
Ce dataset est un fichier JSON avec plusieurs exemples de test.
Chaque exemple contient une requete utilisateur, une categorie attendue, et une plage de prix attendue.
Cela veut dire qu'on ne fait pas une evaluation floue ou impressionniste.
On rejoue des cas fixes, donc comparables et reproductibles.

Le script de benchmark charge d'abord ce dataset,
puis pour chaque exemple il relance exactement le vrai pipeline:
extraction des contraintes,
appel de l'API DummyJSON,
filtrage et scoring locaux,
recommendation structuree du LLM,
puis verification.

Ensuite, il compare plusieurs choses.
Premiere comparaison: est-ce qu'on obtient au moins un candidat.
Deuxieme comparaison: est-ce que la categorie du meilleur candidat correspond a la categorie attendue.
Troisieme comparaison: est-ce que le prix du meilleur candidat tombe bien dans l'intervalle attendu du dataset.
Quatrieme comparaison, la plus importante: est-ce que l'identifiant recommande par le LLM appartient bien aux identifiants des produits reellement retournes par l'API.

La metrique cle est provenance_ok.
La regle est simple: l'ID recommande doit etre present dans les IDs retournes par l'API.
Si ce n'est pas le cas, on detecte une recommendation non verifiable.

Autrement dit, on teste bien que le systeme s'appuie sur la source API comme source de verite,
et qu'il n'invente pas un produit hors de cette source.

Il faut aussi etre rigoureux sur ce que cela prouve exactement.
Ce benchmark prouve tres bien l'ancrage factuel par identifiant produit.
En revanche, il ne prouve pas mathematiquement zero hallucination semantique dans le texte explicatif du LLM.
Il ne prouve pas non plus que le LLM a lui-meme appele l'API,
car dans cette architecture c'est le code Python qui appelle DummyJSON, puis transmet les candidats au modele.
Donc ce qu'on valide reellement, c'est que le systeme global utilise bien l'API comme source de verite et que le modele ne sort pas de cette source.

Point important de rigueur:
ce benchmark prouve bien l'ancrage factuel par ID,
mais il ne prouve pas mathematiquement zero hallucination semantique.
On est donc solides sur ce qu'on mesure, et transparents sur les limites.

Si je relie maintenant cela au code,
preference_agent.py extrait les contraintes sous forme structuree,
api_search_agent.py appelle l'API DummyJSON et construit les api_products,
search_agent.py applique les filtres et calcule le score local,
recommendation_agent.py force une sortie JSON avec best_product_id,
gui.py enregistre toute la trace online dans benchmark_results.json,
analytics_dashboard.py relit cette trace et recalcule les KPI,
et benchmark_dummyjson.py rejoue le pipeline complet sur le dataset pour mesurer category match, budget respected et provenance.

En conclusion,
notre systeme ne se contente pas de recommander,
il recommande de facon auditable, avec preuve de provenance et suivi de qualite."

---

## H) Mini version mot a mot (60 secondes)

"Sur ma partie, l'essentiel est simple.
On applique les concepts du cours avec une architecture multi-agent, des sorties JSON contraintes, et une verification de verite sur API.

Les outils hors cours, comme Gradio, DummyJSON et matplotlib, servent a rendre la solution demonstrable et mesurable.

L'evaluation combine online et offline.
Online, chaque requete est tracee et convertie en KPI.
Offline, un benchmark rejoue un dataset fixe et calcule des taux comparables.

La metrique critique est provenance_ok:
l'ID recommande doit etre dans les IDs API.
Donc on mesure concretement le risque d'invention, et pas seulement la qualite percue."
