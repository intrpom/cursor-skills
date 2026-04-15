# Rozložení Cursor skills (sjednoceno)

`~/.cursor/skills` ukazuje sem (symlink na tento repozitář). Agent má jeden jasný „domov“ pro vlastní skills; built-in zůstávají u Cursoru.

## Kde co žije

| Místo | Účel |
|--------|------|
| **Tento repozitář** (`~/.cursor/skills`) | Vlastní skills: integrace, skripty, opakovatelné postupy. Sem patří nové skills. |
| **`~/.cursor/skills-cursor/`** | Built-in od Cursoru + spravované skills. Obsah nesmažuj ani naduplikuj do tohoto repa — vznikly by dva záznamy se stejným účelem. |

Manifest: `~/.cursor/skills-cursor/.cursor-managed-skills-manifest.json` (builtin + managed ID).

## Skills v tomto repozitáři

| Složka | K čemu |
|--------|--------|
| `db-backup-postgres-sqlite` | Zálohy Postgres → JSON, restore do SQLite |
| `email-validation-nextjs` | Vrstvená validace e-mailu (Next.js) |
| `mailgun-nextjs-integration` | Mailgun + App Router |
| `stripe-nextjs-integration` | Stripe checkout / webhooks + Next.js |
| `create-hook` | Cursor hooks, `hooks.json`, skripty |
| `statusline` | Vlastní status line v CLI |
| `update-cli-config` | `~/.cursor/cli-config.json`, CLI preference |

## Built-in u Cursoru (`skills-cursor`, nedržet kopii zde)

- `create-rule`, `create-skill`, `create-subagent`, `migrate-to-skills`, `update-cursor-settings`, `shell`
- Spravované: `babysit`

## Pravidlo při přidání skill

Nová skill = nová složka `nazev-skillu/SKILL.md` **jen v tomto repu**, frontmatter s `name` a `description`.
