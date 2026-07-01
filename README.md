<a id="readme-top"></a>

[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![project_license][license-shield]][license-url]

<br />
<div align="center">
<h3 align="center">SecureForm</h3>

  <p align="center">
    Moteur de formulaires métier piloté par AST (Rust/Axum + Svelte 5), conçu pour BTP/Audit avec chiffrement PII et audit trail par design.
    <br />
    <a href="docs/adr"><strong>Voir les ADR »</strong></a>
    <br />
    <br />
    <a href="https://github.com/QuentinGouttaya/SecureForm/issues/new?labels=bug">Report Bug</a>
    &middot;
    <a href="https://github.com/QuentinGouttaya/SecureForm/issues/new?labels=enhancement">Request Feature</a>
  </p>
</div>

<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#built-with">Built With</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
      </ul>
    </li>
    <li><a href="#usage">Usage</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>

## About The Project

MVP en architecture N-tier (pas de CQRS/Event Sourcing — écarté pour éviter la sur-ingénierie tant que le besoin de replay n'est pas prouvé).

```
[Svelte 5 / SvelteKit]  → présentation, fetch REST
        ↓
[Axum handlers]         → 1 handler = valide + écrit + retourne l'état
        ↓
[Domain / services]     → validation métier, évaluation AST
        ↓
[PostgreSQL]             → tables CRUD + trigger d'audit append-only
```

Principes structurels (pas des couches ajoutées après coup) :
- **SSOT strict** — PostgreSQL source de vérité, Rust valide, Svelte rend.
- **Conformité by design** — chiffrement PII (AES-256-GCM, crypto-shredding) et audit trail dès le schéma.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

### Built With

* [![Rust][Rust.dev]][Rust-url]
* [![Axum][Axum.dev]][Axum-url]
* [![Svelte][Svelte.dev]][Svelte-url]
* [![PostgreSQL][Postgres.dev]][Postgres-url]
* [![SQLx][SQLx.dev]][SQLx-url]
* [![TypeScript][TS.dev]][TS-url]
* [![TailwindCSS][Tailwind.dev]][Tailwind-url]

Versions ciblées (vérifiées juin 2026) : PostgreSQL 18.4, Axum 0.8.9, SQLx 0.8.6 (éviter 0.9-alpha), Svelte/SvelteKit 5 / 2.61+, TypeScript 6.0.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Getting Started

### Prerequisites

* Rust (stable, édition 2024)
* Node.js 22+
* PostgreSQL 18.4+
* Docker & Docker Compose

### Installation

1. Clone le repo
   ```sh
   git clone https://github.com/QuentinGouttaya/SecureForm.git
   cd dynamic-form-engine
   ```
2. Backend
   ```sh
   cd backend
   cargo build --release
   cp .env.example .env
   cargo sqlx migrate run
   ```
3. Frontend
   ```sh
   cd ../frontend
   npm install
   npm run dev
   ```
4. Renseigne les secrets dans `.env`
   ```env
   DATABASE_URL=postgres://user:pass@localhost:5432/form_engine
   JWT_SECRET=<256 bits min>
   HMAC_SECRET=<256 bits pour signature AST>
   AES_KEY_STORE_URL=<url du service de gestion des clés>
   ```
5. Stack complète
   ```sh
   docker compose up -d
   ```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Usage

### Modèle de données

```sql
CREATE TABLE form_definitions (
    id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ast     JSONB NOT NULL,       -- AST complet, Rust only
    version INT   NOT NULL DEFAULT 1
);

CREATE TABLE form_submissions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id     UUID NOT NULL REFERENCES form_definitions(id),
    payload     JSONB NOT NULL,    -- PII chiffrées AES-256-GCM
    created_at  TIMESTAMPTZ DEFAULT now()
);

-- Append-only, alimenté par trigger — pas d'event store
CREATE TABLE audit_log (
    id     BIGSERIAL PRIMARY KEY,
    row_id UUID NOT NULL,
    action TEXT NOT NULL,
    diff   JSONB NOT NULL,
    at     TIMESTAMPTZ DEFAULT now()
);
```

### Flux AST & Zero-Trust

```
GET  /forms/:id    → Axum charge l'AST depuis PG, filtre par rôle JWT, signe HMAC-SHA256, envoie {ast, sig}
                    → Svelte interprète pour l'UX only (feedback de surface)
POST /forms/:id/submit → vérifie sig (rejet si tampered) → recharge l'AST depuis PG (ignore celui du client)
                        → revalide, vérifie ACL par champ → INSERT (trigger → audit_log)
```

Protections : whitelist des field IDs, depth/complexity limits (10 niveaux, 100 nœuds), enum Rust exhaustif pour les types de nœuds, nonce anti-replay.

### Handler type

```rust
async fn submit_form(
    State(pool): State<PgPool>,
    Extension(claims): Extension<Claims>,
    Json(payload): Json<FormSubmission>,
) -> Result<Json<FormResponse>, AppError> {
    let ast = load_ast(&pool, payload.form_id).await?;
    evaluate_ast(&ast, &payload)?;
    let row = sqlx::query_as!(FormSubmission, /* INSERT ... RETURNING * */)
        .fetch_one(&pool).await?;
    Ok(Json(row.into()))
}
```

_Pour le détail des choix retirés (CQRS/ES, snapshots, vues matérialisées) et leur justification, voir [docs/adr](docs/adr)._

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Roadmap

- [ ] DPIA complète (Art. 35 RGPD), inclure métadonnées `audit_log`
- [ ] Pentest endpoint de soumission (AST tamper, replay)
- [ ] Validation périmètre NIS2 avec DPO/RSSI
- [ ] Politique de rotation des clés AES documentée
- [ ] CSP testé (rapport de violation activé)
- [ ] Rate limiting calibré
- [ ] Migration `audit_log` → event store si besoin de replay prouvé

See the [open issues](https://github.com/QuentinGouttaya/SecureForm/issues) for the full list.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Contributing

1. Fork le projet
2. Crée ta branche (`git checkout -b feature/AmazingFeature`)
3. Commit (`git commit -m 'Add some AmazingFeature'`)
4. Push (`git push origin feature/AmazingFeature`)
5. Ouvre une Pull Request

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## License

Distributed under the project_license. See `LICENSE.txt` for more information.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Contact

Quentin
Project Link: [https://github.com/QuentinGouttaya/SecureForm](https://github.com/QuentinGouttaya/SecureForm)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Acknowledgments

* [PostgreSQL 18 release notes](https://www.postgresql.org/docs/release/18.4/)
* [Axum releases](https://github.com/tokio-rs/axum/releases)
* [Svelte 5 docs](https://svelte.dev/)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- MARKDOWN LINKS & IMAGES -->
[contributors-shield]: https://img.shields.io/github/contributors/QuentinGouttaya/SecureForm.svg?style=for-the-badge
[contributors-url]: https://github.com/QuentinGouttaya/SecureForm/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/QuentinGouttaya/SecureForm.svg?style=for-the-badge
[forks-url]: https://github.com/QuentinGouttaya/SecureForm/network/members
[stars-shield]: https://img.shields.io/github/stars/QuentinGouttaya/SecureForm.svg?style=for-the-badge
[stars-url]: https://github.com/QuentinGouttaya/SecureForm/stargazers
[issues-shield]: https://img.shields.io/github/issues/QuentinGouttaya/SecureForm.svg?style=for-the-badge
[issues-url]: https://github.com/QuentinGouttaya/SecureForm/issues
[license-shield]: https://img.shields.io/github/license/QuentinGouttaya/SecureForm.svg?style=for-the-badge
[license-url]: https://github.com/QuentinGouttaya/SecureForm/LICENSE
[Rust.dev]: https://img.shields.io/badge/Rust-000000?style=for-the-badge&logo=rust&logoColor=white
[Rust-url]: https://www.rust-lang.org/
[Axum.dev]: https://img.shields.io/badge/Axum-000000?style=for-the-badge&logo=rust&logoColor=white
[Axum-url]: https://github.com/tokio-rs/axum
[Svelte.dev]: https://img.shields.io/badge/Svelte-4A4A55?style=for-the-badge&logo=svelte&logoColor=FF3E00
[Svelte-url]: https://svelte.dev/
[Postgres.dev]: https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white
[Postgres-url]: https://www.postgresql.org/
[SQLx.dev]: https://img.shields.io/badge/SQLx-2F3241?style=for-the-badge&logo=rust&logoColor=white
[SQLx-url]: https://github.com/launchbadge/sqlx
[TS.dev]: https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white
[TS-url]: https://www.typescriptlang.org/
[Tailwind.dev]: https://img.shields.io/badge/TailwindCSS-38B2AC?style=for-the-badge&logo=tailwind-css&logoColor=white
[Tailwind-url]: https://tailwindcss.com/

