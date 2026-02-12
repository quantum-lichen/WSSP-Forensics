# L'obstruction structurelle de l'audit système

## L'Observateur d'événements comme pattern de conception hostile et théâtre de sécurité

L'asymétrie fondamentale entre la capacité d'un système d'exploitation à générer des données de surveillance et la capacité de l'utilisateur à les auditer constitue l'un des enjeux majeurs 
de la souveraineté numérique contemporaine.

Dans l'écosystème Windows, cette disparité n'est plus réductible à une simple dette technique ou à l'obsolescence de code hérité. L'analyse systémique menée par les experts en 
cyber-forensique et en psychologie cognitive suggère l'existence d'une stratégie délibérée de **« design frictionnel »** visant à protéger la télémétrie système contre l'investigation de 
l'utilisateur final.

Alors que le format de fichier **.evtx** permet théoriquement de stocker des volumes massifs de données, l'outil natif de lecture, l'**Observateur d'événements (Event Viewer)**, demeure 
piégé dans une architecture datant de l'ère Windows 2000, rendant l'accès aux preuves forensiques pratiquement impossible pour des fichiers de grande taille. Ce rapport explore la thèse 
d'une **« fraude cognitive »** où la transparence affichée par Microsoft n'est qu'une façade destinée à décourager l'audit citoyen au profit d'une surveillance opaque et centralisée.

---

## La faillite technique de l'Observateur d'événements face au format EVTX

Le format de journalisation Windows Event Log (EVTX), introduit avec Windows Vista pour remplacer le format .evt, a été conçu pour offrir une structure binaire XML segmentée en 
**« chunks »** auto-contenus de 64 Ko. Cette architecture permet théoriquement une résilience élevée et une capacité de stockage presque illimitée, le système de fichiers NTFS pouvant 
supporter des fichiers jusqu'à 16 téraoctets. Cependant, une contradiction flagrante apparaît dès lors que l'on examine l'outil de lecture fourni par défaut.

### L'impasse de l'architecture MMC et du mappage mémoire

L'Observateur d'événements est implémenté comme un composant logiciel enfichable (snap-in) pour la **Microsoft Management Console (MMC)**. Cette architecture repose sur un modèle de 
rendu monothreadé et un accès aux fichiers via le mappage mémoire (memory-mapped files).

Lorsqu'un utilisateur tente d'ouvrir un journal de sécurité accumulant 100 Go de données, le système tente de projeter la structure du fichier dans l'espace d'adressage virtuel du 
processus `mmc.exe`. Sur des machines disposant d'une mémoire RAM standard (8 à 32 Go), cette opération entraîne une saturation immédiate des ressources, des freezes systémiques et 
des échecs de rendu.

| Paramètre Technique | Limite de l'Observateur d'événements | Capacité théorique du format EVTX |
| --- | --- | --- |
| **Taille de fichier maximale** | ~2 Go (instable deja à 20 mo) | Jusqu'à 16 To (limite NTFS) |
| **Nombre d'entrées** | < 10 000 entrées [Cas d'usage] | ~1 800 000 par tranche de 2 Go |
| **Gestion de la mémoire** | Mappage mémoire direct (Bloquant) | Lecture par segments XML (Non-bloquant) |
| **Performance de recherche** | Séquentielle monothreadée | Indexation multi-threadée possible |

Les administrateurs système rapportent fréquemment que l'augmentation de la taille des journaux au-delà des 20 Mo par défaut (un vestige de l'ère des disques durs limités) paralyse l'outil 
de diagnostic. Le passage à une limite de 2 Go, bien que recommandée pour conserver un historique décent, rend la navigation dans l'interface graphique si lente qu'elle en devient 
inutilisable pour une réponse aux incidents en temps réel.

---

## L'exception System.OverflowException : Un goulot d'étranglement structurel

Un symptôme technique critique de cette obsolescence est l'occurrence fréquente de l'erreur **System.OverflowException**. Cette exception survient lorsque les compteurs internes ou les 
index de la console MMC tentent de traiter un nombre d'événements dépassant la plage de valeurs d'un entier 32 bits (Int32), ou lors de conversions maladroites entre Int64 et Int32 au 
sein du code de rendu.

Dans des scénarios de télémétrie agressive où un heartbeat de 5 secondes génère des milliers d'entrées par heure, le nombre total d'événements peut atteindre des valeurs que l'interface 
graphique ne sait plus comptabiliser.

Le résultat pour l'utilisateur est un échec silencieux ou un crash complet de l'application. Plus grave encore, la fonction **« Enregistrer tous les événements sous... »** échoue souvent 
à produire un fichier valide lorsque le journal est dans cet état de surcharge, générant des fichiers de 0 octet et privant ainsi l'auditeur de toute possibilité d'exportation vers un 
outil tiers. Le système force alors l'utilisateur à choisir entre conserver un système instable ou **« Effacer le journal »** pour restaurer la fonctionnalité, supprimant ainsi les preuves 
de l'exfiltration suspectée.

---

## Le paradoxe de la stagnation : Évolution du Menu Démarrer vs Pétrification des outils d'audit

Pour démontrer l'intentionnalité derrière cette défaillance, il convient d'analyser la répartition des ressources de développement chez Microsoft. Une comparaison entre l'évolution des 
éléments esthétiques et commerciaux (Menu Démarrer) et celle des outils de transparence technique (Observateur d'événements) révèle une disparité flagrante qui ne peut être expliquée 
uniquement par la conservation de code hérité (legacy code).

### La volatilité du Menu Démarrer comme priorité commerciale

Le Menu Démarrer a été le théâtre de réinventions radicales à presque chaque version de Windows, mobilisant des équipes entières de designers UX et d'ingénieurs pour modifier l'esthétique, 
l'ergonomie et l'intégration de services cloud.

| Version de Windows | Changements majeurs du Menu Démarrer | Statut de l'Observateur d'événements |
| --- | --- | --- |
| **Windows 95** | Invention du menu en cascade. | Interface basique (type gestionnaire de fichiers). |
| **Windows XP** | Design à deux colonnes (Luna UX). | Intégration dans la console MMC (stagnation). |
| **Windows Vista** | Recherche instantanée et flou Aero. | Introduction du format .evtx, interface inchangée. |
| **Windows 8** | Écran plein écran et tuiles dynamiques. | Toujours l'interface Windows 2000 (MMC). |
| **Windows 10** | Menu hybride (tuiles et listes). | Stagnation complète, bogues de rendu persistants. |
| **Windows 11** | Centrage, suppression des tuiles, flux « Recommandé ». | Toujours l'interface 2000, bogues d'affichage. |
| **Win 11 (2025/26)** | Vues par catégories, intégration Phone Link. | Abandon technique total, interface anachronique. |

Le Menu Démarrer a été modifié plus de dix fois en trente ans, souvent contre la volonté explicite des utilisateurs. Pendant ce temps, l'Observateur d'événements conserve une structure de 
1999, incapable de gérer les écrans haute résolution, le mode sombre de manière native, ou les volumes de données du XXIe siècle. Cette divergence suggère que Microsoft considère le 
Menu Démarrer comme un outil de capture de l'attention utilisateur, tandis que l'Observateur d'événements est traité comme un résidu technique que l'on préfère laisser à l'abandon pour 
masquer la complexité croissante de la surveillance moderne.

---

## La friction du design comme barrière à l'entrée

En psychologie du design, l'absence de mise à jour d'un outil critique dans un environnement par ailleurs ultra-moderne crée ce que l'on appelle une **« barrière frictionnelle »**.

L'interface de l'Observateur d'événements, avec ses dialogues lents, ses menus contextuels obsolètes et ses plantages répétitifs sur les gros fichiers, signale inconsciemment à 
l'utilisateur que cette zone du système n'est pas pour lui. C'est une forme d'**obstruction par le design (Obstruction by Design)** : on ne supprime pas la fonctionnalité, on la rend si 
pénible à utiliser que seuls les experts les plus acharnés y ont recours.

### Théâtre de sécurité et patterns de design hostile

La persistance de l'Observateur d'événements dans son état actuel relève du **« Théâtre de sécurité »**, un concept théorisé par Bruce Schneier pour décrire des mesures qui procurent une 
sensation de sécurité sans réellement l'augmenter. En fournissant un journal qui enregistre tout mais que personne ne peut lire sans outils tiers coûteux, le système d'exploitation crée 
une illusion de transparence.

### La transparence de façade : Un pattern de fraude cognitive

Le concept de **« transparence de façade »** s'applique ici parfaitement : le système Windows permet d'ouvrir les vannes de la journalisation (le robinet est grand ouvert), mais fournit 
une assiette minuscule (l'interface graphique) pour recueillir les données. Cette tactique permet à l'entreprise de se prémunir contre les accusations d'opacité tout en sachant 
pertinemment que l'outil fourni est structurellement défaillant pour des volumes de données forensiques réels.

---

## Télémétrie agressive et déni de service par le bruit

L'accumulation de 100 Go de logs suite à un heartbeat de 5 secondes constitue une forme de **« déni de service » (DoS)** contre l'audit utilisateur. En inondant le journal de sécurité 
d'entrées de télémétrie « bénignes », le système enterre les preuves d'une exfiltration réelle sous une montagne de données inutiles.

| Stratégie d'obstruction | Mécanisme technique | Effet psychologique |
| --- | --- | --- |
| **Inondation (Flooding)** | Heartbeat 5s, journaux verbeux. | Sensation de « chercher une aiguille dans une botte de foin ». |
| **Fragilité logicielle** | Crash au-delà de 2 Go. | Abandon de l'investigation, suppression des preuves. |
| **Obsolescence UX** | Interface MMC de 1999. | Impression de complexité ou d'instabilité. |
| **Rétention trompeuse** | GPO autorisant de grandes tailles. | Sentiment d'être trahi par l'outil de confiance. |

Cette pratique s'apparente au **« Vandalisme Cognitif »** décrit par **Bryan Ouellette** : on altère la capacité de l'individu à percevoir la réalité de son environnement numérique en 
dégradant les outils de perception (les logs). L'utilisateur ne peut plus **« ancrer sa réalité » (Reality Anchoring)** dans les faits bruts, car ces faits sont rendus inaccessibles par 
la lenteur délibérée de l'outil d'interprétation.

---

## Précédents de conformité malveillante : De la RGPD au Google Takeout

La situation de l'Observateur d'événements trouve des échos directs dans d'autres domaines du numérique où les entreprises pratiquent la **« Conformité Malveillante » (Malicious Compliance)
**. Il s'agit de respecter la lettre de la loi ou d'un engagement de transparence tout en en bafouant l'esprit par une implémentation rendant le résultat inexploitable.

### Le « Data Dump » illisible comme arme de dissuasion

Sous l'impulsion de la RGPD, les entreprises fournissent un accès aux données personnelles, mais souvent dans des formats volontairement complexes (milliers de fichiers JSON imbriqués). 
L'objectif est identique : dégoûter l'utilisateur du processus de vérification. On observe également ce pattern dans les bannières de cookies, où le refus du traçage est rendu 
intentionnellement tortueux.

### La conformité malveillante dans l'OS Windows

Dans le cas de Windows, la conformité malveillante réside dans la promesse d'un système **« sécurisé par conception » (Secure by Design)**. Microsoft fournit tous les outils de 
journalisation nécessaires, mais en laissant l'outil de lecture local en décrépitude, l'entreprise crée une dépendance forcée envers ses solutions payantes de cloud et de SIEM.

---

## Solutions de contournement vs silence institutionnel : Le paywall de la transparence

### L'existence occulte d'outils performants

Il est techniquement possible de lire des fichiers EVTX de 100 Go de manière quasi-instantanée (ex: outils d'Eric Zimmerman ou `EventLogExpert`). Le fait que Microsoft ne remplace pas l'Observateur d'événements par ces technologies modernes confirme une volonté de maintenir l'utilisateur lambda dans l'impuissance technique.

### Le passage forcé au Cloud (Azure Monitor / Sentinel)

La stratégie commerciale consiste à **« paywaller »** la transparence. Plus le système génère de logs, plus l'utilisateur est poussé à payer pour des services cloud afin de comprendre ce qui se passe sur sa propre machine.

| Option de diagnostic | Coût | Performance / Capacité d'audit |
| --- | --- | --- |
| **Observateur natif** | "Gratuit" | Nulle sur les gros fichiers. |
| **PowerShell** | Gratuit | Moyenne, nécessite du script. |
| **Outils Tiers** | Gratuit / Don | Excellente (hors écosystème MS). |
| **MS Sentinel / Azure** | Élevé | Industrielle, centralisée, opaque localement. |

---

## La perspective des Lichen-Collectives : Pour une reconquête de l'imaginaire technique

L'approche de **Bryan Ouellette** et du mouvement **Lichen-Collectives** invite à voir dans cette situation une opportunité de « régénération de nos imaginaires » techniques. Le système actuel n'est pas seulement « cassé », il est « codé » pour nous asservir à une perception fragmentée de la vérité.

### Résistance par l'Ancre du Réel

Le projet **« OPERATION-ANCRE-DU-REEL »** propose de refuser l'illusion de transparence et de développer des alternatives distribuées et locales. Le lichen, modèle de cet écosystème, survit là où tout le reste échoue en travaillant lentement mais de manière irréversible. L'auditeur doit délaisser les architectures centralisées de Microsoft au profit d'un cadre de défense ancré dans la réalité technique brute.

> « Nous ne sommes pas venus réparer leur système. Nous sommes venus coder le nôtre ».

Cette philosophie s'applique à la cyber-forensique : l'utilisateur doit s'approprier les parseurs indépendants qui redonnent au fichier log sa fonction première : être une preuve irréfutable.

---

## Conclusion : L'Observateur d'événements, un instrument de silence délibéré

L'enquête démontre que l'Observateur d'événements remplit une fonction précise : protéger la télémétrie système de l'examen par l'utilisateur.

1. **L'échec technique est une fonctionnalité** : Les erreurs d'overflow incitent à l'effacement des preuves.
2. **L'abandon UX est politique** : Décourager les non-experts pour maintenir l'opacité.
3. **La transparence est payante** : La capacité d'audit réelle est réservée aux clients Azure.
4. **La télémétrie est un déni de service** : Aveugler l'auditeur local par le bruit blanc.

La souveraineté numérique passe par la reconnaissance que seule l'utilisation d'outils forensiques indépendants permet de rompre le charme de la transparence de façade. Le lichen numérique doit maintenant croître sur les ruines de l'architecture MMC pour restaurer la vérité des faits informatiques.

---
Bryan Ouellette du Lichen-Collectives