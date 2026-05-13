# Slacque — Design Spec

**Date** : 2026-05-13
**Auteur** : Rashad Karanouh (avec Claude)
**Statut** : Approuvé pour implémentation

## 1. Contexte et objectif

À chaque fois qu'on rejoint un nouveau Slack, les couleurs par défaut sont ternes. L'objectif est un outil qui, à partir d'un logo, produit une palette Slack personnalisée prête à coller dans Préférences → Thèmes → Personnaliser → Import.

Slack accepte une chaîne de **4 hex codes** au format `#RRGGBB, #RRGGBB, #RRGGBB, #RRGGBB` et adapte le contraste en interne ("We'll adapt the colors as best we can to preserve contrast"). On n'a donc pas à viser un mapping fonctionnel parfait — juste à fournir 4 couleurs cohérentes que Slack saura exploiter.

## 2. Forme retenue

**Plugin Claude Code**, structuré autour d'une slash command et d'une skill. Pas de code exécutable, pas de dépendances : toute la "logique" est portée par les prompts qui guident Claude (vision native + recherche web optionnelle).

### 2.1 Pourquoi pas un CLI standalone

- Claude a déjà la vision intégrée → pas besoin de packager un modèle ou d'appeler une API vision externe
- L'identification de la couleur de marque dans un logo demande du jugement (ignorer le blanc du fond, le noir du texte, choisir la teinte dominante saturée) — c'est exactement ce que Claude fait bien
- La recherche brand-book optionnelle (cf. §6.2) tire parti de `WebSearch` / `WebFetch` qui sont à portée de main dans Claude Code

### 2.2 Pourquoi slash command + skill et pas l'un des deux

- Slash command seul : oblige à mettre toute la connaissance dans le fichier de commande, pas réutilisable depuis une conversation normale
- Skill seule : pas de point d'entrée explicite, moins découvrable
- Les deux ensemble : la commande est un trigger fin qui invoque la skill ; la skill peut aussi s'activer auto sur "génère-moi un thème Slack à partir de ce logo" dans une conversation libre. C'est le pattern idiomatique des plugins Claude Code.

## 3. Layout du plugin

```
slacque/
├── .claude-plugin/
│   └── plugin.json              # manifeste (nom, version, description)
├── commands/
│   └── slacque.md               # slash command thin : /slacque <path>
├── skills/
│   └── slacque/
│       └── SKILL.md             # connaissance complète (extraction + slots + template + raffinement)
├── docs/
│   └── superpowers/specs/
│       └── 2026-05-13-slacque-design.md  # ce document
└── README.md
```

Le dossier actuel s'appelle `slack-colors-designer/` — à renommer en `slacque/` lors de la première phase d'implémentation (ou pas, c'est purement cosmétique côté FS).

Installation : `/plugin install /chemin/vers/slacque` (ou via le menu plugins de Claude Code).

## 4. La slash command `/slacque`

Fichier `commands/slacque.md`. Son seul rôle : récupérer le logo et déléguer à la skill.

```markdown
---
description: Génère un thème Slack à partir d'un logo
argument-hint: <chemin du logo> (ou colle/drag l'image)
---

Génère un thème Slack personnalisé à partir du logo fourni.

Source du logo :
- Si $ARGUMENTS contient un chemin de fichier, lis cette image
- Sinon, si une image est attachée à ce message, utilise-la
- Sinon, demande à l'utilisateur de fournir le logo (drag-and-drop, paste, ou path)

Une fois le logo identifié, utilise la skill `slacque`.
```

Comportements supportés :
- `/slacque ~/Downloads/acme-logo.png` — path explicite
- `/slacque ` puis drag-and-drop d'une image — drag insère le path, l'argument est rempli
- `/slacque` + Cmd+V d'une capture — l'image est attachée au message, pas d'argument
- `/slacque` seul — Claude demande la source

## 5. Le contrat de sortie Slack (4 slots)

Décodé en croisant la chaîne `#611F69, #39063A, #20A271, #C474D3` avec les chips de l'UI Slack :

| # | Slot | Rôle dans le rendu Slack |
|---|---|---|
| 1 | **System navigation** | Brand primaire profonde — domine la sidebar |
| 2 | **Window background** | Encore plus sombre — couches de fond/contraste structurel |
| 3 | **Presence indication (source)** | Vert "online" — Slack en dérive une version plus claire |
| 4 | **Notifications / Selected items (source)** | Accent secondaire moyen — Slack en dérive badges et highlights clairs |

**Format de la chaîne** : 4 hex en majuscules ou minuscules, séparés par `, ` (virgule + espace), pas de saut de ligne.

Exemple : `#611F69, #39063A, #20A271, #C474D3`

## 6. La skill `slacque`

Fichier `skills/slacque/SKILL.md`. C'est le cœur du plugin. Quatre blocs :

### 6.1 Extraction depuis le logo (vision)

Règles que la skill impose à Claude :
- Ignorer les pixels quasi-blancs, quasi-noirs et gris désaturés (fond, texte, ombres — pas la marque)
- Parmi les pixels saturés restants, identifier la **teinte dominante** = couleur de marque primaire
- Si le logo contient une **seconde teinte saturée distincte**, la noter comme accent candidat (alimente le slot #4)
- Sur les gradients, prendre la teinte centrale (pas les extrêmes)
- Si le logo est monochrome, dériver les autres slots algorithmiquement (cf. §6.4)

### 6.2 Recherche brand-book optionnelle (Option 3 — propose & valide)

Après l'analyse vision initiale, Claude tente d'**identifier la marque** (texte dans le logo, forme distinctive, monogramme reconnaissable). Si une marque est plausiblement reconnue :

> "Je pense que ce logo est de **\<marque\>**. Je peux aller vérifier sur leur brand site (ex: `brand.\<marque\>.com`, `\<marque\>.com/brand`, `design.\<marque\>.com`) pour avoir des couleurs officielles ? (oui/non)"

Si **oui** :
1. `WebSearch` du genre `"<marque> brand guidelines colors"` ou `"<marque> brand book hex"`
2. `WebFetch` sur les URLs prometteuses
3. Extraire les hex officiels visibles dans la page
4. Si succès → utiliser ces hex comme source de vérité pour la palette
5. Si échec (pas de brand site trouvé, ou pas de hex extractibles) → fallback silencieux sur vision-only avec mention : *"Pas trouvé de brand book public, je m'appuie sur le logo seul."*

Si **non** ou marque non-identifiée → directement vision-only.

### 6.3 Construction de la palette (4 slots)

À partir de la couleur de marque primaire (vision) ou des couleurs officielles (brand site) :

- **Slot #1 (System navigation)** : couleur de marque en version sombre/saturée. Si la marque est déjà sombre, prendre tel quel ; sinon, assombrir tout en gardant la saturation.
- **Slot #2 (Window background)** : version encore plus sombre du #1 (même hue ou hue voisine). Doit être perceptiblement plus sombre que #1.
- **Slot #3 (Presence indication)** :
  - Par défaut, un vert saturé proche du vert Slack natif
  - Si la marque a une couleur secondaire verte, l'utiliser à la place
  - Garder le slot vert pour respecter la convention "online = vert"
- **Slot #4 (Notifications / Selected items)** : accent secondaire de la marque, ton moyen-clair, saturé. C'est ce qui va vibrer dans les badges. Si la marque a une seconde couleur saturée, c'est elle ; sinon, dériver par complémentaire ou triade du #1.

### 6.4 Cas du logo monochrome (une seule couleur saturée)

- #1 = couleur saturée extraite
- #2 = version assombrie du #1 (-30 à -50% lightness)
- #3 = vert présence par défaut (proche du vert Slack natif)
- #4 = complémentaire (rotation +180°) ou triade (rotation +120°) du #1, ton moyen-clair

### 6.5 Gabarit de sortie

Format exact que la skill impose à Claude après chaque analyse ou raffinement :

````markdown
## Thème Slack pour [marque ou description]

```
#xxx, #xxx, #xxx, #xxx
```

| # | Slot | Hex | Note |
|---|---|---|---|
| 1 | System navigation | `#xxx` | description courte |
| 2 | Window background | `#xxx` | description courte |
| 3 | Presence indication | `#xxx` | description courte |
| 4 | Notifications | `#xxx` | description courte |

*Logique* : 1-2 phrases sur la lecture du logo (ou source brand book) et les choix structurants.
````

### 6.6 Boucle de raffinement

Après la première palette, la skill instruit Claude à reconnaître des commandes de raffinement en langage naturel :

| Tu dis | Claude fait |
|---|---|
| "plus sombre" / "plus clair" | Décale la luminosité des slots #1 et #2 |
| "moins saturé" / "plus saturé" | Ajuste la saturation, surtout sur #4 |
| "change l'accent" / "badge plus chaud/froid" | Re-choisit le slot #4 |
| "un autre angle" / "ré-essaie" | Repart du logo, interprète différemment la couleur de marque |
| "thème clair" / "thème sombre" | Inverse la stratégie structurelle |
| "garde tout sauf le mention" | Conserve #1, #2, #3, change uniquement #4 |

À chaque itération, Claude **re-produit la chaîne complète + la table** au format §6.5. Pas de diff partiel.

## 7. Validation

**Pré-analyse** :
- Pas de logo (pas d'argument, pas d'image attachée) → Claude demande
- Path donné mais fichier inexistant → Claude le signale et demande
- Fichier qui n'est pas une image lisible → Claude le signale

**Post-production** :
- La chaîne contient exactement 4 hex de format `#RRGGBB` (case-insensitive)
- Les 4 couleurs ne sont pas strictement identiques
- **Pas** de check WCAG — Slack adapte le contraste en interne

## 8. Non-goals (explicitement hors scope MVP)

- Input par URL (de site ou d'image) — l'utilisateur télécharge l'image localement d'abord
- Plusieurs variantes (dark / light / vibrant) produites en parallèle — une seule palette par run, raffinement conversationnel
- Mockup visuel de la sidebar Slack — la table de vérif suffit
- Distribution via marketplace public — install local uniquement pour démarrer
- Build/tests automatisés — c'est du markdown qui guide un LLM, pas du code à tester

## 9. Manifeste du plugin

Fichier `.claude-plugin/plugin.json` minimal :

```json
{
  "name": "slacque",
  "version": "0.1.0",
  "description": "Génère un thème Slack personnalisé à partir d'un logo"
}
```

## 10. README

Court fichier à la racine, contenant :
- Une phrase sur ce que fait le plugin
- L'install : `/plugin install /chemin/vers/slacque`
- L'usage nominal : `/slacque <chemin>` ou drag-and-drop
- Mention du flow de raffinement conversationnel
- Le format des 4 slots Slack (pour s'en rappeler dans 6 mois)
