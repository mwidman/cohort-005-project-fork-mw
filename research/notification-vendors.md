# Notification vendors for a Cadence in-app feed

_Researched 2026-11-11. Primary sources only (vendor docs, vendor GitHub, npm registry). npm data came from `npm view <pkg> version peerDependencies time.modified` and `npm pack` on 2026-11-11. "npm modified" is the registry's `time.modified`, which is a close proxy for the latest publish date._

## Context

**Question:** Which hosted notification service should Cadence use to add an in-app notification feed?

**Cadence stack:** React Router 8.3 framework mode with SSR (`@react-router/node`, `@react-router/serve`), React 19.2, a Node 22 server, SQLite via better-sqlite3 and drizzle-orm, Vite 7, and Tailwind 4 with shadcn/radix. Auth uses cookie sessions and a dev user switcher. Users have integer IDs. There is no external auth provider.

**Criteria**
- **Must have:** a hosted in-app feed with a React component or headless hooks, a Node server SDK for triggering, and a free tier usable for development.
- **Nice to have:** open source or self-hostable, real-time (websocket) updates, per-recipient preferences, and an email channel that can be added later.
- **Out of scope:** pricing at scale. Each section still says whether a free dev tier exists and gives its rough limits.

## Comparison

| Vendor | Must-haves (feed UI / Node SDK / free tier) | Nice-to-haves (OSS / real-time / prefs / email) | Browser auth model | Recipient identification | React 19 / SSR |
|---|---|---|---|---|---|
| **Knock** | Yes: `@knocklabs/react` + `@knocklabs/react-core` hooks / `@knocklabs/node` / Free, 10k messages per month | Not OSS or self-hostable (no option found) / Yes, socket / Yes, on Free / Yes | Public key plus an RS256 JWT (`sub` = user ID) signed locally with `signUserToken`; "enhanced security mode" enforces it | Inline identify on trigger or upfront `PUT /v1/users/{id}`; your internal ID recommended | Peer `react ^17 \|\| ^18 \|\| ^19`; ships `"use client"`; `next` and `@tanstack/react-router` peers are optional |
| **Novu** | Yes: `@novu/react` `<Inbox/>` + hooks / `@novu/api` / Free, 10k workflow runs per month | **Yes, MIT core, self-hostable** / Yes, WebSocket / Yes / Yes | `subscriberId` plus `subscriberHash` = HMAC-SHA256(secretKey, subscriberId), computed locally | Inline subscriber on trigger (`to`) or upfront create; app user ID recommended | Peer `react ^18 \|\| ^19`; SSR not documented |
| **Courier** | Yes: `@trycourier/courier-react` + `useCourier` / `@trycourier/courier` / Free, 10k sends per month (Inbox on Free not stated) | Not OSS / Yes, `listenForUpdates()` / Yes / Yes | JWT minted by a **network call** to `POST /auth/issue-token` with scopes; no SDK auto-refresh | Send to `user_id`; docs say create the profile first | Peer `react >=18`; docs say **client-side only** |
| **MagicBell** | Yes: `@magicbell/react` (new) or `@magicbell/magicbell-react` (older) / `magicbell-js` / Free "Builder", 1,000 deliveries per month | SDKs open source, platform not self-hostable (no option found) / Likely (depends on `reconnecting-websocket`) / Yes / Yes | HS256 User JWT (`user_external_id` or `user_email`, `api_key`) signed with the project secret | `external_id` on broadcast recipients; `saveUser` upsert; auto-create on broadcast not documented | `@magicbell/react` declares **no** react peer; older package needs `react >=18.3.1`; SSR not documented |
| **SuprSend** | Yes: `@suprsend/react` + `@suprsend/react-core` hooks / `@suprsend/node-sdk` / Free, 10k notifications per month (inbox carries SuprSend branding) | Not OSS (no option found) / Feed socket exists, push behavior not documented / Preferences listed only under Business on the pricing page / Yes | Public key plus an ES256 JWT (`entity_id`, `scope.tenant_id`) signed locally; "enhanced security mode" | `distinct_id` on pre-synced users or an inline recipient object on trigger | Peer `react >=16.8.0`; SSR not documented |

## Vendor details

### Knock

- **Feed UI.** `@knocklabs/react` provides the drop-in feed and is the successor to `@knocklabs/react-notification-feed` ([docs](https://docs.knock.app/in-app-ui/react/overview)). The page says the SDK "includes a complete set of React hooks that you can use to build your own UI on top of" ([docs](https://docs.knock.app/in-app-ui/react/overview)). The hooks live in `@knocklabs/react-core`, which `@knocklabs/react` depends on (npm registry). Setup is `KnockProvider` (`apiKey`, `user`) wrapping `KnockFeedProvider` (`feedId`) ([docs](https://docs.knock.app/in-app-ui/react/feed)).
- **Node SDK.** `@knocklabs/node` 1.41.1, npm modified 2026-11-10 ([npm](https://www.npmjs.com/package/@knocklabs/node)).
- **Free tier.** $0 per month, up to 10,000 messages per month, unlimited channels, in-app feeds with unlimited retention, and preferences included. "We don't hard cap your messages." ([pricing](https://knock.app/pricing))
- **Real-time.** The feed holds a socket. With auto-disconnect on, an inactive tab drops it and reconnects "to continue receiving real-time updates" ([docs](https://docs.knock.app/in-app-ui/react/feed)).
- **Preferences and email.** Preferences are on the Free plan, and delivery channels are unlimited ([pricing](https://knock.app/pricing)).
- **OSS or self-host.** The docs and pricing pages I read offer no self-hosting option. I did not verify this further.
- **Browser auth.** Without enhanced security mode, "a request with your public key can act as any user." With it on, Knock rejects public-key requests that lack a signed user token. The token is an **RS256 JWT** signed with a private signing key from the dashboard, with `sub` set to the user ID and `iat`/`exp` recommended. Signing happens locally, so it costs no network call. The helper is `signUserToken` from `@knocklabs/node/lib/tokenSigner`; it reads `KNOCK_SIGNING_KEY` and defaults to a 3600 s expiry ([docs](https://docs.knock.app/in-app-ui/security-and-authentication)).
  - **Cadence server work:** in the root (or layout) loader, call `signUserToken(String(user.id))` and return it with the user. The client passes it as `userToken` ([docs](https://docs.knock.app/in-app-ui/security-and-authentication), [docs](https://docs.knock.app/in-app-ui/react/feed)).
- **Recipients.** Knock supports inline identification inside a workflow trigger, and "Knock guarantees that recipients are identified before executing any other action." It also supports upfront `PUT /v1/users/{user_id}` and bulk upserts. Knock recommends "using your internal user identifier for the `id` value" ([docs](https://docs.knock.app/managing-recipients/identifying-recipients)). Cadence's integer IDs would be sent as strings. The docs page does not state format restrictions on IDs.
- **React 19 and SSR.** `@knocklabs/react` 0.15.1 (npm modified 2026-11-06) has peers `react`/`react-dom` `^17 || ^18 || ^19`. It also lists `next` and `@tanstack/react-router` as peers, but both are marked optional in `peerDependenciesMeta` (npm registry). The bundle's `dist/esm/index.mjs` begins with `"use client"` (inspected via `npm pack`). That directive is a no-op outside RSC. SSR behavior is not covered in the feed docs ([docs](https://docs.knock.app/in-app-ui/react/feed)), so plan to render the feed client-only.

### Novu

- **Feed UI.** `@novu/react` provides `<Inbox />` ([docs](https://docs.novu.co/platform/quickstart/react)). The headless hooks are `useNotifications`, `useCounts`, and `useNovu` ([useNotifications](https://docs.novu.co/platform/sdks/react/hooks/use-notifications), [useCounts](https://docs.novu.co/platform/sdks/react/hooks/use-counts), [useNovu](https://docs.novu.co/platform/sdks/react/hooks/use-novu)). Current version is 3.19.2, npm modified 2026-09-24.
- **Node SDK.** Use `@novu/api` (3.19.1, npm modified 2026-09-02) and call `novu.trigger({ to: { subscriberId, ... }, workflowId, payload })` ([docs](https://docs.novu.co/platform/sdks/server/typescript)). **`@novu/node` is deprecated.** Its npm deprecation notice reads "March 20, 2025 marks the end of support for @novu/node … migrate to the new @novu/api package" (npm registry).
- **Free tier.** $0 with no credit card required. It includes 10,000 workflow runs per month, up to 5 workflows, 2 environments, 3 team members, 24-hour activity feed retention, and all channels including email and in-app ([pricing](https://novu.co/pricing)).
- **OSS and self-host.** The core is "fully open source, licensed under MIT license." The `enterprise` folder is under a commercial license ([GitHub](https://github.com/novuhq/novu)). The community self-hosted edition runs the API, Worker, WS, and Dashboard services. "Some features exclusive to Novu's cloud-managed solution will not be available" when self-hosted, and that page does not list which ones ([docs](https://docs.novu.co/community/self-hosting-novu/overview)). Self-hosted on the Free plan is not offered; the pricing page lists self-hosted deployment under Enterprise ([pricing](https://novu.co/pricing)). That refers to the commercial offering, not the MIT code.
- **Real-time.** `useNotifications` has a `realtime` option that, "when false, disables the WebSocket subscription." `useCounts` updates in real time ([useNotifications](https://docs.novu.co/platform/sdks/react/hooks/use-notifications), [useCounts](https://docs.novu.co/platform/sdks/react/hooks/use-counts)). `<Inbox>` accepts `socketUrl` and `socketOptions` ([docs](https://docs.novu.co/platform/sdks/react)).
- **Preferences and email.** `Inbox` takes preference props, and a `Subscription` component exists ([docs](https://docs.novu.co/platform/sdks/react)). Email is on the Free plan ([pricing](https://novu.co/pricing)).
- **Browser auth.** Novu uses HMAC: "The hash is computed as `HMAC-SHA256(secretKey, subscriberId)` and returned as a lowercase hex string." The client passes it as `subscriberHash`. With HMAC enabled in the In-App provider settings and no hash supplied, the Inbox does not load ([docs](https://docs.novu.co/platform/inbox/prepare-for-production)).
  - **Cadence server work:** in a loader, compute `crypto.createHmac('sha256', NOVU_SECRET_KEY).update(String(user.id)).digest('hex')`. This needs no SDK and no network call. Return it with the user.
  - The hash never expires. It is a static per-user value, which is weaker than a short-lived JWT (inference from the docs above).
- **Recipients.** "Use your application's internal user ID as the `subscriberId`." Subscribers can be created just-in-time by passing the subscriber object in `to` on trigger, or created upfront ([docs](https://docs.novu.co/platform/concepts/subscribers)). The Inbox also auto-creates an unknown `subscriberId` ([docs](https://docs.novu.co/platform/quickstart/react)).
- **React 19 and SSR.** Peers are `react`/`react-dom` `^18.0.0 || ^19.0.0 || ^19.0.0-0` (npm registry). The docs state "React 18.0.0 or later (`^18.0.0` or `^19.0.0`)" ([docs](https://docs.novu.co/platform/sdks/react)). The docs pages I read have no SSR or `'use client'` guidance, and I found no `"use client"` directive in the published `dist`. **Unverified;** plan to render client-only.

### Courier

- **Feed UI.** `@trycourier/courier-react` 9.3.5 (npm modified 2026-11-09) is for React 18+, and `@trycourier/courier-react-17` is for React 17 ([docs](https://www.courier.com/docs/sdk-libraries/courier-react-web)). It wraps Courier's `courier-ui-inbox`, `courier-ui-toast`, and `courier-ui-preferences` packages (npm dependencies). The `useCourier` hook supports headless UIs. **`@trycourier/react-inbox` is deprecated** ("Moved to @trycourier/courier-react", npm registry).
- **Node SDK.** `@trycourier/courier` 10.0.0, npm modified 2026-10-29, called as `client.send.message({ message: { to: { user_id }, template, data } })` ([docs](https://www.courier.com/docs/sdk-libraries/node)).
- **Free tier.** The Developer plan includes "up to 10k / mo." sends. The pricing page **does not state whether Inbox is included on Free** ([pricing](https://www.courier.com/pricing)). **Unverified.**
- **Real-time.** You must call `inbox.listenForUpdates()` after auth, which "opens the realtime connection." Without it the inbox loads once ([docs](https://www.courier.com/docs/sdk-libraries/courier-react-web)).
- **Preferences and email.** `<CourierPreferences />` is the preference center ([docs](https://www.courier.com/docs/sdk-libraries/courier-react-web)). Email is a core Courier channel.
- **OSS or self-host.** No self-hosting option found in the docs I read. Not further verified.
- **Browser auth.** The backend calls `POST https://api.courier.com/auth/issue-token` with the workspace API key, a `scope` (e.g. `user_id:<id> inbox:read:messages inbox:write:events`), and `expires_in`. The browser calls `Courier.shared.signIn({ userId, jwt })`. Omitting `expires_in` "mints a token that never expires," and "the SDKs also do not refresh tokens" ([docs](https://www.courier.com/docs/platform/inbox/authentication)).
  - **Cadence server work:** the loader makes an outbound HTTP call to Courier per token. This adds latency and an external dependency to page loads unless the token is cached in the session. Cadence also needs a refresh path.
- **Recipients.** Messages address `to: { user_id }`. The Node docs say "Create the profile before sending to a `user_id`" ([docs](https://www.courier.com/docs/sdk-libraries/node)). Whether an inline `{ user_id, email }` creates or updates a profile is **not stated** on that page.
- **React 19 and SSR.** Peers are `react`/`react-dom` `>=18.0.0` (npm registry). "Courier Inbox and Toast support Next.js but render client-side only" ([docs](https://www.courier.com/docs/sdk-libraries/courier-react-web)). With React Router SSR, render it client-only.

### MagicBell

- **Feed UI.** There are two package lines:
  - `@magicbell/react` 1.14.0 (npm modified 2026-07-29) is the one the current docs document. It provides `ContextProvider` (`token` = User JWT), `FloatingInbox`, `Inbox`, and `WebPushButton` ([docs](https://www.magicbell.com/docs/libraries/magicbell-react)).
  - The older `@magicbell/magicbell-react` 11.5.1 and `@magicbell/react-headless` 5.5.1 were last modified 2026-05-22 (npm registry). The current docs page does not mention them, and the new docs do not document headless hooks.
- **Node SDK.** `magicbell-js` 1.8.0 (npm modified 2026-07-29) uses `ProjectClient` from `magicbell-js/project-client` and supports `createBroadcast` and `saveUser` ("Creates or updates a user") ([docs](https://www.magicbell.com/docs/libraries/magicbell-js)). The older `magicbell` package (4.5.1) also exists.
- **Free tier.** The "Builder" plan is $0 per month with "1,000 Deliveries/mo.", "Unlimited Users", and 1 project. Deliveries are counted per channel ([pricing](https://www.magicbell.com/pricing)). This is the smallest free tier in the comparison.
- **Real-time.** `@magicbell/react` depends on `reconnecting-websocket` (npm registry), which strongly suggests websocket updates. The docs page does not describe them, so this is **not verified in docs**.
- **Preferences and email.** Broadcasts support `overrides.channels.in_app` and `email`. Delivery is "based on user preferences" ([docs](https://www.magicbell.com/docs/primitive/broadcast)). The packaged dist includes `user-preferences.js` (npm pack).
- **OSS or self-host.** The client SDKs live in the public `magicbell/magicbell-js` repo (npm `repository.url`). No self-hosting of the platform found. Not further verified.
- **Browser auth.** The User JWT is **HS256**, signed with the project Secret Key. Claims are `user_external_id` or `user_email`, plus `api_key`. "Never expose it in the browser." ([docs](https://www.magicbell.com/docs/api/authentication/user))
  - **Cadence server work:** sign the JWT in a loader with a generic JWT library such as `jsonwebtoken`. It needs no network call.
- **Recipients.** Broadcast recipients carry `external_id`, `email`, and other fields ([docs](https://www.magicbell.com/docs/primitive/broadcast)). The docs **do not say** whether unknown `external_id`s are auto-created. `saveUser` can upsert upfront.
- **React 19 and SSR.**
  - `@magicbell/react` declares **no `peerDependencies`** even though its dist imports `react` (npm pack). npm will not flag a React version mismatch.
  - The older `@magicbell/magicbell-react` requires `react >= 18.3.1`.
  - The new package's component files start with `"use client"` (npm pack).
  - SSR is not documented.

### SuprSend

- **Feed UI.** `@suprsend/react` 1.5.0 (npm modified 2026-10-22) provides drop-in Inbox and Preferences. `@suprsend/react-core` 2.4.0 provides providers and hooks: `SuprSendProvider`, `SuprSendFeedProvider`, `useFeedData`, `useFeedClient`, and `useFeed` ([docs](https://docs.suprsend.com/docs/react-sdk), [docs](https://docs.suprsend.com/docs/react-sdk-headless-feed)). **`@suprsend/react-inbox` has reached end of support** ([docs](https://docs.suprsend.com/docs/inbox-react)) and is deprecated on npm.
- **Node SDK.** `@suprsend/node-sdk` 1.15.1, npm modified 2026-05-19. Triggering builds a `WorkflowTriggerRequest` and calls `supr_client.workflows.trigger(...)` ([docs](https://docs.suprsend.com/docs/node-trigger-workflow-from-api)). I did not open that page directly; the pattern comes from search snippets of the docs. **Verify the constructor signature before use.**
- **Free tier.** "10k Notifications/mo" and unlimited channels. The free in-app feed carries "SuprSend branding." **User Preferences are listed only under the Business plan**, and log retention is 30 days ([pricing](https://www.suprsend.com/pricing)).
- **Real-time.** The headless docs reference "the feed socket" status (`reachability.socket.status`) and a `socketHost` option, and they say data "gets updated internally" ([docs](https://docs.suprsend.com/docs/react-sdk-headless-feed)). Push behavior is implied but not described explicitly.
- **OSS or self-host.** No self-hosting option found. Not further verified.
- **Browser auth.** Clients use a public key. In "enhanced security mode" requests also need a server-signed **ES256 JWT** with claims `entity_type: "subscriber"`, `entity_id`, `iat`, `exp`, and `scope.tenant_id`. The client calls `identify(userId, userToken)`, and there is an optional `refreshUserToken` callback ([docs](https://docs.suprsend.com/docs/client-authentication)).
  - **Cadence server work:** sign the ES256 JWT in a loader and provide a small refresh endpoint.
- **Recipients.** Recipients are `distinct_id`s of pre-synced users, or inline recipient objects with `$email`, `$channels`, and similar fields, up to 100 per trigger ([docs](https://docs.suprsend.com/docs/node-trigger-workflow-from-api)). As above, this comes from search snippets.
- **React 19 and SSR.** Peer is `react >= 16.8.0`, which is open-ended and includes 19 (npm registry). `@suprsend/react` pulls in `@emotion/react` and older `react-markdown@6` (npm registry). SSR is not documented.

### Considered and rejected

- **Engagespot.** `@engagespot/react-component` 3.0.1 was last modified 2025-05-09, and `@engagespot/node` 1.7.0 on 2024-11-06 (npm registry). Neither has had a release in more than a year, which is a weak maintenance signal compared with the five above. I did not research it further.

## Fit for Cadence

All five vendors meet the must-haves on paper. All of them accept arbitrary string user IDs, so Cadence would send `String(user.id)`. All of them require a server-side secret, which fits React Router loaders naturally. In every case the feed should render **client-only**: Courier says so explicitly, and none of the others document SSR. The dev user switcher means the token or hash must be re-derived per request from the session user, not cached across switches.

**Suggested shortlist (for the decision elsewhere):**

1. **Knock.** It has the strongest dev free tier (10k messages, preferences included), local RS256 token signing with a first-party helper, inline identify on trigger (no upfront user sync needed), explicit React 19 peer support, and very active releases (Node SDK published 2026-11-10).
2. **Novu.** It is the only option with an MIT-licensed, self-hostable core. It has headless hooks with WebSocket real-time, a trivial HMAC auth (one `crypto.createHmac` call), and inline subscriber creation. The trade-offs are 24-hour activity retention on Free and a non-expiring per-user hash.
3. **SuprSend (alternate).** It offers a comparable 10k free tier, ES256 JWT auth with a refresh callback, and inline recipients. However, preferences appear Business-only and the free inbox is branded.

**Less suitable:**
- **Courier** requires an outbound API call to mint each Inbox JWT, and Inbox on Free is unconfirmed.
- **MagicBell** has the smallest free tier (1,000 deliveries), two overlapping React package lines, and no declared React peer range.

## Open questions

- Does Courier's free Developer plan include Inbox, and does an inline `to: { user_id, email }` upsert a profile? Neither is stated on the pages read.
- Which features are missing from Novu's self-hosted community edition? Is Inbox HMAC enforced the same way when self-hosted?
- Do any of the React SDKs touch `window` or `document` at module import time? This decides whether a `ClientOnly` wrapper is enough or whether a lazy `import()` is needed under React Router SSR. Answering it requires a quick spike per shortlisted vendor.
- Does MagicBell auto-create users on broadcast to an unknown `external_id`? Which React package line is long-term?
- Are SuprSend preferences actually unavailable on Free, or only unlisted? Does the Node SDK constructor match the snippet pattern?
- Knock and Novu: what is the CSP or bundle impact of their bundled UI deps (Knock pulls `@telegraph/*` and Radix; Novu not inspected)?
- Can Knock's free plan Feed API and Novu's 24-hour activity retention coexist with Cadence's dev/test workflow (e.g. seeded notifications)?

## Sources

- Knock: https://docs.knock.app/in-app-ui/security-and-authentication · https://docs.knock.app/managing-recipients/identifying-recipients · https://docs.knock.app/in-app-ui/react/overview · https://docs.knock.app/in-app-ui/react/feed · https://knock.app/pricing · https://www.npmjs.com/package/@knocklabs/react · https://www.npmjs.com/package/@knocklabs/react-core · https://www.npmjs.com/package/@knocklabs/node
- Novu: https://docs.novu.co/platform/inbox/prepare-for-production · https://docs.novu.co/platform/quickstart/react · https://docs.novu.co/platform/sdks/react · https://docs.novu.co/platform/sdks/react/hooks/use-notifications · https://docs.novu.co/platform/sdks/react/hooks/use-counts · https://docs.novu.co/platform/sdks/react/hooks/use-novu · https://docs.novu.co/platform/sdks/server/typescript · https://docs.novu.co/platform/concepts/subscribers · https://docs.novu.co/community/self-hosting-novu/overview · https://novu.co/pricing · https://github.com/novuhq/novu · https://www.npmjs.com/package/@novu/react · https://www.npmjs.com/package/@novu/api · https://www.npmjs.com/package/@novu/node
- Courier: https://www.courier.com/docs/platform/inbox/authentication · https://www.courier.com/docs/sdk-libraries/courier-react-web · https://www.courier.com/docs/sdk-libraries/node · https://www.courier.com/pricing · https://www.npmjs.com/package/@trycourier/courier-react · https://www.npmjs.com/package/@trycourier/courier · https://www.npmjs.com/package/@trycourier/react-inbox
- MagicBell: https://www.magicbell.com/docs · https://www.magicbell.com/docs/api/authentication/user · https://www.magicbell.com/docs/libraries/magicbell-react · https://www.magicbell.com/docs/libraries/magicbell-js · https://www.magicbell.com/docs/primitive/broadcast · https://www.magicbell.com/pricing · https://www.npmjs.com/package/@magicbell/react · https://www.npmjs.com/package/@magicbell/magicbell-react · https://www.npmjs.com/package/magicbell-js
- SuprSend: https://docs.suprsend.com/docs/client-authentication · https://docs.suprsend.com/docs/react-sdk · https://docs.suprsend.com/docs/react-sdk-headless-feed · https://docs.suprsend.com/docs/inbox-react · https://docs.suprsend.com/docs/node-trigger-workflow-from-api · https://www.suprsend.com/pricing · https://www.npmjs.com/package/@suprsend/react · https://www.npmjs.com/package/@suprsend/react-core · https://www.npmjs.com/package/@suprsend/node-sdk
- Engagespot: https://www.npmjs.com/package/@engagespot/react-component · https://www.npmjs.com/package/@engagespot/node
