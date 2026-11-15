# login.flisk.si

One sign-in for every Flisk site. A visitor signs in with GitHub once, here. This Worker hands the site a short signed token. The site checks it with a public key. No site needs its own GitHub app or secret.

```
site /auth/start ──► login.flisk.si/start ──► GitHub ──► login.flisk.si/callback
                                                               │ signed token (60 s, for that site only)
site /auth/done?s=<state>&flisk_token=<token> ◄────────────────┘
```

The GitHub token is thrown away after we read the user's name. Sites that need GitHub repo access (chat does, for push and PRs) ask for it themselves, separately.

## Add a site in 3 steps

1. **Public key.** Put `LOGIN_PUBLIC_KEY = "g_3cGlqaMDf4mM4Bgfv8EO6js7yW_bcSW7cOUpSwLc0"` in your Worker's `[vars]`. It is public, not a secret.
2. **Start.** In `/auth/start`, make a random `state`, set it in an HttpOnly cookie, and redirect to `https://login.flisk.si/start?return=<your origin>/auth/done?s=<state>` (URL-encode the `return`).
3. **Done.** In `/auth/done`, require `s` to equal the state cookie, then verify `flisk_token` against the public key and your own origin as audience. Start your own session from `{login, id}`.

Copy `verifyLoginToken` from [grow's `src/login-token.js`](https://github.com/flisk-sh/grow/blob/main/src/login-token.js). Working examples: grow, learn, chat.

## Rules

- Return URLs must be `https://*.flisk.si`, never `pages.flisk.si` (user pages live there).
- A token is valid for 60 seconds and only for the `aud` it was issued for.
- Always check the state cookie, or anyone can sign a visitor in as someone else.

## Run and deploy

```
bun test
bun scripts/make-keys.mjs      # new key pair; put the public half in wrangler.toml
bunx wrangler@4 deploy
```

Secrets (set with `wrangler secret put`): `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `SIGNING_KEY`, `STATE_SECRET`.
