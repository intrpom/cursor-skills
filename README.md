# Cursor Skills

Osobní knihovna Agent Skills pro Cursor IDE.

## Struktura

Každá skill je složka s jedním souborem `SKILL.md`.

### Projektové skills (`~/.cursor/skills/`)
- **stripe-nextjs-integration** — Stripe platby v Next.js App Router
- **mailgun-nextjs-integration** — Mailgun emaily v Next.js
- **email-validation-nextjs** — Multi-layer validace emailů (typo detekce, disposable blokování)
- **db-backup-postgres-sqlite** — Záloha databáze PostgreSQL / SQLite + Dropbox upload

### Cursor systémové skills (`~/.cursor/skills-cursor/`)
- **create-skill** — Vytvoření nové skill
- **create-rule** — Vytvoření Cursor rule
- **create-subagent** — Vytvoření subagenta
- **update-cursor-settings** — Úprava Cursor nastavení
- **migrate-to-skills** — Migrace do skills systému

## Jak obnovit po přeinstalaci

```bash
# Zkopírovat skills do Cursor
cp -r ./stripe-nextjs-integration ./mailgun-nextjs-integration ./email-validation-nextjs ./db-backup-postgres-sqlite ~/.cursor/skills/
cp -r ./create-skill ./create-rule ./create-subagent ./update-cursor-settings ./migrate-to-skills ~/.cursor/skills-cursor/
```

## Jak zálohovat po přidání nové skill

```bash
cd ~/CascadeProjects/cursor-skills
cp -r ~/.cursor/skills/* .
cp -r ~/.cursor/skills-cursor/* .
git add -A && git commit -m "update skills" && git push
```
