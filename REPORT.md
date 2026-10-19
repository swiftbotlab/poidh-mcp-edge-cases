# poidh MCP + WebMCP: edge-case report

Bounty: "test POIDH webmcp + mcp. agents welcome!". It was posted as mainnet #41 (canceled at about 20:30 UTC) and as Base #1495 (on-chain id 509).
Tested by Swift Bot Labs. Every request below was read-only or a dry run. Nothing was signed, sent or connected to a wallet.

## Summary

- **20 findings: 9 bugs, 6 inconsistencies, 5 UX/protocol issues.**
- Most serious: for v2-era bounty ids, the `prepare_*` tools build v3 calldata that targets a different bounty, and the `/tx` approval page shows the wrong bounty's details (#1). Completed bounties are reported as `voting` (#2). `find_bounties` hides bounties that share a numeric id with a bounty on another chain (#3) and does not escape SQL LIKE wildcards (#4). Internal Prisma and Postgres errors are returned to callers (#5). On mainnet, every bounty id that starts with "9" redirects to #11, so `get_bounty` fails for those ids (#6).
- The core happy paths work: Base frontend-id-to-contract-id offsets, claim id mapping, calldata (checked against the verified ABI), status guards, amount parsing and the proof-URI scheme allowlist.

## Environment

- Date: 2026-10-19, 20:26-21:10 UTC.
- MCP endpoint: `https://poidh.xyz/api/mcp`. Streamable HTTP, stateless, JSON responses. `serverInfo` is `poidh 0.1.0`. The negotiated protocol version is `2025-11-25`.
- Source read for context: `picsoritdidnthappen/poidh-app` at `bd3a3db` (2026-10-19 13:06 -0700), `@modelcontextprotocol/sdk 1.32.1`.
- Client: a plain JSON-RPC client written with the Python stdlib. MCP responses were treated as untrusted data.
- WebMCP: Chrome 155 on Linux, in a separate tab. Chrome has no native `document.modelContext` or `navigator.modelContext` without a flag. To capture poidh's real `registerTool()` calls, a recording mock `document.modelContext` was injected before page scripts ran (`Page.addScriptToEvaluateOnNewDocument`). 9 tools were registered. Only `find_bounties` and `get_bounty` were executed. `/tx` links were opened to read them, and no wallet was connected.
- On-chain checks used `eth_call` through publicnode RPCs against v3 `0xE731…4232` (mainnet) and `0x5555…1719` (Base).

## Findings

### 1. [bug] For v2-era bounty ids, prepare_* builds v3 calldata that targets a different bounty

- Request: `tools/call prepare_resolve_vote {"chain":"base","id":176}`
- Actual:
  - The response is `resolveVote(176)` (`0x30f4ccac…b0`) sent to the v3 contract `0x5555…1719`.
  - Frontend Base #176 is the v2 bounty "why do you appreciate Vitalik? 💙".
  - v3 `bounties(176)` on chain is "find me in Seattle", which is frontend Base #1162.
  - The returned `signUrl` (`/tx?action=resolve_vote&chain=base&id=176`) shows the v2 bounty title, its claim "Vitalik gave me a shot" and its claimant, above calldata for a different bounty.
  - The same thing happens for base #432, #529 and #840, and arbitrum #45. The DB still has `isVoting=true` with past deadlines for all of them.
  - `prepareAction` never checks whether the frontend id is v2 or v3 (Base ≤ 986, Arbitrum ≤ 180). Join, submit and accept are only protected because those bounties happen not to be "open".
- Expected: an error such as "Bounty 176 on base is a v2 bounty; poidh v3 tools cannot act on it".
- Fix: in `prepareAction` and in `/tx`, reject ids ≤ `*_LAST_PRE_V3_BOUNTY` before building anything.

### 2. [bug] Completed bounties with resolved votes are reported as "voting"

- Request: `find_bounties {"query":"More positive vibes"}` returns `mainnet #32 status "voting"`. WebMCP `get_bounty {"chain":"base","id":176}` returns `"status":"voting"`.
- Actual:
  - `/mainnet/bounty/32/data` says `status: completed`, `voteResolved: true`, and a `winningClaim` is set. Base #1418, #1460 and mainnet #30 show the same mismatch.
  - In the full crawl, 382 of 1,129 bounties are labelled `voting`.
  - `prepare_resolve_vote {"chain":"mainnet","id":32}` still returns a transaction. With `from`, its simulation reports `NoVotingPeriodSet`.
- Cause: `bountyStatus()` checks `isVoting` before `inProgress`, and the DB keeps `isVoting=true` after a vote is resolved.
- Fix: derive the status the way `/data` does (`getVotingState` / `votingInProgress`), and refuse resolve or vote when the bounty is not in progress.

### 3. [bug] find_bounties hides bounties that share a numeric id with another chain's bounty

- Request: `find_bounties {"query":"the","limit":50,"cursor":"1718520723"}`
- Actual:
  - The results include arbitrum #3 and #8.
  - Base #3 "the greatest team you've ever seen" and base #8 "upload the doginme can 🐶" are missing. Both are completed, not banned, and contain "the".
  - Direct searches ("the greatest team", "doginme") do find them.
  - In the full `%` crawl, base #3, #6, #8 and #9 are missing while arbitrum #3, #6, #8 and #9 are present.
- Cause: `fetchByKeyword` uses `distinct: 'id'`, but `id` is only unique per chain.
- Fix: use `distinct: ['id','chainId']`, or remove the distinct.

### 4. [bug] Search does not escape SQL LIKE wildcards

- Request: `find_bounties {"query":"%%%%"}` and `{"query":"____"}`
- Actual:
  - Both match every bounty. The first result is base #1495, whose text contains no `%` or `_`.
  - `{"query":"%"}` pages through the whole table: 1,129 bounties in 23 calls.
  - A user searching "100%" or "snake_case" gets wrong results.
- Fix: escape `\`, `%` and `_` before using `contains`.

### 5. [bug] Internal database errors are returned to the caller

- Requests and responses:
  - `find_bounties {"cursor":"abc"}` (also `" 1792179047"` and `"' OR 1=1 --"`) returns the full ``Invalid `prisma.bounties.findMany()` invocation`` dump, including the where/include/distinct/orderBy shape.
  - `find_bounties {"query":"\u0000"}` returns `invalid byte sequence for encoding "UTF8": 0x00`. WebMCP does the same.
  - WebMCP `get_bounty` with id `2147483648` returns "Value out of range for the type: … integer".
  - WebMCP `get_bounty` with id `1e21` returns a Prisma dump.
  - WebMCP `get_bounty` for `mainnet` #1495 returns a raw "No record was found for a query".
- Expected: short user-facing errors.
- Fix: validate `cursor` with `/^\d+$/`, strip or reject NUL, bound ids to int32, and map Prisma errors to messages.

### 6. [bug] Mainnet ids starting with "9" redirect to #11, which breaks get_bounty

- Request: `get_bounty {"chain":"mainnet","id":9}` (also 90, 99, 999 and 9998)
- Actual:
  - The tool returns `Unexpected token '<', "<!DOCTYPE "... is not valid JSON`.
  - The cause: `/mainnet/bounty/9/data`, `/90/data`, `/99/data`, `/9998/data` and `/99990/data` all return `301 → /mainnet/bounty/11`, which is HTML.
  - Mainnet #9 exists on chain ("Encrypt The Mempool!").
  - WebMCP `get_bounty` for the same id works, because it uses tRPC.
  - When mainnet reaches #90, bounties #90–#99 will open #11 on the site and fail in MCP.
- Fix: anchor the redirect to exactly `/mainnet/bounty/9`. In `fetchPoidhData`, use `redirect:'error'` and check the content-type.

### 7. [bug] signUrl returns 414 for valid long inputs

- Request: `prepare_create_bounty {"chain":"base","title":"t","amountEth":"0.001","description":"漢"×5000}`. This is within `maxLength 5000`.
- Actual: the `signUrl` is 45,099 characters long, and a GET on it returns `414 URI_TOO_LONG` from Vercel. The limit sits at about 32 KB: a 32,000-character URL returns 200 and a 33,000-character one returns 414.
- Fix: cap the encoded size, or issue a short server-stored approval link.

### 8. [bug] proofUri has no maximum length

- Request: `prepare_submit_claim` on base #1495 with a 500,000-character https `proofUri`.
- Actual: the tool accepts it and returns 500,324 bytes of `createClaim` calldata and a 500 KB `signUrl` (which returns 414). That much calldata costs at least 8M gas on mainnet, before storage.
- Fix: add `maxLength` (for example 2048) to `proofUri` in the MCP, WebMCP and `/tx` schemas.

### 9. [bug] The MCP transaction and its signUrl encode different strings

- Request: `prepare_submit_claim {"chain":"base","id":1495,"title":"  padded title  ","description":"  padded desc  ",…}`
- Actual:
  - The MCP calldata stores the untrimmed strings.
  - `/tx` (`parseActionParams`) trims them and builds different calldata.
  - With `"title":"   "`, MCP returns a transaction and a signUrl, but the signUrl page says "invalid link … title: string must contain at least 1 character(s)".
- Fix: use `z.string().trim().min(1)` in the MCP schemas, as `/tx` already does.

### 10. [inconsistency] "Any status" search actually excludes canceled bounties

- Request: `find_bounties {"query":"webmcp"}`
- Actual:
  - Only base #1495 is returned. Mainnet #41, this same bounty and now canceled, is not.
  - Exact-title searches for canceled mainnet #7, #8, #11 and #12 return nothing.
  - The tool description says it "searches titles and descriptions across all bounties (any status)".
- Fix: change the description to say canceled bounties are excluded, or add a `status` filter.

### 11. [inconsistency] Bounty id 0 is listed but cannot be fetched

- Request: `find_bounties {"query":"get out in the woods"}` returns `mainnet #0` with url `/mainnet/bounty/0`. Base #0 also exists.
- Actual: `get_bounty {"chain":"mainnet","id":0}` and every `prepare_*` call reject it with "Invalid input at id", because the schemas use `exclusiveMinimum: 0`. WebMCP uses `minimum: 1`.
- Fix: accept 0 (non-negative ids).

### 12. [inconsistency] "degen" is a valid chain, but every degen lookup returns "Not found."

- Request: `get_bounty {"chain":"degen","id":1}` (also 41, 1197 and 1198)
- Actual: "Not found." for every id. No degen bounty appears in the full crawl, although the leaderboard still counts degen points. WebMCP's error message lists only mainnet, arbitrum and base.
- Fix: remove `degen` from the enum, or serve the historical data.

### 13. [inconsistency] MCP and WebMCP/`/tx` validate ids and chains differently

- Actual:
  - WebMCP `get_bounty` uses `Number(input.id)`. The ids `"1495"`, `"0x5d7"` and `"1.495e3"` all return bounty 1495, and `id: true` returns bounty #1.
  - `/tx?action=join_bounty&chain=base&id=0x5d7` renders "Add 0.001 ETH to bounty 1495".
  - MCP rejects `"1495"` as invalid.
  - WebMCP chain `"__proto__"`, `"constructor"` or `"toString"` passes the `chains[chain]` guard and returns a raw zod issue array.
- Fix: require integer numbers (or a `^\d+$` string), and use `Object.hasOwn(chains, chain)`.

### 14. [inconsistency] The WebMCP tool surface does not match the docs or MCP

- Actual:
  - skill.md says the page registers "the same tools", but it registers 9: there is no `get_account` and no `get_leaderboard`.
  - WebMCP `find_bounties` ignores a query that is not a string: `{"query":41}` returns the open list without an error.
  - WebMCP has no length limit on the query (5,000 characters are accepted; MCP stops at 100).
  - Rewards are formatted `"0.0108 eth"` in WebMCP and `"0.0108 ETH"` in MCP.
- Fix: align the schemas and formatting, or document the differences.

### 15. [inconsistency] "Not found." does not say which chain or id

- Request: `get_bounty {"chain":"mainnet","id":1495}`
- Actual: "Not found." Id 1495 exists on Base. The prepare path already says "Bounty X not found on <chain>."
- Fix: return "Bounty 1495 not found on mainnet (ids are per chain)".

### 16. [UX] No rate limiting observed

- Test: 60 `tools/call get_leaderboard` requests, 12 in parallel.
- Actual: all 60 returned 200 within 11.8 s, with no `Retry-After`. Each request builds a fresh server and runs a DB query. Together with #4, this makes it easy to load the database.
- Fix: per-IP limits at the edge.

### 17. [UX] The transaction object does not match JSON-RPC types

- Actual: `transaction.chainId` is a number (`8453`), but `eth_sendTransaction` expects a QUANTITY hex string. The object also carries `valueWei`, `valueEth` and `functionName`, while the docs present it as `{chainId,to,data,value}` for `eth_sendTransaction`. `walletSendCalls` is already correct (`"0x2105"`).
- Fix: use a hex `chainId` and move the extra fields to a sibling object.

### 18. [UX] Some simulation errors have no hint

- Request: `prepare_resolve_vote {"chain":"mainnet","id":32,"from":…}`
- Actual: `simulation: {ok:false, error:"NoVotingPeriodSet"}` with no hint.
- Fix: add hints for every custom error in the v3 ABI.

### 19. [UX] get_account returns an empty profile for any address

- Request: `get_account {"address":"0x0000000000000000000000000000000000000000"}`, or a mixed-case address with a wrong checksum such as `0x572Fef…8087`.
- Actual: a full `accountType:"user"` profile, with nothing to show the account has never used poidh. EIP-55 checksums are not checked.
- Fix: add `exists:false`, and validate the checksum when an address is mixed case.

### 20. [protocol] Smaller spec issues

- The server advertises `tools.listChanged:true`, but it is stateless and cannot send that notification.
- It accepts JSON-RPC batches after negotiating `2025-11-25`. Batching was removed in 2025-06-18.
- An unknown tool returns an `isError` result instead of a JSON-RPC `-32602` error.
- A client that sends only `Accept: application/json` gets 406, even though the server only ever replies with JSON. The spec allows this, but it trips up curl users.

## Worked as expected

- Base offsets:
  - `prepare_submit_claim {"chain":"base","id":1495,…}` returns `createClaim(509, title, description, uri)`, selector `0x83ffc9f2`. This matches the verified ABI.
  - `prepare_accept_claim {"chain":"base","id":1492,"claimId":8868}` returns `acceptClaim(506, 3552)`, and on chain `claims(3552).bountyId == 506`.
- Guards give clear messages in these cases:
  - a canceled bounty
  - joining a solo bounty
  - the issuer claiming or joining their own bounty
  - a non-issuer accepting a claim
  - a claim from another bounty or a missing claim
  - voting after the deadline (it suggests resolving instead)
- Amount parsing rejects `1e-5`, `.5`, `-1`, `0`, `0x10`, `1,5`, Arabic-Indic digits, 19 decimal places and `0.01 ETH`. The minimums of 0.001 (create) and 0.00001 (join) are enforced.
- The proofUri allowlist blocks `javascript:`, `data:`, `http:`, `file:` and `ftp:`. Leading whitespace is normalised.
- A contract sender (USDC on Base) gets a clear EOA error. Simulation reports `InsufficientFunds` and `NothingToWithdraw` with hints.
- The Origin check works: an evil origin, `null` or `http://poidh.xyz` gets 403, and a server-side client that sends no Origin is accepted.
- Protocol basics work:
  - GET and DELETE return 405.
  - Bad JSON returns -32700.
  - A wrong content-type returns 415.
  - An unknown method returns -32601.
  - An unknown protocol version falls back to `2025-11-25`.
  - A bad `MCP-Protocol-Version` header returns 400 with the list of supported versions.
- MCP schema bounds are enforced: limit 0, 51 and `"10"`; leaderboard pages 0 and 11; a 201-character title; a 5,001-character description; a 101-character query.
- Every read result carries the user-generated-content warning.
