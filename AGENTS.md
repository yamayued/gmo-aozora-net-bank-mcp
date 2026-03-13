# AGENTS.md

This repository is public.

## Purpose

Build and maintain a public MCP server project for the GMO Aozora Net Bank API without exposing real-world sensitive information.

## Non-Leak Rules

- Never commit or paste real API keys, client secrets, access tokens, refresh tokens, certificates, private keys, passwords, or signed request material.
- Never commit or paste real bank account numbers, branch numbers, customer IDs, corporate IDs, transaction histories, balances, beneficiary details, or personally identifiable information.
- Never include internal-only URLs, webhook endpoints, IP addresses, ticket links, Slack messages, email threads, screenshots, exports, or vendor portal data from a real environment.
- Never store secrets in source files, shell history snippets, examples, tests, fixtures, logs, screenshots, notebooks, commit messages, or pull request text.
- Never use production credentials or production data during local development, tests, demos, or CI.

## Data Handling

- Use sanitized mock data only.
- Replace all identifiers with obvious placeholders such as `ACCOUNT_NUMBER_EXAMPLE`, `CLIENT_ID_EXAMPLE`, and `TRANSACTION_ID_SAMPLE`.
- If an example needs realistic structure, preserve format only and change every value.
- If there is any doubt whether data is real, treat it as sensitive and do not commit it.

## Files And Config

- Keep secrets in untracked local files such as `.env.local` only.
- Commit only template files such as `.env.example` with dummy values.
- Add ignore rules before creating local credentials, caches, logs, exports, or test artifacts.
- Review diffs before every commit and remove anything that could identify a real user, company, account, or environment.

## Communication Rules

- When generating documentation, issues, examples, or sample prompts, describe workflows using fictional organizations and dummy account data.
- Summaries for public artifacts must stay at the architecture and sample-data level.
- Do not mention whether the author has access to any real environment unless the user explicitly asks for that clarification.

## Shell And Encoding

- Do not embed Japanese prose directly in inline shell commands.
- Read Japanese messages from UTF-8 `.txt` or `.json` files when shell execution needs them.

## If A Leak Is Suspected

- Stop work that would publish the data.
- Remove the sensitive content from tracked files.
- Rotate or revoke any exposed credential if applicable.
- Tell the user what was found and what still needs cleanup.
