# metaltura/platform

La pile de la plateforme Kogitec : les patrons qui écrivent, depuis un modèle de domaine ou un modèle d'entités déjà dessinées, une application de la plateforme, puis son projet npm sur les paquets `@kogitec/*` publiés. Contrairement aux deux autres piles, elle écrit directement dans le modèle `Platform` sans passer par une Application.

## Ce qu'il contient

- `models/Compositions.yaml` — les compositions de la plateforme (`Platform:component.Composition`, PLAT-950), écrites et modifiées à la main pour tous les domaines : aucun mapper n'y touche. `EntityScreen` est l'écran d'une entité : une page avec un conteneur, la liste à gauche et le formulaire à droite ; le titre de la page et l'entité du conteneur sont liés à ses deux propriétés, `title` et `entity`, et l'aperçu montre `Platform:Doc`. Un changement de son contenu vaut pour l'écran de toutes les entités de tous les domaines.
- `models/DomainToPlatform.yaml` — la pile depuis un domaine (PLAT-720), dans le paquet `platform`. Le patron de tête `domainToPlatform` produit une entité modélisée par entité du domaine (`Platform:Entity`, qui hérite de `Platform:Doc` : un document rangé dans Mongo, avec une copie des champs de ses parents et ses relations, dont l'opposée est recopiée), puis le module du domaine avec un bouton par entité et une route `/<domaine>` qui porte l'écran de chaque entité. Chaque entité qui a un écran reçoit aussi, à la racine du modèle produit, sa liste, son écran et sa publication (PLAT-1046) ; la page de sa route (`entityToScreen`) est une `Page` qui tient le document choisi et pose une carte de son écran. Le modèle produit dépend de `PlatformComponents`, le catalogue des composantes que ces écrans posent. Le patron de tête est la classe `DomainToPlatformStack`, et son instance `domainToPlatform`. Le module, ses boutons et l'écran de chaque entité sont écrits par les fonctions de `EntitiesToPlatform:platform.ScreenPattern` (PLAT-1459) : une correction de l'écran vaut pour les deux patrons.
- `models/EntitiesToPlatform.yaml` — le patron d'un modèle d'entités déjà dessinées (PLAT-1459), dans le paquet `platform`. Le patron de tête `entitiesToPlatform` se branche sur un modèle ou un package dont les éléments sont des entités de la plateforme (`Platform:Entity`). Il n'écrit aucune entité : il écrit dans la cible de son Mapping le module de la source, et rien d'autre.
  - Le module porte le nom de la source en minuscules (`bibliotheque`) et son titre est ce nom mis en mots.
  - Chaque document de la source, packages compris et dans l'ordre de lecture, a un bouton et un écran (`/bibliotheque/livres`). Sa liste, son écran et sa publication sont écrits dans la cible comme pour un domaine, et la page de sa route pose l'écran. Ils nomment l'entité là où elle est dessinée (`Bibliotheque:Livre`).
  - Un document est une entité dont la chaîne `extends` atteint `Platform:Doc`. Une entité qui n'en est pas un n'a ni bouton ni écran, et le journal du serveur le dit avec son chemin (« Bibliotheque:Adresse: not a document, no screen »). Une seconde entité du même nom est sautée, et une source sans document n'écrit rien : le journal le dit aussi.
  - Une entité dont la liste, l'écran ou la publication prendrait un nom déjà pris n'a ni écran, ni bouton, ni route, et le journal du serveur le dit (« Bibliotheque:Auteur: the name AuteurScreen is already taken, no screen »).
  - Le modèle cible dépend du modèle source et de `PlatformComponents` : sans ces dépendances, l'onglet « Erreurs » signale l'écran qui nomme une entité ou une composante introuvable.

  La classe abstraite `ScreenPattern` porte les fonctions que les deux patrons partagent :

  - `isDocument` ;
  - `screenPathOf` ;
  - `screenOf` ;
  - `moduleOf` ;
  - `targetOf` : le document au bout d'un lien, l'entité elle-même ou celle de ce nom parmi les entités gardées ;
  - `labelFieldOf` : le champ qui nomme un document, par la règle de la plateforme (le premier texte simple, sinon `_id`) ;
  - `screenCompositionsOf` : la liste (`<Entité>List`) et l'écran (`<Entité>Screen`) d'une entité, sur le patron d'`EssaiClients` de PLAT-1009 et la mise en page de PLAT-1046 ;
  - `publicationOf` : la publication (`<Entité>Publication`) que l'écran demande, avec le document de chaque lien vers un seul document ;
  - `screenedOf` : les entités qui ont un écran, celles dont aucun de ces trois noms n'est déjà pris.

  `isDocument` prend une entité ; `screenPathOf`, `screenOf` et `moduleOf` prennent des noms et des chemins. `DomainToPlatform` s'en sert pour un domaine. Son diagramme `PatternsClassDiagram` montre `ScreenPattern` et les deux patrons du paquet.
- `models/PlatformToCode.yaml` — le patron de génération de la pile, dans le paquet `platform` : le projet npm d'une application plateforme (PLAT-721), sur les paquets `@kogitec/*` publiés. Le patron de tête `platformToCode` part du modèle ou du package qu'écrit `domainToPlatform` ou `entitiesToPlatform` et rend un seul projet, de la forme de l'app minimale de la plateforme : sa configuration (`package.json` · `tsconfig.json` · `vite.config.ts` · `.npmrc` · `index.html` · `README.md`), un schéma `docSchema` par entité que l'application emploie (`src/common/entities.ts`) : celles du modèle, puis celles que les écrans nomment dans un autre modèle, puis celles que visent leurs relations, celles qu'elles héritent des entités qu'elles étendent comprises (PLAT-1459), les textes traduits du modèle en français et en anglais (`src/common/i18n.ts`), un fichier par module avec ses boutons et ses routes, un composant par composition que ses pages posent (`EntityScreen`), et le serveur de la plateforme, qui sert l'interface et les documents de chaque schéma. Le code est écrit avec l'API déclarative de la plateforme : `Page`, `Row`, `DocScope`, `DocList`, `DocForm`. Un élément qu'il ne sait pas écrire est sauté, et le journal du serveur le dit avec son chemin. Chaque projet porte un `Dockerfile` et un `.dockerignore` qui le construisent en image Docker (PLAT-1461).

  Le schéma d'une entité est celui que la plateforme construit au vol pour la même entité (PLAT-1460) :

  - un document qui nomme sa table (`table`) est gardé dans PostgreSQL : son schéma porte `storage: 'postgres'`, la table, la colonne de sa clé (`keyColumn`) quand il la nomme, et la colonne de chaque champ et de chaque lien dès qu'elle diffère de la clé du champ dans le schéma (`date_depart` est gardé sous `dateDepart` et lu dans la colonne `date_depart`). La table ne s'hérite pas, une colonne si ;
  - un document sans table reste dans MongoDB : son schéma ne change pas ;
  - les six sortes d'une base existante deviennent `long()`, `decimal({ scale })`, `localDate()`, `dateTime()`, `time()` et `json()` ;
  - les droits que l'entité déclare (`readRoles`… `deleteRoles`), ou la première de ses ancêtres qui en déclare, deviennent son `access`.

  Quand une entité nomme sa table, le `README.md` du projet produit gagne la section « An existing PostgreSQL base » : les tables, `POSTGRES_URL` et `POSTGRES_PUBLICATION`, les droits du compte de la base et le réglage qui fait voir en direct ce que d'autres applications écrivent.

`metaltura/examples` montre la pile sur le domaine d'une boutique (`BoutiquePlatform`), sur un modèle d'entités déjà dessinées (`BibliothequePlatform`) et sur une table PostgreSQL existante (`TrajetsPlatform`).

## Ce dont il dépend

| Dépôt | Ce qu'il en emploie | Ligne `depends` |
| --- | --- | --- |
| `metaltura/domain` | `Domain`, que lit `DomainToPlatform` | `metaltura/domain/Domain@v1.0.0` |
| `metaltura/application` | `DomainPattern` de `DomainToApplication`, qu'étendent `DomainToPlatform` et `EntitiesToPlatform`, et `CodePattern` d'`ApplicationToCode`, qu'étend `PlatformToCode` | `metaltura/application/DomainToApplication@v1.0.0`<br>`metaltura/application/ApplicationToCode@v1.0.0` |

Ces modèles système de Metaltura s'écrivent par leur nom seul :

- `Platform` ;
- `PlatformComponents` ;
- `Mapping` ;
- `FileSystem` ;
- `Diagram` ;
- `Uml`.

Un modèle de ce dépôt nomme un autre modèle du dépôt par son nom seul (`DomainToPlatform` nomme `EntitiesToPlatform`).

Un clone de ce dépôt seul s'ouvre dans Metaltura : il récupère `metaltura/domain` et `metaltura/application`, sans `metaltura/examples`.

## L'utiliser depuis un autre dépôt

Un modèle qui branche la pile sur un modèle d'entités déjà dessinées écrit dans sa liste `depends` :

```yaml
depends:
  - metaltura/platform/EntitiesToPlatform@v1.0.0
  - metaltura/platform/Compositions@v1.0.0
  - metaltura/platform/PlatformToCode@v1.0.0
```

C'est la liste de `BibliothequePlatform` dans `metaltura/examples`. Elle nomme en plus, par leur nom seul :

- `Mapping` ;
- son modèle d'entités, `Bibliotheque` ;
- `Platform` ;
- `FileSystem`.

Un modèle qui part d'un domaine nomme `metaltura/platform/DomainToPlatform@v1.0.0` à la place d'`EntitiesToPlatform`, et `metaltura/application/DomainToApplication@v1.0.0` en plus, comme `BoutiquePlatform`.

Metaltura clone chaque dépôt nommé à l'étiquette `v1.0.0`. Toute la librairie est à `v1.0.0` : Metaltura refuse qu'un même dépôt soit demandé à deux versions.

## D'où il vient

Ce dépôt vient du dépôt privé `Kogitec/simple-mda`, `main` à `96a1a75` (PLAT-1461), rangé par sujet par PLAT-1498 en un seul commit : ses modèles étaient sous `models/applications/platform/`. L'historique de ses fichiers reste dans `simple-mda`, archivé.
