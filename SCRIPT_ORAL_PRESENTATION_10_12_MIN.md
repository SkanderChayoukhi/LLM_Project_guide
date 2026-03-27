# Script Oral Pret a Presenter (10-12 minutes)

## Objectif

Ce script est ecrit pour etre lu tel quel a l'oral.
Tu peux le dire mot a mot ou l'adapter legerement.
Le contenu repond exactement aux 5 questions demandees.

---

## Plan rapide de prise de parole

- 0:00 a 0:45 Introduction
- 0:45 a 2:15 Question 1: probleme a resoudre
- 2:15 a 4:30 Question 2: structure de la solution
- 4:30 a 6:15 Question 3: connaissances du cours appliquees
- 6:15 a 7:15 Question 4: ressources hors cours
- 7:15 a 9:15 Question 5: evaluation de la solution
- 9:15 a 10:30 limites et ameliorations
- 10:30 a 11:30 conclusion
- 11:30 a 12:00 transition Q and A

---

## Script complet a lire

## 0:00 a 0:45 Introduction

Bonjour a tous.
Aujourd'hui je vous presente notre projet de shopping assistant multi-agent base sur un LLM.
L'idee est de partir d'une phrase libre utilisateur, de trouver des produits reels via API, de classer les options, puis de produire une recommandation qui reste verifiable et non hallucinee.
Notre fil conducteur pendant tout le projet a ete: utilite pour l'utilisateur, mais aussi fiabilite mesurable.

---

## 0:45 a 2:15 Which problem did you try to solve?

Le probleme qu'on a voulu resoudre est simple en apparence, mais difficile en pratique:
un utilisateur formule son besoin en langage naturel, parfois de facon vague,
et on doit quand meme lui recommander un produit pertinent et realiste.

Le risque principal quand on utilise un LLM est l'hallucination:
le modele peut proposer un produit qui semble credible, mais qui n'existe pas dans la source de donnees.

Donc notre vrai probleme n'etait pas seulement de recommander,
mais de recommander de facon traceable et verifiable.

En resume, on a cherche a repondre a trois exigences en meme temps:

- comprendre correctement l'intention utilisateur,
- proposer une recommandation de qualite,
- garantir que la recommendation est ancree dans des donnees reelles.

---

## 2:15 a 4:30 How did you structure the solution?

On a structure la solution comme un pipeline multi-agent,
ou chaque module a une responsabilite claire.

Etape 1:
On recoit la requete en texte libre dans l'interface.

Etape 2:
Le preference agent extrait des contraintes structurees:
product, budget et usage.

Etape 3:
Le api search agent interroge DummyJSON,
soit par endpoint categorie,
soit par endpoint recherche texte.

Etape 4:
Le search agent applique une logique locale deterministic:
filtrage par budget,
filtrage strict par tags,
puis scoring de pertinence.

Etape 5:
Le recommendation agent demande au LLM une sortie JSON structuree,
avec un best product id choisi uniquement parmi les candidats.

Etape 6:
On verifie la provenance:
on controle que best product id appartient bien aux ids retournes par l'API.

Etape 7:
On enregistre tout dans benchmark results,
puis le dashboard met a jour les KPI en temps reel.

Ce decoupage nous a permis de separer clairement:

- intelligence linguistique du LLM,
- logique metier locale explicable,
- et verification factuelle.

---

## 4:30 a 6:15 Which knowledge from the course did you apply?

Sur la partie connaissances du cours, on a applique cinq points de facon tres concrete.

Premier point: decomposition agentique.
On a separe le systeme en modules specialises: extraction, recherche API, scoring, recommendation et evaluation.
Cette separation suit directement la logique vue en cours: un role clair par agent pour eviter les boites noires.

Deuxieme point: prompt engineering structure.
On n'a pas demande des reponses libres au LLM.
On impose du JSON avec des cles connues, ce qui permet un traitement automatique fiable.

Troisieme point: grounding factuel.
La recommendation finale est confrontee a une source de verite externe.
Concretement, on verifie que l'id recommande existe dans les resultats API.

Quatrieme point: hybridation deterministic + generatif.
Le tri et les filtres sont locaux et explicables,
et le LLM est utilise pour la comprehension du besoin et l'explication utilisateur.

Cinquieme point: evaluation continue.
Chaque execution est tracee, puis exposee dans des KPI.
Donc on ne juge pas la solution a l'intuition, mais sur des indicateurs mesurables.

---

## 6:15 a 7:15 Did you need resources or tools not covered in the course?

Oui, on a utilise plusieurs outils complementaires:

- Gradio pour l'interface utilisateur et le dashboard,
- DummyJSON comme catalogue public de produits,
- matplotlib pour visualiser les tendances,
- un mecanisme custom de benchmark temps reel.

Le point important a retenir, c'est que ces outils ne remplacent pas le cours,
ils operationalisent les concepts du cours dans un systeme demoable et testable.

Par exemple:

- Gradio nous donne la couche interaction et observabilite,
- DummyJSON fournit une source de verite reproductible,
- matplotlib rend lisible l'evolution de la qualite,
- le logging JSON rend possible l'audit et le benchmark.

---

## 7:15 a 9:15 How does your solution evaluate?

Notre evaluation se fait sur deux niveaux.

Niveau 1, en ligne:
a chaque requete utilisateur,
on enregistre contraintes, resultats API, classement local, recommendation finale et statut de provenance.

Le dashboard calcule ensuite des KPI,
notamment:

- provenance rate,
- budget adherence,
- API coverage,
- total queries.

Niveau 2, hors ligne:
on a un dataset de cas de test,
et un script benchmark qui rejoue tout le pipeline automatiquement.

Pour chaque exemple,
on mesure si:

- on a des candidats,
- la categorie est coherente,
- le budget est respecte,
- et surtout si la recommendation est bien issue de l'API.

La metrique la plus critique pour nous est la provenance,
car elle mesure directement le risque d'hallucination.

Micro-explication du flux benchmark en 30 secondes:
on prend une requete utilisateur,
on stocke les contraintes extraites,
on stocke les produits retournes par l'API,
on stocke le classement local,
on stocke l'id recommande par le LLM,
puis on calcule provenance_ok avec la regle: best_product_id doit etre dans les ids API.
Ensuite le dashboard relit benchmark_results.json toutes les 5 secondes et recalcule les KPI.

Detail important pour montrer la rigueur de l'evaluation:
on suit aussi l'adherence budget avec une tolerance explicite,
prix recommande <= budget \* 1.3,
ce qui evite d'eliminer trop agressivement des produits proches du budget.

Donc, au final, l'evaluation combine:

- robustesse factuelle (provenance),
- adequation economique (budget),
- disponibilite donnees (API coverage),
- et tendance temporelle (dashboard).

---

## 9:15 a 10:30 Limites et ameliorations

La solution est fonctionnelle, mais on voit deja les limites.

Les limites actuelles:

- scoring base sur heuristiques et mots-cles,
- mappings statiques,
- validation JSON LLM encore perfectible,
- dataset de benchmark encore limite en taille.

Nos ameliorations prioritaires:

- validation stricte de schema,
- reranking semantique avec embeddings,
- enrichissement des metriques,
- plus de tests automatiques.

---

## 10:30 a 11:30 Conclusion

Pour conclure,
notre contribution principale n'est pas seulement une recommendation produit,
mais une recommendation verifiable.

On combine:

- la flexibilite du LLM pour comprendre l'utilisateur,
- la robustesse d'une logique deterministe pour filtrer et scorer,
- et une verification de provenance qui reduit les hallucinations.

Ce projet est donc a la fois demonstrable en live,
et auditable techniquement.

---

## 11:30 a 12:00 Transition questions

Merci pour votre attention.
Si vous voulez, je peux maintenant montrer un exemple en direct,
puis expliquer en detail une partie precise:
soit l'extraction des contraintes,
soit le scoring,
soit le benchmark et les KPI.

---

## Mini anti-seche (si tu stresses)

Si tu perds le fil, reviens a cette phrase:
Notre projet transforme une demande libre en recommendation produit,
avec verification automatique que le produit recommande existe vraiment dans la source API.

Puis enchaine avec:
probleme, architecture, cours applique, outils externes, evaluation.

---

## Questions probables du jury et reponses courtes

Question:
Pourquoi un pipeline multi-agent au lieu d'un seul prompt geant?

Reponse:
Parce qu'on veut separer ce qui doit etre deterministic de ce qui peut etre generatif.
Cette separation augmente la lisibilite, la debuggabilite et la fiabilite.

Question:
Comment prouvez-vous qu'il n'y a pas d'hallucination?

Reponse:
On ne le prouve pas absoluement,
mais on detecte automatiquement le cas principal:
si l'id recommande n'est pas dans les ids API, la provenance echoue.

Question:
Pourquoi DummyJSON?

Reponse:
Pour avoir une source publique simple, stable et reproductible,
qui permet de tester rapidement le pipeline sans dependre d'un backend prive.

Question:
Votre score est-il optimal?

Reponse:
Non, c'est un baseline explicable.
Il est volontairement simple pour etre interpretable,
ensuite on peut evoluer vers du reranking semantique.

Question:
Quelle est la prochaine etape la plus importante?

Reponse:
La validation stricte des sorties LLM et l'amelioration de la pertinence semantique.
