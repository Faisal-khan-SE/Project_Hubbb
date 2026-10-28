# Contributing

## Workflow
1. **Pick the task** from `docs/PROMPT_PACK.md` (check `docs/PROGRESS.md` for what is next and the dependency table).
2. **Branch** from up-to-date `main` with the exact branch name from the task: `git checkout main && git pull && git checkout -b feat/A1-foundation`.
3. **Implement** only that task, inside its permitted scope.
4. **Test**: write tests with the code and run them. Never claim a test passed unless it ran and passed.
5. **PR** to `main` using the template. Update `docs/PROGRESS.md` in the same PR.
6. **Review** by the other founder. Changes to `contracts/` or `docs/decisions.md` need both founders.
7. **Merge** only when CI is green. Then start the next task from fresh `main`.

## Commit messages
Conventional style, small and meaningful:
```
feat(auth): add email OTP verification
fix(payments): make gateway callback idempotent
test(rules): add penalty boundary cases
docs(adr): add ADR 0003 estimator v1
chore(ci): add ai-service job
```
Types: `feat`, `fix`, `test`, `docs`, `refactor`, `chore`. Scope = module or folder.
No fake or backdated commits; each founder commits only their own work.

## Never commit
- Secrets of any kind (passwords, API keys, Gmail App Password, tokens, gateway salts). Use `.env` (git-ignored) and keep `.env.example` updated with placeholders.
- `.env`, build output (`target/`, `.next/`, `dist/`), `node_modules/`, virtualenvs, zip files.

## Working without the GitHub connector (zip flow)
1. `git checkout main && git pull && git archive -o marketplace.zip HEAD`
2. Upload the zip to a new AI chat with the task prompt.
3. Create the task branch, extract the returned zip over the repo folder, delete any files listed as removed.
4. `git status`, then `git add . && git commit -m "<message>" && git push -u origin <branch>` and open a PR.
