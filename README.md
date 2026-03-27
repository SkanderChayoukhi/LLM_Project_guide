# Guide Complet du Projet Multi-Agent Shopping Assistant

## 1) Objectif de ce document

Ce document explique le projet comme si tu partais de zero:

- ce que le systeme cherche a resoudre,
- comment il est structure,
- comment chaque fichier Python fonctionne,
- comment les donnees circulent,
- comment l'evaluation est faite,
- et comment presenter tout cela en 10-12 minutes.

Le but est de te rendre autonome sur le code et sur la presentation.

---

## 2) Vision globale du projet

Le projet implemente un assistant d'achat en ligne base sur un pipeline multi-agent.

En entree:

- une phrase libre de l'utilisateur, par exemple: "i want a floral perfume with a 60 euros budget"

En sortie:

- des contraintes structurees (produit, budget, usage),
- une liste de produits recuperes depuis une API publique (DummyJSON),
- un classement local des produits (scoring),
- une recommandation finale generee par LLM,
- un enregistrement de benchmark avec verification de provenance.

Idee centrale:

- on laisse le LLM faire ce qu'il fait bien (comprendre du langage naturel et argumenter),
- on garde le controle sur les donnees factuelles via l'API et les regles de scoring,
- on mesure la fiabilite en verifiant que la recommandation existe bien dans les resultats API.

---

## 3) Probleme concret que le projet resout

Probleme metier:

- un utilisateur exprime un besoin de facon floue,
- il faut transformer ce besoin en requete exploitable,
- trouver des produits reels,
- eviter les hallucinations de LLM,
- et donner une recommandation argumentee.

Sous-problemes techniques:

1. Extraction d'information depuis texte libre.
2. Recherche produit dans un catalogue.
3. Filtrage/scoring selon budget et pertinence.
4. Recommandation finale lisible.
5. Evaluation continue de la qualite.

Risque principal traite:

- Hallucination: le LLM peut inventer un produit non present dans les donnees.

Mecanisme de mitigation:

- verifier que best_product_id choisi par le LLM appartient bien aux IDs retournes par l'API.

---

## 4) Architecture du systeme

Pipeline fonctionnel:

1. Interface (GUI ou CLI) recoit la requete utilisateur.
2. Agent de preferences extrait des contraintes structurees.
3. Agent de recherche API recupere les produits DummyJSON.
4. Agent de recherche locale filtre + score.
5. Agent de recommandation LLM choisit le meilleur produit.
6. Systeme de benchmark enregistre tout.
7. Dashboard analytics lit les resultats et affiche les KPIs.

Decoupage en modules:

- constraints.py: modele de donnees utilisateur.
- preference_agent.py: extraction NLP -> contraintes.
- api_search_agent.py: interaction HTTP avec DummyJSON.
- search_agent.py: logique locale de filtrage et scoring.
- recommendation_agent.py: production de recommandation (texte ou JSON structure).
- gui.py: application Gradio principale + benchmark temps reel.
- analytics_dashboard.py: visualisation des metriques.
- run_all.py: lance GUI + dashboard en parallele.
- benchmark_dummyjson.py: benchmark batch sur dataset de tests.

---

## 5) Explication detaillee fichier par fichier

## 5.1 constraints.py

Role:

- definir la structure commune des contraintes utilisateur.

Contenu principal:

- dataclass UserConstraints avec:
  - product: str (obligatoire)
  - budget: Optional[float]
  - usage: Optional[str]

Pourquoi c'est important:

- standardise la communication entre agents,
- evite les dictionnaires ad hoc,
- simplifie le typage et la maintenance.

---

## 5.2 preference_agent.py

Role:

- transformer la phrase utilisateur en JSON simple exploitable.

Fonctions:

1. \_fallback_parse(text)

- parseur regex de secours,
- extrait le premier nombre comme budget,
- met product=text brut,
- usage=None.

Interet:

- resilence. Si le LLM echoue (filtre de contenu, JSON invalide, erreur reseau), le pipeline ne plante pas.

2. extract_constraints(text, model)

- construit un SystemMessage qui impose:
  - format JSON strict,
  - cles attendues: product, budget, usage,
  - extraction en anglais,
  - pas de texte hors JSON.
- envoie aussi un HumanMessage avec la requete.
- parse json.loads(response.content).
- retourne UserConstraints.
- en cas d'exception: fallback regex.

Points forts:

- prompt structure et contraint,
- fallback robuste.

Limites:

- except tres large (broad exception) masque la nature fine des erreurs,
- absence de validation stricte du schema (types, bornes).

---

## 5.3 api_search_agent.py

Role:

- obtenir les produits depuis DummyJSON.

Constantes:

- CATEGORY_KEYWORDS: associe des mots-clés a des categories DummyJSON.

Fonctions:

1. \_detect_category_from_product(product)

- tokenise en mots minuscules,
- detecte une categorie si intersection entre mots utilisateur et dictionnaire de mots-clés.

2. build_api_query(constraints)

- prend constraints.product,
- garde seulement les 1-2 premiers mots,
- objectif: eviter requete trop specifique pour un moteur de recherche simple.

3. search_products_api(constraints, max_results=5)

- strategie hybride:
  - si categorie detectee: endpoint categorie /products/category/{category},
  - sinon endpoint recherche /products/search?q=...
- timeout=10s,
- gestion des RequestException -> retourne liste vide,
- normalise les produits avec schema local stable:
  - id, title, description, category, price(float), rating, brand, tags, thumbnail, images.

Pourquoi c'est important:

- separe source de verite externe et logique locale,
- normalise les champs pour les modules suivants.

Limites:

- mapping categories statique et incomplet,
- pas de retries ni backoff,
- pas de cache API.

---

## 5.4 search_agent.py

Role:

- appliquer la logique metier locale de filtrage/scoring.

Constantes:

- CATEGORY_HINTS: mot -> categorie preferee.
- TAG_MAPPING: mot-clé utilisateur -> tag exact DummyJSON.

Fonction cle: search_products(catalogue, c)

Etapes detaillees:

1. Preparation requete

- query_product = c.product.lower()
- query_usage = c.usage.lower()
- split en mots.

2. Detection de tags cibles

- parcourt tous les mots de product + usage,
- si mot present dans TAG_MAPPING -> ajoute tag cible.

3. Parcours catalogue
   Pour chaque item:

- recupere title/brand/description/category/tags en minuscules,
- convertit price en float.

4. Filtre budget

- si budget existe et price > budget \* 1.3 -> exclu.
- cette marge 30% rend le systeme moins strict.

5. Filtre tag strict

- si target_tags non vide,
- il faut au moins un tag cible present,
- sinon item exclu.

6. Scoring

- +2 si mot produit retrouve dans title/brand/description/tags,
- +3 bonus si categorie correspond a CATEGORY_HINTS,
- +1 si mot usage retrouve dans title/description,
- bonus prix si budget:
  - max(0, int((budget - price)/10)).

7. Resultat

- copie item + champ \_score,
- tri decroissant sur \_score,
- retourne liste candidate.

Atouts:

- explicable,
- deterministic,
- transparent pour benchmark.

Limites:

- matching lexical simple (pas semantique),
- dependance forte a l'anglais et aux mappings,
- poids de score fixes non appris.

---

## 5.5 recommendation_agent.py

Role:

- produire la recommandation finale via LLM.

Deux modes:

1. build_recommendation(...)

- retour texte libre en francais,
- utilise meilleur candidat + 1-2 alternatives,
- prompt explicatif.

2. build_recommendation_structured(...)

- mode critique pour verification/benchmark,
- exige JSON strict avec:
  - best_product_id,
  - best_product_title,
  - best_product_price,
  - alternative_product_ids,
  - explanation.
- impose que l'id soit pris dans la liste fournie.
- parse json.loads,
- fallback parse_error si sortie non-JSON.

Pourquoi c'est central:

- ce mode structure rend possible la verification automatique de provenance.

Limites:

- pas de schema validator (ex: pydantic/jsonschema),
- pas de garde si id present mais incoherent avec titre/prix.

---

## 5.6 main.py

Role:

- point d'entree CLI simple.

Fonctions:

1. get_llm()

- initialise ChatOpenAI avec variables d'environnement:
  - AI_MODEL,
  - AI_ENDPOINT,
  - AI_API_KEY,
  - temperature=0.1.

2. main()

- lit la phrase utilisateur,
- extrait contraintes,
- appelle API search,
- applique scoring local,
- affiche candidats,
- genere recommandation texte.

Interet:

- utile pour debug rapide hors interface web.

---

## 5.7 gui.py

Role:

- interface principale utilisateur + collecte benchmark en temps reel.

Elements majeurs:

1. Initialisation

- load_dotenv(),
- MODEL singleton via get_llm(),
- chemin benchmark_results.json.

2. record_to_benchmark(...)

- construit une entree complete avec:
  - timestamp,
  - requete utilisateur,
  - contraintes,
  - produits API,
  - produits classes,
  - recommandation,
  - provenance_ok.
- provenance_ok = best_product_id present dans IDs API.
- lit fichier existant (si vieux format dict, reset vers liste),
- append puis ecrit JSON.

3. run_assistant(user_text)

- controle entree vide,
- extraction contraintes,
- recuperation API (max_results=50),
- scoring local,
- generation HTML contraintes,
- generation cartes produits HTML (top 6),
- appel recommendation structuree,
- rendu recommendation HTML,
- enregistrement benchmark,
- retourne trois blocs HTML.

4. Construction UI Gradio

- textbox + bouton Search,
- zones HTML contraintes/candidats/recommandation,
- events click/submit relies a run_assistant,
- theme + CSS custom.

Notes UX:

- style visuel travaille (gradient, cards, ranking, thumbnails),
- reactivite simple sans etat conversationnel persistant.

Limites:

- HTML inline important (maintenabilite reduite),
- logique metier et logique presentation melangees dans run_assistant,
- gestion erreurs I/O basique.

---

## 5.8 analytics_dashboard.py

Role:

- tableau de bord de monitoring du benchmark.

Composants:

1. DashboardState

- cache des donnees benchmark,
- lock thread pour eviter races,
- reload seulement si fichier modifie.

2. calculate_metrics(data)
   Calcule:

- total_queries,
- provenance_rate,
- budget_adherence,
- api_coverage.

Detaillons budget_adherence:

- considere seulement les requetes avec budget,
- retrouve prix du produit recommande (best_product_id) dans api_products,
- compte conforme si prix <= budget \* 1.3.

3. get_trend_data(data)

- trie par timestamp,
- calcule tendances cumulatives:
  - provenance rate cumulatif,
  - budget adherence cumulative.

4. create_trend_chart(data)

- matplotlib,
- echantillonnage points pour lisibilite,
- courbes provenance vs budget.

5. create_metrics_summary_table(metrics)

- table HTML avec status icones (OK/Warning/Fail) selon seuils.

6. create_results_table(data)

- table des 10 dernieres requetes.

7. create_dashboard()

- compose interface Gradio dashboard,
- Timer 5 secondes pour auto-refresh,
- bouton manual refresh.

Atouts:

- monitoring temps reel,
- visibilite immediate de la qualite.

Limites:

- metriques simples (pas latence, pas precision@k),
- pas de pagination ni export direct depuis dashboard.

---

## 5.9 run_all.py

Role:

- lancer GUI (7860) et dashboard (7861) ensemble.

Logique:

- subprocess.Popen de gui.py,
- wait 2s,
- subprocess.Popen de analytics_dashboard.py,
- boucle infinie pour garder les process vivants,
- Ctrl+C -> terminate/kill proprement.

Interet:

- tres pratique en demo.

Limites:

- pas de health checks de ports,
- import os non utilise,
- depend de sleeps fixes.

---

## 5.10 benchmark_dummyjson.py

Role:

- evaluation hors ligne sur dataset.

Structures:

- BenchmarkExample: cas de test attendu.
- BenchmarkResult: trace complete d'un run.

Pipeline benchmark:

1. load_dataset() lit dataset_dummyjson.json.
2. run_benchmark():

- initialise modele,
- boucle sur exemples,
- extract_constraints,
- search_products_api,
- search_products,
- build_recommendation_structured,
- checks:
  - provenance_ok,
  - category_match,
  - budget_respected,
- sauvegarde resultats detailles.

3. resume final + ecriture JSON.

Metriques rapportees:

- taux de cas avec candidat,
- taux category match,
- taux budget respected,
- taux provenance ok.

Atout majeur:

- traçabilite complete par exemple.

---

## 6) Donnees et fichiers non-Python importants

1. requirements.txt

- dependances coeur:
  - langchain-openai,
  - langchain-core,
  - langchain,
  - langchain-azure-ai,
  - langchain-mcp-adapters,
  - python-dotenv,
  - requests,
  - gradio,
  - matplotlib,
  - numpy.

2. dataset_dummyjson.json

- jeu de test avec 6 exemples,
- chaque exemple contient:
  - query,
  - categorie attendue,
  - intervalle prix attendu.

3. benchmark_results.json

- historique des executions GUI,
- format liste d'entrees tracees,
- source du dashboard.

4. README.md

- documentation existante,
- deja riche mais plus orientee vue d'ensemble que lecture fine du code.

---

## 7) Flux de donnees de bout en bout

Soit une requete utilisateur U.

Etape A: NLP -> contraintes

- C = extract_constraints(U)
- C = {product, budget, usage}

Etape B: recherche API

- P_api = search_products_api(C)
- P_api est la source de verite externe.

Etape C: ranking local

- P_ranked = search_products(P_api, C)
- ajoute \_score et trie.

Etape D: recommandation structuree

- R = build_recommendation_structured(C, P_ranked)
- R contient best_product_id.

Etape E: verification provenance

- provenance_ok = (R.best_product_id in {ids de P_api})

Etape F: persistance benchmark

- ecriture dans benchmark_results.json.

Etape G: visualisation

- dashboard recharge fichier et met a jour KPIs/charts.

---

## 8) Reponses directes aux 5 questions de presentation

## Q1. Which problem did you try to solve?

Nous avons voulu resoudre le probleme de recommandation produit fiable a partir de requetes naturelles.

Difficulte principale:

- un LLM seul peut bien reformuler, mais peut halluciner.

Donc objectif final:

- combiner intelligence linguistique (LLM) + controle de verite (API + verification d'ID).

## Q2. How did you structure the solution?

Structure en pipeline multi-agent specialise:

1. extraction de contraintes,
2. recherche API,
3. filtrage/scoring local,
4. recommandation LLM,
5. verification provenance,
6. benchmarking temps reel + dashboard.

Chaque module a une responsabilite claire.

## Q3. Which knowledge from the course did you apply?

Connaissances appliquees:

- decomposition agentique (separation des roles),
- prompt engineering structure (JSON-only, consignes explicites),
- grounding des sorties sur donnees externes,
- evaluation par metriques quantitatives,
- observabilite de pipeline (traces stockees et dashboard).

## Q4. Did you need resources/tools not covered in the course?

Oui:

- Gradio pour l'interface et le dashboard web,
- DummyJSON comme source externe de catalogue,
- matplotlib pour visualiser les tendances,
- mecanisme custom de benchmark continu.

## Q5. How does your solution evaluate?

Evaluation sur deux axes:

1. En ligne (temps reel):

- chaque requete est enregistree,
- dashboard affiche provenance_rate, budget_adherence, api_coverage.

2. Hors ligne (dataset):

- benchmark_dummyjson.py execute plusieurs cas,
- calcule category_match, budget_respected, provenance_ok,
- produit un rapport JSON detaille.

Point cle:

- la metrique de provenance teste directement le risque d'hallucination.

---

## 9) Analyse critique (forces et limites)

Forces:

- architecture lisible et modulaire,
- verification de provenance explicite,
- benchmark + dashboard = vraie observabilite,
- fallback extraction pour robustesse.

Limites:

- dependance a des mappings mots-cles manuels,
- scoring heuristique non appris,
- peu de garde-fous schema pour JSON LLM,
- internationalisation partielle (prompt EN, interface FR/EN mixte),
- dataset de benchmark encore petit.

---

## 10) Propositions d'amelioration prioritaires

1. Validation stricte de sortie LLM

- utiliser un schema formel (pydantic/jsonschema),
- rejeter/corriger sorties invalides automatiquement.

2. Scoring plus semantique

- embeddings pour matching intention-produit,
- reranking hybride lexical + semantique.

3. Evaluation plus riche

- ajouter precision@k, recall@k, taux "no recommendation",
- suivre temps de reponse et erreurs API.

4. Qualite logicielle

- tests unitaires par agent,
- tests d'integration pipeline,
- separer generation HTML et logique metier.

5. Robustesse infra

- retries API avec backoff,
- cache intelligent,
- health checks au lancement multi-process.

---

## 11) Script de presentation (10-12 minutes)

## Slide 1 (1 min) - Probleme

- "Un utilisateur decrit un besoin en langage naturel."
- "Le risque: une recommandation plausible mais fausse (hallucination)."
- "Notre objectif: recommandation utile ET verifiable."

## Slide 2 (2 min) - Architecture

- montrer pipeline 6-7 etapes,
- insister sur separation LLM vs logique deterministe.

## Slide 3 (2 min) - Deep dive extraction + recherche

- preference_agent: JSON constraints + fallback,
- api_search_agent: categorie vs recherche texte.

## Slide 4 (2 min) - Deep dive scoring + reco

- search_agent: filtres + score,
- recommendation_agent: mode structure + id obligatoire.

## Slide 5 (2 min) - Evaluation

- provenance metric,
- dashboard temps reel,
- benchmark dataset hors ligne.

## Slide 6 (1-2 min) - Limites et roadmap

- limites actuelles,
- 3 ameliorations prioritaires.

Conclusion (30s)

- "Le systeme est deja exploitable en demo et surtout auditable."
- "La prochaine etape est d'augmenter la precision semantique et la robustesse formelle."

---

## 12) Commandes utiles pour demo

Lancer GUI + dashboard:

- python code/run_all.py

Lancer seulement GUI:

- python code/gui.py

Lancer seulement dashboard:

- python code/analytics_dashboard.py

Lancer benchmark dataset:

- python benchmark_dummyjson.py

---

## 13) Resume ultra-court

Ce projet met en place un assistant d'achat multi-agent qui:

- comprend une demande libre,
- cherche des produits reels,
- classe les candidats,
- genere une recommandation,
- et verifie automatiquement que la recommandation est bien ancree dans la source de donnees.

La valeur cle du projet est la combinaison:

- utilite produit (UX + recommandation),
- et fiabilite mesurable (provenance + benchmark + dashboard).
