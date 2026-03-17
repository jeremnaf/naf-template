# Custom Templates Plugin – Backstage

Plugin Backstage fournissant un Software Template personnalisé. Utilise
uniquement des **actions natives** (`fetch:template`, `publish:github`) —
**aucun plugin backend requis**.

---

## Architecture

```
backstage-plugin-custom-templates/
└── src/templates/
    ├── template.yaml                  ← Définition du Software Template
    └── skeletons/
        ├── empty/
        │   └── ${{ values.name }}-config.yml
        ├── mcp/
        │   └── ${{ values.name }}-config.yml
        ├── rest/
        │   └── ${{ values.name }}-config.yml
        └── skills/
            └── ${{ values.name }}-config.yml
```

---

## Pourquoi `fetch:template` suffit

| Besoin | Solution native |
|--------|----------------|
| Substituer `${{ values.name }}` | Moteur Nunjucks intégré à `fetch:template` |
| Choisir le bon fichier selon l'adaptateur | Étapes conditionnelles `if:` dans `template.yaml` |
| Pousser sur GitHub | Action native `publish:github` |
| Plugin backend custom | ✗ Inutile |

`fetch:template` résout l'URL `./skeletons/<adapter>/` **relativement à l'emplacement
du `template.yaml`** dans le dépôt GitHub. Les noms de fichiers contenant
`${{ values.name }}` sont également traités par le moteur de templating.

---

## Flux d'exécution

```
Utilisateur choisit adapter=MCP, name=my-service
        │
        ▼
  Étape fetch-empty  → ignorée  (if: false)
  Étape fetch-mcp    → exécutée (if: true)
        │  fetch:template lit skeletons/mcp/${{ values.name }}-config.yml
        │  remplace ${{ values.name }} → "my-service"
        │  écrit  my-service-config.yml  dans le workspace
        ▼
  Étape publish:github
        │  pousse my-service-config.yml sur le repo GitHub choisi
        ▼
  Lien vers le dépôt affiché à l'utilisateur
```

---

## Installation

### 1. Pousser le plugin sur GitHub

Le plugin doit être hébergé sur GitHub pour que `fetch:template` puisse
résoudre les chemins relatifs vers les skeletons.

```bash
git add .
git commit -m "feat: add custom-adapter backstage template"
git push origin main
```

### 2. Importer le template dans le catalog Backstage

Dans `app-config.yaml` :

```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/<org>/<repo>/blob/main/src/templates/template.yaml
      rules:
        - allow: [Template]
```

> ⚠️ Le `target` doit pointer vers l'URL **GitHub** du `template.yaml` (pas un chemin local)
> pour que les URLs relatives `./skeletons/` soient résolues correctement par Backstage.

### 3. Configurer l'intégration GitHub

Dans `app-config.yaml` :

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

scaffolder:
  defaultAuthor:
    name: Backstage Scaffolder
    email: scaffolder@example.com
  defaultCommitMessage: "feat: initial scaffold from Backstage"
```

### 4. Aucune modification du backend requise ✓

Pas de `backend.add(...)`, pas de dépendance supplémentaire.

---

## Utilisation

1. Ouvrez Backstage → menu **Create**
2. Sélectionnez **"Custom Adapter Component"**
3. Renseignez **Name** et **Adapter**
4. Choisissez le dépôt GitHub cible
5. Cliquez **Create** — le fichier `<name>-config.yml` est poussé sur GitHub

---

## Ajouter un nouvel adaptateur

1. Créer `src/templates/skeletons/<new-adapter>/${{ values.name }}-config.yml`
2. Ajouter la valeur dans l'`enum` du `template.yaml`
3. Ajouter une étape `fetch:template` avec le `if:` correspondant
4. Pousser sur GitHub — le catalog se met à jour au prochain refresh
