# Deploying `susu-web` to Vercel

`susu-web` is a **static single-page application**. `pnpm build` produces `dist/`, Vercel serves it,
and the browser does the rest — including all the routing. That last part is the whole subject of
this page, because it is the one thing a host cannot infer and the one thing that breaks in
production but never locally.

`vercel.json` in the repository root is the configuration. It is small, and every line of it is
load-bearing.

---

## The 404 on refresh, and why it happens

If you deploy this app to Vercel with no `vercel.json`, this happens:

- The home page loads.
- Clicking around works.
- Then you refresh, open a deep link, return from the Supabase authentication email, or press back
  into a route — and you get a page saying **"This page doesn't exist"** with a request ID like
  `cpt1::vhrzs-1789511907193-5a353e9e864c`.

**That page is Vercel's, not this app's.** The distinction is worth internalising, because it tells
you immediately which layer failed:

| | |
| --- | --- |
| This app's 404 | `src/pages/NotFound.tsx`. Reads **"Page not found"** / "There is nothing at this address." Rendered in the app's own styling, with a link back to Susu Protocol. |
| Vercel's 404 | **"This page doesn't exist"**, `404 NOT_FOUND`, and a region-prefixed request ID (`cpt1::…`). Unstyled. |

Seeing the second one means **the request never reached the application**. React Router never ran,
because `index.html` was never served.

### Why it is only *some* navigations

Vercel resolves every URL as a **filesystem path** by default, which is correct for frameworks built
on file-based routing (Next.js, SvelteKit) and wrong for a single-page app. This app has exactly one
HTML entry document, so in a build with no rewrite rule:

- `/` → a real file exists → works.
- `/assets/index-a1b2c3.js` → a real file exists → works.
- `/app/groups/123` → **no such file** → Vercel's 404.

Client-side navigation never asks the server for anything, so it is unaffected. But the browser does
ask the server whenever it needs a **document**, and that happens on more occasions than "typing a
URL":

- refreshing on a route
- opening a deep link, or a bookmark
- the redirect from a Supabase confirmation or recovery email
- pressing back into a history entry that was originally loaded as a document
- opening the app in a new tab from a link

So "it only breaks when I go back" is really "it breaks whenever the browser asks the server for a
route instead of asking the router". The fix is to make the server answer *every* path with
`index.html` and let the router decide what it means.

---

## The fix

`vercel.json`:

```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

That is the entire fix for the 404, and it is the same guarantee the other two deployment paths
already have: nginx's `try_files $uri /index.html`, and Render's `routes:` block in `render.yaml`.
All three say *serve the real file if one exists, otherwise serve the entry document.*

**It cannot break the hashed assets.** Vercel serves a file that exists at the requested path before
considering a rewrite, so `/assets/index-a1b2c3.js` is still served as itself. This matters more than
it looks: if a missing asset were answered with `index.html`, the browser would report
*"expected a JavaScript module, got text/html"* — a message that points at your code rather than at
your routing.

### The file must be at the repository root and committed

Vercel reads `vercel.json` from the **repository root** and nowhere else, and it must be tracked by
git. An uncommitted or ignored `vercel.json` produces no error at all — the rewrites simply never
apply, and the 404 looks exactly the same as if the file did not exist. If this deployment is still
404ing after adding the file, check it is actually in the commit:

```bash
git log --oneline -- vercel.json
```

---

## One setting in the dashboard, and it is not obvious

**Set Framework Preset to `Vite`.** In Vercel: Project → Settings → Build & Development Settings.

Vercel's own troubleshooting documentation is explicit that a preset of **Other** can stop even a
correctly written `vercel.json` rewrite from taking effect. The file looks right, the deploy
succeeds, and the 404 remains — which is a genuinely difficult thing to debug, because the evidence
points at the rewrite rather than at the setting that disables it.

`vercel.json` also states `buildCommand` and `outputDirectory` explicitly. With the Vite preset those
are redundant, and they are what makes the deployment correct if the preset is ever wrong.

---

## Environment variables

The same values as [the Render deployment](DEPLOY_RENDER.md#the-values-render-prompts-for), set in
Vercel under Project → Settings → Environment Variables:

| Variable | Value |
| --- | --- |
| `VITE_APP_URL` | This site's URL, e.g. `https://susu-web.vercel.app` |
| `VITE_SUPABASE_URL` | Supabase project URL |
| `VITE_SUPABASE_ANON_KEY` | Supabase **anon/publishable** key — never the service-role key |
| `VITE_API_BASE_URL` | `https://<api>.onrender.com/api/v1` — note the `/api/v1` prefix |
| `VITE_STELLAR_NETWORK` | `testnet` |
| `VITE_STELLAR_RPC_URL` | `https://soroban-testnet.stellar.org` |
| `VITE_EXPLORER_BASE_URL` | `https://stellar.expert/explorer/testnet` |
| `VITE_FACTORY_CONTRACT_ID` | `susu-contracts/docs/TESTNET.md` |
| `VITE_USDC_CONTRACT_ID` | `susu-contracts/docs/TESTNET.md` |

Two things carry over from the Render notes and are worth repeating because Vercel makes them worse:

- **`VITE_` variables are compiled in at build time.** Vercel does not redeploy when you change an
  environment variable, so a change needs a manual redeploy before it has any effect.
- **Only browser-safe values may go here.** They are published to every visitor. The service-role key
  belongs to `susu-api` alone.

---

## Verify

```bash
# The site is up.
curl -sS -o /dev/null -w '%{http_code}\n' https://<site>.vercel.app

# The fix: a deep link must be 200 and must return the entry document, not
# Vercel's 404. This is the check that reproduces the bug before the fix.
curl -sS -o /dev/null -w '%{http_code}\n' https://<site>.vercel.app/app/groups/123

# Still the entry document, and not a MIME error.
curl -sS -o /dev/null -w '%{http_code}\n' https://<site>.vercel.app/login

# A missing asset still 404s honestly rather than being answered with index.html.
curl -sS -o /dev/null -w '%{http_code}\n' https://<site>.vercel.app/assets/does-not-exist.js

# Security headers.
curl -sSI https://<site>.vercel.app | grep -iE 'x-frame-options|referrer-policy|x-content-type-options'
```

The third check is the one that matters most, and it is easy to forget to run. Deep links passing is
only half the fix; they have to pass *without* the rewrite swallowing real files.

Then open the site, connect Freighter on **Testnet**, and confirm it can read a group. Finally, add
the site's origin to Supabase under **Authentication → URL Configuration → Redirect URLs**, as
`{your URL}/app` and `{your URL}/reset-password` — a Supabase email redirect is a document request,
so it lands on the 404 too if the rewrite is missing.

---

## What `vercel.json` does not contain

**No `Content-Security-Policy`, and no `Cross-Origin-Opener-Policy`.** The same two headers are
deliberately absent from `docker/security-headers.conf` and `render.yaml`, for the same reasons.

A useful CSP here has to name the Supabase origin, the Soroban RPC endpoint, the block explorer, and
allow the Freighter extension's injected script. A policy written without testing those against a
live deployment breaks either the wallet flow or the transaction pipeline, and both failures look
like bugs in this app rather than in its headers.

`Cross-Origin-Opener-Policy` is actively harmful here: the Freighter adapter talks to the browser
extension through a window handle, and that header severs it.

**No `cleanUrls`.** Vercel's documentation notes that `cleanUrls: true` strips `.html` from paths, so
a rewrite whose destination is `/index.html` is silently ignored and must be written as `/index`
instead. There is nothing to gain from it here, and a rewrite that quietly stops working is exactly
the failure this page exists to prevent.

**No `Cache-Control` for HTML.** Vercel's defaults already revalidate the entry document, and a rule
matching `/(.*)` would apply the same policy to the hashed assets and the entry document alike. The
one cache rule present targets `/assets/(.*)`, whose filenames contain a content hash and are
therefore genuinely immutable.
