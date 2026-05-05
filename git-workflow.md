# Gitflow — Réponses aux questions et méthodologie custom

## Partie 1 — Questions

### Mise en œuvre technique

#### T1 — Quel est le protocole de communication utilisé en standard par le client Git ?

Le protocole standard est **SSH** (port 22). C'est celui utilisé par défaut pour les opérations authentifiées de `push`/`pull`/`fetch` sur les plateformes type GitHub, GitLab, Bitbucket, Azure DevOps. Il offre chiffrement, authentification par paire de clés et bonnes performances.

> Git supporte aussi nativement son propre protocole `git://` (port 9418), mais celui-ci n'est plus utilisé en pratique car non chiffré et sans authentification.

#### T2 — Quel est le protocole de communication utilisé en secours par le client Git ?

Le protocole de secours est **HTTPS** (port 443). Il est utilisé quand SSH n'est pas disponible (réseaux d'entreprise filtrés, proxys, postes sans clé SSH configurée). L'authentification se fait alors par token (PAT) ou via un credential manager. C'est plus lent que SSH sur de gros transferts mais ça passe partout.

#### T3 — Est-il possible de synchroniser un dépôt local avec plusieurs dépôts distants ? Si oui, comment ?

Oui, c'est tout à fait possible. Un dépôt local peut avoir autant de remotes que nécessaire — chacun identifié par un nom unique (`origin` est juste la convention par défaut).

```bash
# Voir les remotes existants
git remote -v

# Ajouter un second remote
git remote add upstream https://github.com/autre-org/projet.git

# Récupérer depuis ce remote
git fetch upstream

# Pousser sur un remote spécifique
git push upstream develop
```

**Cas d'usage typiques** : fork synchronisé avec son upstream, miroir d'un dépôt sur deux plateformes (GitHub + Bitbucket), backup sur un serveur interne en parallèle de l'origin.

#### T4 — Effet de `git revert` sur un commit ajouté par mégarde

`git revert <hash>` **crée un nouveau commit** dont le contenu annule les modifications du commit ciblé. L'historique n'est pas réécrit : le commit fautif reste visible, et un nouveau commit "inverse" vient s'ajouter par-dessus.

```bash
git revert a1b2c3d
# → crée un commit "Revert "ADD: fichier ajouté par mégarde""
```

**Différence clé avec `git reset`** : `revert` est non-destructif et safe sur une branche partagée. `reset` réécrit l'historique et casse les copies des autres devs si la branche est déjà poussée.

> ⚠️ Attention : `revert` annule les changements **logiques** du commit, mais le fichier reste présent dans l'historique. Pour purger réellement un fichier (mot de passe, secret, gros binaire), il faut `git filter-repo` ou `BFG Repo-Cleaner` + force-push coordonné.

---

### Mise en œuvre méthodologique

#### M1 — Différence principale entre `git merge` et `git rebase` en termes de gestion de l'historique

| Aspect | `git merge` | `git rebase` |
|---|---|---|
| **Historique** | Préservé tel quel, avec un commit de merge | Réécrit, linéaire |
| **Traçabilité** | On voit clairement la branche d'origine | L'origine de la branche disparaît |
| **Risque** | Aucun sur branche partagée | Casse les autres copies si rebase d'une branche poussée |
| **Lisibilité** | Historique en "diamants", parfois touffu | Historique propre, lecture séquentielle |

**En résumé** : `merge` conserve la vérité chronologique du travail, `rebase` produit un historique propre mais fictif. La règle d'or : `rebase` en local sur ta branche perso avant PR, `merge` pour intégrer dans une branche partagée.

#### M2 — Autre méthode de branching abordée — en quoi diffère-t-elle ?

L'autre méthode abordée est **GitHub Flow**. Ses différences avec le Gitflow custom :

- **Une seule branche permanente** : `main`. Pas de `develop`, pas de `release`, pas de `hotfix` constante.
- **Workflow ultra-simple** : on branche depuis `main`, on développe, on ouvre une PR, on merge dans `main`, on déploie immédiatement.
- **Pas de notion de version préparée** : chaque merge sur `main` est potentiellement déployable en production.
- **Adapté au continuous deployment** sur des produits web/SaaS, pas aux logiciels avec cycle de release formel.

À l'inverse, le Gitflow (et a fortiori la version custom présentée ici) sépare clairement développement, préparation de livraison et production — utile dès qu'il y a une recette, des versions taguées, ou plusieurs environnements. Là c'est pertinent et destinée à une équipe.

#### M3 — Objectif concret d'une Pull Request / Merge Request en travail d'équipe

Une PR n'est pas qu'un bouton "merge", c'est un **point de synchronisation et de contrôle qualité avant intégration**. Concrètement elle sert à :

- **Faire relire le code** par un autre membre — détection de bugs, dette technique, écarts de convention
- **Discuter des choix techniques** via les commentaires de revue, avant qu'ils ne deviennent du legacy
- **Documenter le changement** : description, ticket lié, captures, contexte — utile 6 mois plus tard
- **Déclencher la CI/CD** automatiquement (tests, linting, build, scans de sécurité)
- **Tracer les responsabilités** : qui a écrit, qui a relu, qui a approuvé
- **Diffuser la connaissance** : le reviewer apprend ce qui change dans le projet, ça réduit le bus factor

C'est aussi une garde-fou contre les pushs directs sur des branches protégées.

---

## Partie 2 — Méthodologie Gitflow remodelée (avec code review et PR croisées)

> Cette version intègre les bonnes pratiques d'équipe : **PR systématiquement assignée à un autre membre**, code review obligatoire, et levée de l'exception "auteur = reviewer" qui était tolérée dans la version initiale.

### 1. Branches permanentes

| Branche | Rôle | Protégée |
|---|---|---|
| `main` | Production stable, image fidèle de ce qui tourne en prod | ✅ Oui |
| `release` | Préproduction, recette, versionnée à chaque livraison | ✅ Oui |
| `develop` (ou `dev`) | Intégration continue du développement actif | ✅ Oui |
| `hotfix` | Branche permanente, copie conforme de `main`, prioritaire pour les correctifs urgents prod | ✅ Oui |

**Branche protégée** = pas de push direct, merge uniquement via PR validée.

### 2. Convention de nommage des branches

Format : `fromBranchSource/[auteur/]type/nomDescriptif` en camelCase.

| Type | Exemple | Source |
|---|---|---|
| Feature | `fromDev/feature/userLogin` | `develop` |
| Feature avec auteur (optionnel) | `fromDev/rMardargent/feature/userLogin` | `develop` |
| Hotfix de release | `fromRelease/hotfix/fixEmailBug` | `release` |
| Hotfix de prod (urgence) | `fromHotfix/hotfix/fixCriticalAuth` | `hotfix` |

### 3. Convention de messages de commit

Préfixer le message par le type d'opération en majuscules :

| Préfixe | Usage |
|---|---|
| `ADD:` | Ajout de code, fichier, fonctionnalité |
| `FIX:` | Correction de bug |
| `CLEAN:` | Nettoyage (commentaires, code mort, formatage) |
| `WIP:` | Work in progress, commit intermédiaire |
| `REFACTO:` | Refactoring sans changement fonctionnel |
| `DOC:` | Mise à jour de documentation |
| `TEST:` | Ajout ou modification de tests |

Exemple : `ADD: formulaire de connexion utilisateur avec validation email`

### 4. Cycle de vie d'une feature — workflow détaillé

```
develop ──► fromDev/feature/xxx ──► PR + review ──► develop
                                       │
                                       └─► CI passe + 1 approbation reviewer ≠ auteur
```

**Étapes obligatoires :**

1. **Création** depuis `develop` à jour
   ```bash
   git checkout develop
   git pull
   git checkout -b fromDev/feature/userLogin
   ```

2. **Développement local** avec commits préfixés (`ADD:`, `FIX:`…)

3. **Push de la branche** sur le remote
   ```bash
   git push -u origin fromDev/feature/userLogin
   ```

4. **Ouverture de la PR** vers `develop` avec :
   - Titre clair décrivant l'apport
   - Description : contexte, ce qui change, comment tester, ticket lié
   - **Reviewer assigné = un autre membre de l'équipe** (jamais soi-même)
   - Labels appropriés (feature, bug, refacto…)

5. **Code review** par le reviewer assigné (cf. section 6)

6. **Corrections éventuelles** par l'auteur, push sur la même branche → la PR se met à jour automatiquement

7. **Approbation + CI verte** → merge par l'auteur (ou le reviewer)

8. **Conservation de la branche 14 jours** après merge avant suppression manuelle, pour permettre un retour rapide si besoin

### 5. Code review — règles d'équipe

**Règle absolue : l'auteur de la PR ≠ le reviewer.** L'exception "auteur = reviewer" de la version initiale est levée.

**Ce que le reviewer vérifie :**

- Le code fait ce que la PR annonce
- Pas de code mort, pas de `console.log` oublié, pas de secret hardcodé
- Conventions de nommage (branches, commits, variables) respectées
- Tests présents et pertinents si la zone modifiée le justifie
- Pas de régression évidente
- Lisibilité, complexité raisonnable
- Documentation à jour si l'API publique change

**Ce que le reviewer ne fait pas :**

- Réécrire le code à la place de l'auteur
- Bloquer sur des préférences personnelles (sauf si conventions équipe)
- Approuver à la chaîne sans lire — "LGTM" sans justification = pas une review

**Délai cible de review :** 24 h ouvrées. Au-delà, l'auteur peut relancer ou demander à un autre reviewer.

**En cas de désaccord :** discussion en commentaire de PR, et si non résolue, escalade en sync rapide ou décision de l'arbitre technique.

### 6. Versionnement (semver `X.Y.Z`)

| Composant | Incrément quand… | Exemple |
|---|---|---|
| `X` (majeur) | Changement majeur, breaking change, fonctionnalité structurante | `1.0.0` → `2.0.0` |
| `Y` (mineur) | Nouvelle release standard avec features | `1.1.0` → `1.2.0` |
| `Z` (correctif) | Hotfix sur `release` ou `main` | `1.2.0` → `1.2.1` |

Les tags se posent **systématiquement sur `release`** (pas sur `develop`) au moment du merge `develop` → `release`, et lors de chaque hotfix corrigé en release.

```bash
git tag -a 1.2.0 -m "Release 1.2.0"
git push origin 1.2.0
```

### 7. Préparation d'une release

```bash
git checkout develop && git pull
# Ouvrir une PR develop → release, assigner un reviewer
# Après merge :
git checkout release && git pull
git tag -a 1.2.0 -m "Release 1.2.0"
git push origin 1.2.0
```

### 8. Gestion des hotfixes

#### Hotfix en `release` (bug détecté en recette)

```bash
git checkout release
git checkout -b fromRelease/hotfix/fixEmailBug
# correction + commits
git push -u origin fromRelease/hotfix/fixEmailBug
```

PR vers `release` → review → merge → tag correctif (`1.2.1`) → **backmerge dans `develop`** via PR.

#### Hotfix en `main` (bug critique en production)

`hotfix` est en permanence une copie conforme de `main`. En cas de bug critique :

```bash
git checkout hotfix && git pull
git checkout -b fromHotfix/hotfix/fixCriticalAuth
# correction + commits
git push -u origin fromHotfix/hotfix/fixCriticalAuth
```

PR vers `hotfix` → review (même en urgence, on ne saute pas la review) → merge → tag correctif → **PR de `hotfix` vers `main`** → puis **backmerge dans `release` ET `develop`** pour ne pas perdre le correctif aux prochaines livraisons.

### 9. Passage en production

```
release ──► PR vers main ──► review ──► merge ──► déploiement prod
                                          │
                                          └─► hotfix réaligné sur main
```

Après merge dans `main`, mettre à jour `hotfix` pour qu'elle reste copie conforme :

```bash
git checkout hotfix
git merge main
git push origin hotfix
```

### 10. Récap des règles non négociables

- ✅ Toute fusion passe par une PR
- ✅ Toute PR a un reviewer **différent de l'auteur**
- ✅ La CI doit être verte avant merge
- ✅ Branches permanentes (`main`, `release`, `develop`, `hotfix`) protégées
- ✅ Tags semver posés sur `release`
- ✅ Backmerge systématique après hotfix
- ✅ Conventions de nommage (branches + commits) appliquées sans exception
- ❌ Pas de push direct sur les branches protégées
- ❌ Pas d'auto-approbation de PR
- ❌ Pas de force-push sur une branche partagée

### 11. Schéma de synthèse du flux

```
                  ┌────────────────────────────────────┐
                  │              hotfix                │ ◄─── copie conforme de main
                  └─────────────┬──────────────────────┘
                                │ PR + review
                                ▼
   main ◄──── PR ──── release ◄──── PR ──── develop ◄──── PR ──── fromDev/feature/xxx
    ▲                    │                      ▲
    │                    │ tag X.Y.Z            │
    │                    ▼                      │
    │              fromRelease/hotfix/xxx ──────┘ (backmerge)
    │
    └── déploiement prod
```