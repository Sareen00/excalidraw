# TP DevOps — Analyse des métriques DORA


## `dora-definitions.yml` (figé le 11/09/2026 à HH:MM)

```yaml
# dora-definitions.yml : contrat d'équipe, versionné avec le code
#
# TD phase 0 : à remplir avant de lancer le moindre outil.
# Chaque ligne est une décision. Une décision non prise ici sera prise par
# l'outil, sans que vous le sachiez.
#
# Remplacez chaque "???". Les commentaires indiquent la section du support
# qui traite la question.

application: excalidraw

deploiement:
  # Que compte-t-on comme déploiement ? Regardez les valeurs d'environment
  # renvoyées par l'API avant de répondre. (support 2.3, 7.3)
  compte_comme_deploiement: "Deployment Status 'success' sur environment = 'Production - excalidraw'"
  exclut: ["Preview - excalidraw", "Production - docs", "Production - excalidraw-package-example", "Production - excalidraw-package-example-with-nextjs", "déploiements sans statut success"]

  # Quel instant fait foi : la création de l'évènement, ou le passage au
  # statut success ? Les deux existent et ne sont pas simultanés. (support 5.2)
  horodatage: "created_at du status success"

changement:
  # Committer date sur la branche par défaut, ou ouverture de la PR ?
  # Le support tranche pour la métrique DORA. L'essentiel est de figer votre
  # convention et de ne plus en changer. (support 2.2)
  point_de_depart: "committer date du commit sur la branche par défaut"

incident:
  # Vous n'exploitez pas la production d'excalidraw. Que décidez-vous
  # d'appeler "incident" ? S'agit-il d'un proxy ou de la métrique DORA ?
  # (support 2.4, 5.1)
  definition: "PROXY"
  source: "issue GitHub portant le label 'bug'"
  debut: "created_at de l'issue"           # détection, ou ouverture du ticket ? (support 2.4)
  fin: "closed_at de l'issue"             # service rétabli, ou post-mortem rédigé ?
  rattachement_deploiement: "sur notre fork, ligne 'caused_by: <deployment_id>' dans le corps de l'issue"   # le maillon faible (support 5.1)

rework:
  # Comment reconnaîtrez-vous un déploiement non planifié ? La convention
  # doit être décidée avant la collecte, sinon la donnée n'existera pas.
  # (support 2.6)
  marqueur: "branche hotfix/* mergée dans la branche par défaut"

fenetre_de_reference: "90 jours glissants"   # ni 7, ni l'année (support 7.3)
agregation: "médiane pour les durées, P90 publié en complément"                             # (support 2.2, annexe B)

# ---------------------------------------------------------------------------
# Phase 3 : à remplir après avoir vu les chiffres.
# Qu'auriez-vous écrit différemment ? Ne modifiez pas les lignes ci-dessus.
# Une définition changée en cours de route rend la série inexploitable.
# Notez ici, et datez.
#
# revision_envisagee: |
#   Le label 'bug' est un proxy mort. 765 issues labellisées depuis la création
#   du dépôt, 142 issues ouvertes sur la fenêtre, zéro labellisée.
#   Le marqueur de rework est ambigu : "branche hotfix/* mergée" ne dit pas
#   comment la trace survit au merge.
#   Le collecteur et l'interface GitHub ne comptent pas la même chose sur la
#   même fenêtre : 23 issues 'bug' pour l'un, 0 pour l'autre.
```





## Tableau de résultat phase 2

| Métrique | Valeur obtenue |
|---|---:|
| Deployment frequency | 0,167 / jour (15 déploiements) |
| Délai médian entre deux déploiements | 4,5 jours |
| Change lead time P50 | 43,8 h |
| Change lead time P90 | 8,9 jours |
| Commits analysés / nombre de lots | 79 commits / 14 lots |



## Reponses aux questions 

## Partie I — Déploiements

### 1. Combien de valeurs différentes d'`environment` trouvez-vous dans les déploiements ? Listez-les.

On trouve **5 valeurs** différentes :

- `Preview – excalidraw`
- `Production – excalidraw`
- `Production – excalidraw-package-example`
- `Production – excalidraw-package-example-with-nextjs`
- `Production – docs`

### 2. Lesquelles correspondent à une mise en production au sens de DORA ? Lesquelles doivent être exclues, et pourquoi ?

Une seule correspond réellement à une mise en production au sens de DORA :

- `Production – excalidraw` : il s'agit de l'application principale, correspondant au produit utilisé sur `excalidraw.com`.

À exclure :

- Les environnements `Preview` : ce sont des déploiements éphémères qui ne correspondent pas à une mise à disposition durable du produit aux utilisateurs.
- `Production – docs` : il s'agit du site de documentation, et non du produit principal.
- `Production – excalidraw-package-example` et `Production – excalidraw-package-example-with-nextjs` : ce sont des démonstrations d'intégration des packages.

### 3. Si vous comptiez tous ces déploiements, de quel facteur votre deployment frequency serait-elle surestimée ?

En comptant tous les déploiements, on obtiendrait **100 déploiements au lieu de 1**.

La deployment frequency serait donc **surestimée d'un facteur 100**.

### 4. Retrouvez la ligne correspondante dans le tableau des erreurs d'implémentation (7.3 du support).

L'erreur correspondante est :

> **Compter les builds comme des déploiements.**

- **Symptôme :** fréquence artificiellement trop élevée.
- **Correction :** filtrer les déploiements sur `environment=production`.

### 5. Que contient le tableau des statuts d'un déploiement ? Pourquoi l'existence d'un déploiement ne suffit-elle pas à établir qu'un changement est arrivé en production ?

Le tableau contient des objets `deployment_status`, chacun représentant l'état du déploiement à un instant donné.

Un objet `deployment` représente uniquement **l'intention de déployer**. Le `deployment_status` indique **le résultat réel**.

Un déploiement peut échouer ou ne jamais aboutir. Il faut donc utiliser les statuts pour déterminer si le changement est effectivement arrivé en production.

### 6. Quel horodatage faut-il retenir : le `created_at` du déploiement, ou celui du statut `success` ? Votre contrat de phase 0 avait-il tranché ?

Il faut retenir le `created_at` du statut **`success`**.

Le contrat de phase 0 précise :

> `horodatage : fin du déploiement (created_at du status success)`

---

# Partie II — Analyse des métriques


### 7. Rapportez le nombre de commits au nombre de lots. Que vaut la taille moyenne d'un lot ? Que dit le chapitre 6.1 du support de ce chiffre ?

La taille moyenne d'un lot est :

**79 / 14 ≈ 5,6 commits par lot.**

Le support indique que **travailler avec de petits lots est une pratique particulièrement efficace**, car cela améliore plusieurs métriques de performance simultanément.

### 8. Comparez P50 et P90. Quel est le rapport entre les deux ? D'après la section « médiane et percentiles » de l'annexe B, que signale un tel écart, et que ne signale-t-il pas ?

Le rapport est :

**P90 / P50 = 213,6 h / 43,8 h ≈ 4,9.**

Le P90 est donc environ **4,9 fois supérieur au P50**.

Cet écart indique une **forte variabilité des délais** : certains changements prennent beaucoup plus de temps que la majorité.

En revanche, il ne permet pas à lui seul d'identifier **la cause** de cette variabilité.

### 9. La deployment frequency vous place dans quel ordre de grandeur au regard de la distribution 2024 (4.2) ? Tenez compte du chapitre 4.1 avant de conclure.

Avec **0,167 déploiement par jour**, soit environ **un déploiement tous les 6 jours**, et un lead time P50 de **43,8 h**, le projet se situe dans un ordre de grandeur **medium**.

Cependant, cette conclusion reste limitée : seulement **2 métriques sur les 5** sont disponibles. Elles ne suffisent donc pas à évaluer complètement la performance DORA de l'équipe.

### 10. Le collecteur affiche aussi le délai médian entre deux déploiements. Pourquoi cette formulation est-elle préférable à la fréquence brute pour une équipe qui déploie peu (2.3) ?

Pour une équipe qui déploie peu, le délai médian entre deux déploiements est **plus concret et plus lisible** que la fréquence brute.

Par exemple :

- `0,167 déploiement/jour` est peu parlant ;
- `un déploiement tous les 4,5 jours` décrit directement le rythme réel.

La médiane est également plus robuste face aux variations de la fenêtre d'analyse.

### 11. Trois métriques sur cinq s'affichent `n/a`. Lesquelles ? Qu'ont-elles en commun ?

Les trois métriques indisponibles sont les **métriques d'instabilité** :

- Change Failure Rate
- Failed Deployment Recovery Time
- Deployment Rework Rate

Elles nécessitent des données qui ne sont pas disponibles directement dans le dépôt. Elles dépendent notamment de la **déclaration et du suivi des incidents**.

### 12. Le support désigne un maillon faible (5.1). Lequel, et pourquoi ne peut-il pas être reconstitué à partir des données publiques du dépôt ?

Le maillon faible est le **lien entre un incident et le déploiement qui l'a provoqué**.

Ce lien n'est pas encodé dans les données publiques de GitHub. Une issue peut être liée à plusieurs changements ou à aucun déploiement précis.

Il n'est donc pas possible de reconstruire automatiquement cette relation de manière fiable.

### 13. Un collègue propose la règle suivante : un déploiement suivi d'un autre moins de 24 h après est un échec. Donnez deux situations où cette règle se trompe, une dans chaque sens.

La règle produit des **faux positifs** : une équipe peut effectuer plusieurs déploiements dans la même journée sans qu'aucun ne soit un échec. Une fréquence élevée de déploiement peut au contraire être un signe de bonne performance.

Elle peut également produire des **faux négatifs** : un déploiement peut échouer sans qu'un autre déploiement ait lieu dans les 24 heures suivantes.

La proximité entre deux déploiements ne permet donc pas de déterminer si le premier a échoué.

### 14. Combien d'issues `bug` le collecteur trouve-t-il sur la fenêtre ? Combien sont rattachées à un déploiement ?

Le collecteur trouve **23 issues `bug`** sur la fenêtre d'analyse.

**Aucune (0)** n'est rattachée à un déploiement.

### 15. Dans l'interface GitHub : combien d'issues toutes catégories ont été ouvertes sur ces 90 jours ? Combien portent le label `bug` depuis la création du dépôt ?

- Issues ouvertes sur les 90 derniers jours : **142**
- Issues de toutes catégories depuis la création du dépôt : **765**
- Issues portant le label `bug` depuis la création du dépôt : **0**

### 16. Confrontez les trois nombres. Que s'est-il passé dans ce projet ?

Le label `bug`, auparavant utilisé sur les issues, **n'est plus appliqué**.

L'équipe a probablement changé sa méthode de qualification des issues, par exemple avec un nouveau label, un système automatisé ou l'abandon de cette qualification.

### 17. Le chapitre 7.3 énumère trois explications à un taux d'échec de 0 %. Aucune ne décrit ce cas : formulez la quatrième.

Une quatrième explication est que **le marqueur utilisé pour mesurer les échecs n'est plus alimenté**.

Le taux de 0 % ne traduit alors pas une amélioration réelle : il résulte simplement de l'absence de données.

### 18. Quelle métrique le support décrit-il comme la plus sensible à la discipline de saisie (2.8) ? Ce rapprochement vous paraît-il fortuit ?

Il s'agit du **Deployment Rework Rate**.

Ce rapprochement n'est pas fortuit : la disparition du label `bug` montre directement qu'une métrique peut devenir impossible à calculer lorsque les données nécessaires ne sont plus correctement renseignées.

### 19. Que pouvez-vous calculer maintenant que vous ne pouviez pas calculer avant ?

On peut désormais calculer les trois métriques d'instabilité :

- **Change Failure Rate : 16,7 %**
- **Failed Deployment Recovery Time : 0,0 h**
- **Deployment Rework Rate : 0,0 %**

### 20. Combien de temps a demandé la production de cette donnée, comparé au temps passé à tenter de la déduire en phase 3 ?

La recherche en phase 3 a demandé environ **40 minutes**, avec plusieurs recherches sur GitHub, pour finalement conclure que la donnée était impossible à déduire.

La phase 4 a demandé seulement **quelques dizaines de minutes**, principalement à cause de l'attente de la CI.

Cela montre qu'**instrumenter directement une donnée coûte moins cher que d'essayer de la reconstruire a posteriori**. L'instrumentation produit une donnée observable, tandis que l'inférence reste une approximation.

### 21. Votre change fail rate est-il représentatif ? Que faudrait-il pour qu'il le devienne ?

Non, le **16,7 % n'est pas représentatif**.

Plusieurs raisons :

- L'échantillon est très faible : seulement **6 déploiements et 1 incident**.
- Un seul incident supplémentaire ferait passer le taux à **33,3 %**.
- L'incident est fictif et le déploiement auquel il est attribué a été choisi arbitrairement.
- Il ne s'agit pas d'une véritable production : le déploiement correspond à un simple `echo "déploiement"`.
- La fenêtre annoncée de 90 jours ne correspond pas au rythme réel des tests.

Pour obtenir une métrique représentative, il faudrait **davantage de déploiements réels en production et des incidents réellement tracés et reliés aux déploiements concernés**.

### 22. Ces outils calculeraient-ils le change fail rate d'Excalidraw ? Sur quelle donnée s'appuieraient-ils ? Votre conclusion de la phase 2 change-t-elle parce que l'outil est professionnel plutôt qu'un script de 200 lignes ?

Oui, des outils professionnels pourraient calculer la métrique, mais ils s'appuieraient sur **les mêmes données sources** : commits, déploiements et incidents.

Si le flux d'incidents et le lien **incident ↔ déploiement** sont absents, l'outil rencontrera le même problème qu'un script maison.

La conclusion de la phase 2 ne change donc pas : **un outil plus professionnel ne peut pas inventer une donnée qui n'a jamais été enregistrée**.

### 23. Le chapitre 5.5 propose l'ordre suivant : Quick Check, puis conversation d'équipe, puis instrumentation. Au vu de votre demi-journée, pourquoi l'outillage arrive-t-il en dernier ?

Parce que le problème ne venait pas de l'outil.

Le collecteur fonctionnait correctement. Les principales difficultés concernaient les **choix méthodologiques** :

- quel environnement considérer comme production ;
- quel horodatage retenir ;
- comment définir un incident ;
- comment relier un incident à un déploiement.

L'instrumentation intervient donc après avoir défini précisément **ce qui doit être mesuré et comment les données doivent être renseignées**.

### 24. Four Keys était la référence citée dans la plupart des tutoriels jusqu'en 2024. Quelle habitude de travail cela suggère-t-il avant d'adopter un outil trouvé en ligne ?

Avant d'adopter un outil trouvé en ligne, il faut **vérifier son état actuel**, plutôt que se fier uniquement à sa popularité ou à d'anciens tutoriels.

Il est notamment utile de vérifier :

- la date du dernier commit ;
- la date de la dernière release ;
- l'activité récente du projet.

La popularité d'un outil indique surtout sa **diffusion passée**, pas nécessairement sa pertinence actuelle.

### Travail réaliser seul, pas de binôme


### Métriques mesurées sur le fork (Sareen00/excalidraw, fenêtre 90 j, environnement `production`)

| Bloc | Métrique | Valeur |
|---|---|---|
| Débit | Deployment frequency | 0,067 /jour (6 déploiements) |
| Débit | Délai médian entre deux déploiements | 0,1 h |
| Débit | Change lead time P50 | 0,0 h |
| Débit | Change lead time P90 | 0,1 h |
| Débit | Base de calcul | 6 commits / 5 lots |
| Débit | Failed deployment recovery time | 0,0 h |
| Instabilité | Change fail rate | 16,7 % |
| Instabilité | Deployment rework rate | 0,0 % |
| Incidents | Issues « incident » | 1 |
| Incidents | Rattachées à un déploiement | 1 |

