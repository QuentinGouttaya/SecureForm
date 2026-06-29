# Dynamic Form Engine — Svelte 5 / Rust / PostgreSQL

> Moteur de formulaires métier piloté par configuration (JSON/AST), conçu pour les secteurs BTP et Audit avec des exigences strictes de conformité **RGPD** et de traçabilité **NIS2-compatible**.

***

## 📖 Sommaire

1. [Vision Architecturale](#vision-architecturale)
2. [Fonctionnalités Principales](#fonctionnalités-principales)
3. [Stack Technique](#stack-technique)
4. [Modèle de Données & Conformité](#modèle-de-données--conformité)
5. [Sécurité — AST & Zero-Trust](#sécurité--ast--zero-trust)
6. [Event Sourcing & CQRS](#event-sourcing--cqrs)
7. [Snapshots & Performance](#snapshots--performance)
8. [Démarrage Rapide](#démarrage-rapide)
9. [Structure du Projet](#structure-du-projet)

***

## 🏛 Vision Architecturale

Ce projet implémente une **architecture hybride Event-Driven / CQRS** centrée autour de `JSONB` PostgreSQL. Il abandonne délibérément les paradigmes relationnels classiques (EAV) et les ORMs au profit d'une stack légère, typée statiquement, et orientée performance et conformité.

### Principes fondamentaux

- **Performance avant vélocité** : la complexité initiale est assumée pour garantir l'intégrité et les performances à long terme.
- **SSOT strict** : PostgreSQL est l'unique source de vérité. Rust valide, Svelte rend. Aucun état n'est autorisé à exister uniquement côté client.
- **Append-only by design** : aucun `UPDATE` ou `DELETE` destructif sur les données métier. Toute mutation est un événement immutable.
- **Conformité by design** : le chiffrement PII, la traçabilité et l'audit trail ne sont pas des couches ajoutées — ils sont structurels.

### Périmètre NIS2 — Clarification importante

> ⚠️ La Directive NIS2 (EU 2022/2555) ne couvre **pas** le secteur BTP dans ses Annexes I et II. Une entité BTP n'entre dans le scope NIS2 qu'aux conditions cumulatives : >50 salariés, CA >10M€, **ET** opération dans un secteur listé (énergie, transport, santé, infrastructure numérique, etc.).
>
> Ce projet implémente une architecture **compatible avec les exigences NIS2** (audit trail, event sourcing, zero-trust) comme bonne pratique de sécurité, indépendamment de l'obligation légale.

***

## ✨ Fonctionnalités Principales

- **Moteur de Rendu Réactif** : génération dynamique des champs côté client sans rechargement, pilotée par les Runes Svelte 5 (`$state`, `$derived`).
- **Règles Conditionnelles via AST** : logique de visibilité et validation inter-champs exprimée en JSON (Abstract Syntax Tree). L'AST est **signé (HMAC-SHA256)** avant envoi au client et **revalidé depuis PostgreSQL** à chaque soumission — jamais depuis le payload client.
- **Double niveau d'AST** :
  - **AST Full** (Rust uniquement) — règles d'autorisation, seuils sensibles, logique compliance.
  - **AST Client** (projection filtrée par rôle JWT) — règles UX uniquement, aucune règle d'accès exposée.
- **Sécurité PII (RGPD)** : chiffrement symétrique par champ (AES-256-GCM) avec Key Rotation asynchrone. Le "Right to be Forgotten" est implémenté par **Crypto-shredding** (destruction de clé = destruction fonctionnelle de la donnée), reconnu défendable par l'EDPB.
- **Audit Trail NIS2-compatible** : base de données Event-Sourced (Append-only). Aucune modification destructive.
- **Zero-Trust** : JWT validé obligatoire sur chaque endpoint Axum. Aucun accès direct aux données brutes sans token valide.
- **Subscriptions temps réel** : utilisation de `query.live()` (SvelteKit 2.61+) pour les mises à jour de statut de soumission.

***

## 🛠 Stack Technique

### Versions cibles (juin 2026)

| Composant | Version | Notes |
|-----------|---------|-------|
| **PostgreSQL** | **18.4** | Version stable courante (EOL nov. 2030) |
| **Svelte 5 / SvelteKit** | **2.61+** | Runes stables, `query.live()` disponible |
| **Rust** | Edition **2024** | `async fn` dans les traits stable |
| **Axum** | **0.8.8** | Nouvelle syntaxe `/{id}` pour les path params |
| **SQLx** | **0.8.6** | Stable — éviter 0.9.0-alpha |
| **TypeScript** | **6.0** | Support natif dans le language server Svelte |

### Frontend

- [Svelte 5](https://svelte.dev/) — rendu dynamique via Runes (`$state`, `$derived`, `$effect`)
- [SvelteKit 2.61+](https://kit.svelte.dev/) — SSR, form actions, `query.live()` pour le temps réel
- TailwindCSS v4 — composants visuels
- Vitest & Playwright — tests UI

### Backend

- [Rust](https://www.rust-lang.org/) (Édition 2024)
- [Axum 0.8.8](https://github.com/tokio-rs/axum) — API Web asynchrone, syntaxe `/{id}`
- [SQLx 0.8.6](https://github.com/launchbadge/sqlx) — requêtes async typées sans ORM, compile-time checked
- `aes-gcm` — chiffrement applicatif AES-256-GCM

### Infrastructure

- [PostgreSQL 18.4](https://www.postgresql.org/) — JSONB, `JSON_TABLE`, `NOTIFY/LISTEN`
- [Docker & Testcontainers](https://testcontainers.com/) — tests d'intégration
- EVE-NG / KVM — simulation de topologie réseau isolée

***

## 🔒 Modèle de Données & Conformité

### Ségrégation Cryptographique des PII

Chaque donnée personnelle identifiée (PII) est stockée chiffrée. La clé de chiffrement est identifiée par `key_id` pour permettre la rotation et le crypto-shredding.

```json
{
  "chantier_id": "PRJ-999",
  "nom_employe_blesse": {
    "key_id": "v1",
    "ciphertext": "z8A9fBdG...",
    "nonce": "XyZ123..."
  }
}
```

> **Right to be Forgotten (Art. 17 RGPD)** : détruire la clé associée à `key_id` rend le `ciphertext` irrecoverable sans aucune modification de l'event store. L'EDPB reconnaît cette approche comme équivalent fonctionnel à l'effacement sous réserve d'un chiffrement robuste (AES-256-GCM satisfait cette condition).

> ⚠️ **Tension architecturale connue** : les métadonnées d'événements (`event_type: "BLESSURE_DECLAREE"`, horodatage, `aggregate_id`) restent visibles en clair dans l'event log même après crypto-shredding. Ces métadonnées peuvent elles-mêmes constituer des données personnelles. Ce point doit être adressé dans l'analyse DPIA avant mise en production.

### Schema des tables PostgreSQL

```sql
-- Event Store (append-only — jamais d'UPDATE/DELETE)
CREATE TABLE form_events (
    id            BIGSERIAL PRIMARY KEY,
    aggregate_id  UUID         NOT NULL,
    version       INT          NOT NULL,
    event_type    TEXT         NOT NULL,
    schema_version INT         NOT NULL DEFAULT 1, -- migration future
    payload       JSONB        NOT NULL,
    created_at    TIMESTAMPTZ  DEFAULT now(),
    UNIQUE (aggregate_id, version)  -- optimistic concurrency
);

-- Snapshots pour optimisation des replays
CREATE TABLE form_snapshots (
    aggregate_id   UUID PRIMARY KEY,
    version        INT         NOT NULL,
    state          JSONB       NOT NULL,
    schema_version INT         NOT NULL DEFAULT 1,
    created_at     TIMESTAMPTZ DEFAULT now()
);

-- Vue matérialisée pour Power BI (JSON_TABLE — PG 17+)
CREATE MATERIALIZED VIEW form_responses_flat AS
SELECT
    e.aggregate_id,
    jt.*
FROM form_events e,
JSON_TABLE(
    e.payload, '$'
    COLUMNS (
        chantier_id   TEXT PATH '$.chantier_id',
        type_incident TEXT PATH '$.type_incident',
        gravite       INT  PATH '$.gravite'
        -- champs PII exclus de cette vue
    )
) AS jt
WHERE e.event_type = 'FORM_SUBMITTED';

-- Refresh sans lock (Power BI peut lire pendant le refresh)
-- REFRESH MATERIALIZED VIEW CONCURRENTLY form_responses_flat;
```

***

## 🛡 Sécurité — AST & Zero-Trust

### Architecture AST à double niveau

```
[PostgreSQL]
  └── form_definitions (AST complet, versionné, immuable)
        │
        ▼
[Rust / Axum — GET /forms/:id]
  ├── Charge l'AST depuis PG (jamais depuis le client)
  ├── Filtre les nœuds selon rôle JWT (projection client)
  ├── Signe l'AST projeté : HMAC-SHA256(ast_content, SERVER_SECRET)
  └── Envoie { ast, sig } au client
        │
        ▼
[Svelte 5]
  ├── Interprète l'AST pour le rendu UX uniquement
  ├── Validation de surface (feedback utilisateur)
  └── Soumet { form_data, ast_sig } à Rust
        │
        ▼
[Rust / Axum — POST /forms/:id/submit]
  ├── Vérifie ast_sig → rejet immédiat si tampered
  ├── Recharge l'AST depuis PG (SSOT — ignore l'AST du client)
  ├── Revalide toutes les règles métier
  ├── Vérifie ACL par champ (indépendant de l'AST)
  └── Persiste dans l'event store
```

### Protections implémentées

- **Signature HMAC** : toute modification de l'AST côté client → signature invalide → rejet sans évaluation.
- **Whitelist des field IDs** : chaque `field` référencé dans l'AST est validé contre la liste des champs autorisés pour ce formulaire — prévient les path traversal (`../../admin_secret`).
- **Depth & complexity limits** : max 10 niveaux de nesting AST, max 100 nœuds par arbre — prévient les attaques par récursion.
- **Type Rust exhaustif** : l'AST est désérialisé via `enum NodeType { Condition, Operator, Effect, ... }` — toute règle inconnue est rejetée à la désérialisation, pas à l'exécution.
- **CSP strict** (SvelteKit) : `script-src 'self' 'nonce-{random}'`, `form-action 'self'`.
- **Rate limiting** par endpoint Axum sur les soumissions.
- **`form_instance_id` + nonce** : prévient les replay attacks.

***

## ⚡ Event Sourcing & CQRS

### Séparation Command / Query

```
Client → [Command Handler] → Event Store (append-only) → PostgreSQL write
Client → [Query Handler]  → Vue matérialisée            → PostgreSQL read
Power BI                  → JSON_TABLE sur vue mat.     → PostgreSQL read
```

- **Command** : mute l'état, retourne uniquement un ack + `event_id`.
- **Query** : lit depuis les projections — jamais depuis l'event store brut.
- **Eventual consistency** : après une Command, la vue matérialisée peut être légèrement en retard. Svelte doit implémenter un **optimistic update** côté client pendant la propagation.

### Gestion des projections défaillantes

Chaque projecteur doit être **idempotent**. En cas d'erreur de reconstruction d'une vue matérialisée :

1. Logger l'erreur dans une **Dead Letter Queue** (table `projection_errors`).
2. Rejouer les events depuis la dernière version correcte connue.
3. Les données métier (event store) ne sont jamais corrompues — seule la projection est à reconstruire.

### Versioning des events

Chaque event porte un `schema_version`. En cas de changement de structure :

```rust
pub fn upcast(raw: RawEvent) -> FormEvent {
    match raw.schema_version {
        1 => upcast_v1_to_current(raw),
        2 => serde_json::from_value(raw.payload).unwrap(),
        _ => panic!("Unknown event schema version: {}", raw.schema_version),
    }
}
```

***

## 📸 Snapshots & Performance

### Principe

Sans snapshot : reconstruction = replay de **tous** les events depuis l'origine.
Avec snapshot : reconstruction = snapshot + events **postérieurs uniquement**.

### Seuil recommandé

- `SNAPSHOT_THRESHOLD = 50` events — ajustable selon le profil de charge.
- Déclenchement : `aggregate.version % SNAPSHOT_THRESHOLD == 0` après chaque Command.

### Stratégies disponibles

| Stratégie | Usage | Implémentation |
|-----------|-------|----------------|
| Every N events | Cas général | `version % N == 0` |
| Time-based | Agrégats à faible event rate | Job CRON Tokio |
| On read if stale | Agrégats lus fréquemment | Si events rejoués > seuil → snapshot post-load |
| Manual / admin | Migration, incident | `POST /admin/snapshot/{id}` |

### Schema snapshot (PostgreSQL)

```sql
CREATE TABLE form_snapshots (
    aggregate_id   UUID PRIMARY KEY,
    version        INT         NOT NULL,
    state          JSONB       NOT NULL,  -- FormAggregate sérialisé
    schema_version INT         NOT NULL DEFAULT 1,
    created_at     TIMESTAMPTZ DEFAULT now()
);
```

> **Règle absolue** : les events ne sont **jamais supprimés** même si un snapshot existe. Le snapshot est un cache de performance, pas un remplacement de la source de vérité.

***

## 🚀 Démarrage Rapide

### Prérequis

- Rust (stable, édition 2024)
- PostgreSQL 18.4+
- Node.js 22+ (SvelteKit 2.61+)
- Docker & Docker Compose

### Installation

```bash
# Clone
git clone https://github.com/your-org/dynamic-form-engine.git
cd dynamic-form-engine

# Backend
cd backend
cargo build --release
cp .env.example .env  # configurer DATABASE_URL, JWT_SECRET, HMAC_SECRET
cargo sqlx migrate run

# Frontend
cd ../frontend
npm install
npm run dev

# Stack complète (Docker)
docker compose up -d
```

### Variables d'environnement critiques

```env
DATABASE_URL=postgres://user:pass@localhost:5432/form_engine
JWT_SECRET=<secret_256_bits_minimum>
HMAC_SECRET=<secret_256_bits_pour_signature_ast>
AES_KEY_STORE_URL=<url_du_service_de_gestion_des_cles>
SNAPSHOT_THRESHOLD=50
```

***

## 📁 Structure du Projet

```
dynamic-form-engine/
├── backend/                    # Rust / Axum
│   ├── src/
│   │   ├── main.rs
│   │   ├── api/
│   │   │   ├── commands/       # POST — mutations (append events)
│   │   │   └── queries/        # GET  — lectures (projections)
│   │   ├── domain/
│   │   │   ├── aggregate.rs    # FormAggregate + apply(event)
│   │   │   ├── events.rs       # enum FormEvent + schema_version
│   │   │   └── snapshot.rs     # load/save snapshots
│   │   ├── ast/
│   │   │   ├── types.rs        # enum NodeType (exhaustif, typé)
│   │   │   ├── evaluator.rs    # interpréteur AST (whitelist, depth limit)
│   │   │   ├── signer.rs       # HMAC-SHA256 sign/verify
│   │   │   └── projector.rs    # filtrage AST par rôle JWT
│   │   ├── crypto/
│   │   │   ├── pii.rs          # AES-256-GCM encrypt/decrypt
│   │   │   └── key_store.rs    # rotation de clés, crypto-shredding
│   │   └── infra/
│   │       ├── db.rs           # pool SQLx
│   │       └── projections.rs  # refresh vues matérialisées
│   ├── migrations/             # SQLx migrations
│   └── Cargo.toml
│
├── frontend/                   # Svelte 5 / SvelteKit 2.61
│   ├── src/
│   │   ├── lib/
│   │   │   ├── ast/
│   │   │   │   ├── interpreter.ts  # évaluation AST côté client (UX only)
│   │   │   │   └── types.ts        # types TS mirroring enum Rust
│   │   │   ├── forms/
│   │   │   │   ├── FieldRenderer.svelte
│   │   │   │   └── FormEngine.svelte
│   │   │   └── stores/             # $state globaux typés
│   │   └── routes/
│   │       ├── forms/[id]/
│   │       │   ├── +page.svelte    # rendu formulaire
│   │       │   └── +page.server.ts # load AST depuis Rust
│   │       └── admin/
│   ├── tests/                  # Vitest + Playwright
│   └── package.json
│
├── infra/
│   ├── docker-compose.yml
│   └── nginx/                  # CSP headers, rate limiting
│
└── docs/
    ├── adr/                    # Architecture Decision Records
    └── dpia.md                 # Data Protection Impact Assessment (RGPD)
```

***

## ⚠️ Décisions Architecturales Clés (ADR)

| # | Décision | Justification |
|---|----------|---------------|
| ADR-001 | PostgreSQL 18.4 (pas 17) | Version courante stable, EOL nov. 2030 |
| ADR-002 | SQLx 0.8.6 (pas 0.9.x) | 0.9.0-alpha — instable en production |
| ADR-003 | AST signé HMAC côté Rust | Prévient le tamper client sans overhead réseau |
| ADR-004 | Snapshot threshold = 50 | Équilibre perf/stockage — ajustable |
| ADR-005 | `schema_version` sur events ET snapshots | Prépare les migrations sans casser l'historique |
| ADR-006 | Projections idempotentes + Dead Letter Queue | Resilience sans perte de données métier |
| ADR-007 | Métadonnées events exclues de la DPIA initiale | Risque résiduel identifié — à traiter avant prod |

***

## 📋 Checklist avant Production

- [ ] DPIA complète (Art. 35 RGPD) — inclure les métadonnées d'events
- [ ] Pentesting de l'endpoint de soumission (AST tamper, replay attack)
- [ ] Validation du périmètre NIS2 avec le DPO / RSSI
- [ ] Politique de rotation des clés AES documentée
- [ ] Snapshot strategy validée sur données réelles de volume
- [ ] Dead Letter Queue opérationnelle et monitorée
- [ ] CSP headers testés (rapport de violation activé)
- [ ] Rate limiting calibré selon la charge attendue
