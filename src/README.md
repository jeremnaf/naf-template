# Naftiko Custom Backstage Templates to create Capability component

This folder provides custom Backstage templates for Naftiko (only one for the moment: the capability template).

---

## Architecture

```
templates/
├── README.md
├── capability.yml
└── skeletons/capabilities/
    ├── ${{ values.name }}.naftiko.yml
    └── catalog-info.yaml
```

---

## Installation

### 1. Push template on GitHub

Templates must be hosted on a public GitHub repository to allow `fetch:template` to resolve relative paths to skeletons. You can only push the folder `templates`.

### 2. Import the template configuration file in your Backstage catalog

In `app-config.yaml` :

```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/<org>/<repo>/blob/main/src/templates/capability.yml
      rules:
        - allow: [Template]
```

> ⚠️ The `target` field must be the URL **GitHub** of `capability.yml` (not a local path)
> to allow relative URLs `./skeletons/` to be resolved by Backstage.

### 3. Configure GitHub integration

In `app-config.yaml` :

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

> ⚠️ Be sure to have a GitHub Private Access Token set as env variable GITHUB_TOKEN. This PAT must have right to read/write on repo.

### 4. Backstage component persistence

By default, the Backstage application is configure with an in-memory database

In `app-config.yaml` :

```yaml
backend:
  database:
    client: better-sqlite3
    connection: ':memory:'
```

In this case, after adding a component with the Naftiko custom Backstage template, when re-starting the backstage application, the component won't be in the catalog anymore.

If you want to persist it, modify the configuration in your `app-config.yaml`, or `app-config.local.yaml` (for local tests only)

```yaml
backend:
  database:
    client: better-sqlite3
    connection:
      directory: ./db-data
```

This will create a `packages/backend/db-data` folder where data will be persisted between application restarts. Don't forget to add it to your .gitignore file.

---

## Execution flow

Once the Naftiko custom template is well configured in your Backstage application:

```
The user clicks on Create
        │
        ▼
The user chooses the Naftiko Capability template
        │
        ▼
The user complete the form and validate
        │
        │  the corresponding skeletons files are fetched from the template GitHub repo
        |  nunjucks key words are interpreted (condition, variable values...)
        |  the results files are push to a new GitHub repo
        |  a new service component linked to this new repo is added in the backstage catalog
        ▼
Summary informations are displayed to the user (link to the new repo...).
```

---
