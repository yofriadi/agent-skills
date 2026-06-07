# Worked Example: Follow-on Research for ChatGPT/Codex OAuth Grounding

This is a worked instance of `follow-on` research. It assumes the main `port` research already exists and a narrower question remains: **should the Rust port start with an OAuth-based ChatGPT/Codex integration as the first grounded LLM provider?**

**Parent research:** `examples/port-pi.md`
**Why this is `follow-on` and not a fresh `port`:** the broad translation and phasing decisions are already made. This stream only needs to ground the first-provider choice and the auth contract it imposes.

## Scope & Boundaries

- **Parent research:** `examples/port-pi.md`
- **Decision this stream grounds:** whether the first provider integration should target an OAuth-based ChatGPT/Codex surface, and what auth/runtime constraints that adds to Phase 1.
- **In scope:** official auth flow shape, redirect/callback requirements, token exchange expectations, env/config surface, concrete public examples of OAuth callback handling.
- **Out of scope:** full multi-provider ranking, non-OAuth providers, UI polish, enterprise org provisioning, billing/commercial analysis.
- **Non-functional requirements:** no secrets embedded in binaries, redirect URI must be configurable, license-compatible examples only, auth flow must be testable in local/dev environments.
- **Cross-cutting concerns:** security (token storage, callback validation), observability (failed auth traces), portability (local callback vs hosted callback), future provider abstraction.

## Grounded External Contract

### Official OpenAI / ChatGPT auth facts

- OpenAI documents an OAuth authorization-code token exchange with `grant_type`, `client_id`, `client_secret`, `code`, and `redirect_uri` fields in the request body.
  - Source: `https://developers.openai.com/api/docs/actions/authentication`
- OpenAI documents ChatGPT callback URL shapes under both `chat.openai.com` and `chatgpt.com`:
  - `https://chat.openai.com/aip/{g-YOUR-GPT-ID-HERE}/oauth/callback`
  - `https://chatgpt.com/aip/{g-YOUR-GPT-ID-HERE}/oauth/callback`
  - Source: `https://developers.openai.com/api/docs/actions/authentication`
- OpenAI Apps SDK auth documentation shows the metadata surface ChatGPT expects from an OAuth 2.0 server, including `issuer`, `authorization_endpoint`, `token_endpoint`, supported auth methods, and PKCE metadata.
  - Source: `https://developers.openai.com/apps-sdk/build/auth`

### Concrete public implementation references

- `danny-avila/LibreChat` (MIT) tests authorization-code token exchange inputs, including `code`, `redirect_uri`, and token exchange method selection.
  - Source: `https://github.com/danny-avila/LibreChat/blob/main/packages/api/src/oauth/tokens.spec.ts`
- `mcp-use/mcp-use` (MIT) contains real callback-path handling logic for deriving OAuth proxy behavior from `/oauth/callback` URLs.
  - Source: `https://github.com/mcp-use/mcp-use/blob/main/libraries/typescript/packages/mcp-use/src/auth/callback.ts`

## Implications for the Rust port

### Phase-1 provider contract deltas

Compared with the parent `port` research, choosing this provider first adds explicit auth requirements:

1. **Redirect URI management**
   - The provider integration cannot be a pure API-key wrapper.
   - It needs configurable callback URL registration and validation.
2. **Authorization-code exchange**
   - The Rust provider crate must model `code`, `redirect_uri`, client credentials, and token exchange failures as first-class error cases.
3. **Token persistence**
   - Refresh/access tokens become persistent sensitive state; storage policy must be decided before calling the integration production-ready.
4. **Local-dev story**
   - Because callback URLs are fixed inputs to the auth flow, local development needs a repeatable callback strategy before the integration is easy to test.

### Suggested additions to the parent translation table

| Concern | Rust-side need |
|---|---|
| OAuth 2.0 auth-code flow | dedicated auth module or vetted crate with PKCE support |
| Redirect callback listener | local HTTP callback receiver or externally hosted callback bridge |
| Token persistence | encrypted-at-rest store or OS keychain abstraction |
| Config surface | env/config keys for client ID, client secret, redirect URI, issuer/endpoint metadata |

## Risk Register

| Severity | Risk | Mitigation |
|---|---|---|
| High | OAuth callback flow adds infra and local-dev complexity before core provider parity is proven | spike the callback + token exchange path first; do not hide this inside the generic provider trait |
| High | Token storage requirements may force an early security design decision | define storage policy up front; keep Phase 1 non-production unless storage is settled |
| Med | ChatGPT-hosted callback URL constraints may not match a purely local CLI workflow | test a minimal local-dev callback path in `/spike` before locking the provider order |
| Med | OAuth metadata / endpoint assumptions drift from official docs | pin the implementation to the documented metadata fields and re-verify against official docs during implementation |

## Handoff Recommendation

- **Recommended next step:** `/spike`
- **Spike target:** minimal Rust auth sandbox proving three things end-to-end:
  1. build the authorization URL,
  2. receive/process the callback,
  3. exchange the authorization code for tokens.

## Why this example matters

Use this pattern whenever the main research already answered the large architectural question, but another narrower stream is needed to ground the **first real integration choice**. Keep it separate from the main `RESEARCH.md`; do not overwrite the parent research.