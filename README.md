# GHSA-8rwg-g5qm-95xm

Unauthenticated IDOR in Server Actions of [Sepet](https://github.com/umutcandev/sepet), allowing an anonymous caller to read and modify other users' baskets, receipts, and AI assistant conversations.

- **Advisory:** https://github.com/umutcandev/sepet/security/advisories/GHSA-8rwg-g5qm-95xm
- **Severity:** Critical — CVSS 9.1 (`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N`)
- **Weakness:** [CWE-639](https://cwe.mitre.org/data/definitions/639.html) — Authorization Bypass Through User-Controlled Key
- **Affected:** `master` through commit `3847b42`
- **Fixed:** `master` @ `a03eabe`
- **Credit:** Reported by [@vodleux](https://github.com/vodleux) (Reporter)

## Summary

Seven `"use server"` exported functions (`getBasketDetail`, `getReceiptDetail`, `getSavedBasketsForConversation`, `getSavedReceiptsForConversation`, `appendMessages`, `setConversationTitle`, `setConversationStatus`) enforced ownership through a client-supplied `userId` parameter instead of the authenticated session. Because Next.js automatically registers every `"use server"` export as a public, unauthenticated server action endpoint, an anonymous caller who knew a target object's ID and its owner's `userId` could read that user's basket or receipt data, including the storage key of an uploaded receipt image, inject messages into their AI assistant conversation, or rename and change the status of that conversation — all without any session or credentials.

## Root cause

The seven functions were originally written as internal helpers called by page components, with ownership enforced by the caller-supplied `userId` argument rather than by reading the session (`auth()` / `session.user.id`). Because they were exported from a `"use server"` file, Next.js registered each one as its own POST-able server action endpoint, independent of any page-level authentication check. A sibling set of functions in the same files (`getConversation`, the `delete*` and `rename`/`setConversation*` functions) correctly called `auth()` internally and were not affected.

The fix moved the seven vulnerable helpers out of their `"use server"` files into plain internal modules, so they are no longer registered as public server actions and are only reachable through call paths that already enforce authentication.

## Impact

- **Confidentiality — High:** unauthenticated disclosure of another user's basket and receipt records, including purchase details and the storage key of their uploaded receipt image.
- **Integrity — High:** unauthenticated write access, including message injection into a victim's AI assistant conversation history and unauthorized rename or status changes to that conversation.
- **Availability — None.**
- **Scope:** exposure was limited to the single conversation linked to the initially known object and the baskets/receipts attached to it, not a victim's entire account — no exploitable action listed "all objects for this userId."
- **Prerequisite:** exploitation required both a target object's UUID and its owner's `userId`. Neither was guessable, but the vulnerable endpoints themselves partially exposed both (the owner's `userId` appeared in `getBasketDetail`/`getReceiptDetail` responses and in the plaintext storage path of uploaded images).

## Timeline

- Early October 2026 — Reported privately to the maintainer via `security@trysepet.com`
- Before 2026-10-14 — Fix merged to `master` (`a03eabe`)
- 2026-10-14 — Advisory published (Critical, 9.1)
- Pending — CVE ID requested, not yet assigned
