# Repository split: open issue review

Reviewed 2026-09-10: all 21 open issues and their available comments in `thibmeu/http-message-signatures-directory`. Issues #62 and #96 were subsequently copied to the registry repository with their original text and source attribution.

**#62 and #96** were copied to [registry #1](https://github.com/thibmeu/draft-meunier-webbotauth-registry/issues/1) and [registry #2](https://github.com/thibmeu/draft-meunier-webbotauth-registry/issues/2). The repository owner will close **#110** separately. Keep 16 protocol issues here, keep #60 here with a registry follow-up, and route #19 to separate robots.txt work when a destination exists.

“Directory” refers to key discovery, which is part of the protocol draft; it does not mean the registry of Signature Agent cards.

| Issue | Recommendation | Reason |
| --- | --- | --- |
| [#11: Consider negative test vectors](https://github.com/thibmeu/http-message-signatures-directory/issues/11) | Keep with protocol | Protocol verification failures and negative test vectors. |
| [#19: Consider integration of `Signature-Agent` with robots.txt](https://github.com/thibmeu/http-message-signatures-directory/issues/19) | Separate robots.txt work | Discussion proposes a separate robots.txt extension/registry draft. Keep here pending that destination; this is not the Signature Agent registry. |
| [#27: Consider describing chaining](https://github.com/thibmeu/http-message-signatures-directory/issues/27) | Keep with protocol | Key/certificate chaining in the protocol. |
| [#45: Directory endpoint should support nonce signing](https://github.com/thibmeu/http-message-signatures-directory/issues/45) | Keep with protocol | Nonce signing by the protocol key-directory endpoint. |
| [#52: HTTP Message signature to allow filtering based on key thumbprint](https://github.com/thibmeu/http-message-signatures-directory/issues/52) | Keep with protocol | Filtering the protocol key-directory endpoint. |
| [#60: Consider a mechanism for identifiers (and abuse reporting)](https://github.com/thibmeu/http-message-signatures-directory/issues/60) | Keep; split registry follow-up | Request/customer identifiers belong to the protocol; advertised abuse-reporting metadata belongs to the registry. Split that portion into a registry follow-up. |
| [#62: Support registries linking to registries](https://github.com/thibmeu/http-message-signatures-directory/issues/62) | Copied to registry #1 | Registry-to-registry discovery and trust relationships. |
| [#67: Remove rfc7638 requirement from `keyid` and make it match `kid`](https://github.com/thibmeu/http-message-signatures-directory/issues/67) | Keep with protocol | Signature keyid semantics and matching discovered keys. |
| [#93: Implementers to fall back on existing verification methods](https://github.com/thibmeu/http-message-signatures-directory/issues/93) | Keep with protocol | Verifier behavior and fallback to existing verification methods. |
| [#96: [registry] Rename `Signature-Agent Card` to `Registry Card`](https://github.com/thibmeu/http-message-signatures-directory/issues/96) | Copied to registry #2 | Name of the card defined by the registry draft. |
| [#100: proposal: distinguish same-origin TLS binding from delegated signing](https://github.com/thibmeu/http-message-signatures-directory/issues/100) | Keep with protocol | Key possession and authority-binding proofs in the protocol; mirror any resulting guidance in registry security considerations. |
| [#102: [directory] DNS-rooted key discovery: bind the Signature-Agent key with a DANE-EE TLSA record under a DNSSEC-signed name](https://github.com/thibmeu/http-message-signatures-directory/issues/102) | Keep with protocol | Alternative key discovery and domain binding; the former directory draft is now part of the protocol. |
| [#109: Key format](https://github.com/thibmeu/http-message-signatures-directory/issues/109) | Keep with protocol | Key formats and Signature-Agent discovery types; cross-reference the registry if CIMD support moves there. |
| [#110: Drop the registry?](https://github.com/thibmeu/http-message-signatures-directory/issues/110) | Owner will close | Whether the registry/card draft should exist; the author’s reply explicitly distinguishes its scope from the protocol. |
| [#114: Binding to humans](https://github.com/thibmeu/http-message-signatures-directory/issues/114) | Keep with protocol | Protocol privacy guidance about human identity. |
| [#115: Tracking text is backwards](https://github.com/thibmeu/http-message-signatures-directory/issues/115) | Keep with protocol | Protocol identity and tracking model. |
| [#116: Shorter names 🚲🏠](https://github.com/thibmeu/http-message-signatures-directory/issues/116) | Keep with protocol | Protocol well-known key endpoint and media type names. |
| [#119: 301 on a Signature-Agent](https://github.com/thibmeu/http-message-signatures-directory/issues/119) | Keep with protocol | Redirect handling during Signature-Agent key discovery. |
| [#126: [protocol] `Unencoded-Digest` vs `Content-Digest`](https://github.com/thibmeu/http-message-signatures-directory/issues/126) | Keep with protocol | Digest field covered by request signatures. |
| [#128: 5.2.1 label-keying MUST vs the Appendix E vectors, and whether 5.2.2 already carries the property](https://github.com/thibmeu/http-message-signatures-directory/issues/128) | Keep with protocol | Signature label matching and protocol test vectors. |
| [#133: §5.2: "expiry" is separated from its antecedent and reads as key expiry](https://github.com/thibmeu/http-message-signatures-directory/issues/133) | Keep with protocol | Signature lifetime wording in signing requirements. |

The protocol destination is `webbotauth/draft-ietf-webbotauth-httpsig-protocol`. The registry destination is `thibmeu/draft-meunier-webbotauth-registry`.
