# Contributing to OpenAgeProof

OpenAgeProof is an open specification. Contributions are welcome across the full stack — not just code.

## Where help is needed

**Cryptography review** — the blind issuance scheme (§4.4 of the spec) uses Privacy Pass (RFC 9576) primitives. The design needs independent review and audit before any implementation can be trusted.

**Backend development** — a reference issuer implementation does not yet exist. It needs: payment processor integration (Stripe or equivalent), blind signature issuance, fingerprint storage, and a key distribution endpoint.

**Frontend development** — obtaining and presenting tokens is currently a manual process. Browser extensions, mobile apps, and password manager integrations would make the protocol usable by non-technical users.

**Security research** — stress-test the threat model. Try to break it. The open issues in §13 of the spec are starting points.

**Policy and regulatory analysis** — the spec covers known jurisdictional compatibility (§1.2) but this needs ongoing maintenance as laws evolve. Help mapping the protocol to compliance frameworks is valuable.

**Technical writing** — the spec and mission documents need review, clarification, and translation.

**Design** — the user experience is underdeveloped. The protocol is for everyone but currently only usable by people comfortable with technical setup.

## How to contribute

- **Issues** — open a GitHub issue for bugs in the spec, unclear language, or discussion of design decisions
- **Pull requests** — for changes to spec text, include the rationale; spec changes should reference the relevant section
- **Discussion** — use GitHub Discussions for broader design questions

## Spec changes

The spec uses RFC 2119 normative keywords (MUST, SHOULD, MAY). Changes that affect normative requirements need clear justification and should reference existing protocol sections or prior art where relevant.

Breaking changes to the protocol increment the version number and require a new specification section.

## License

By contributing, you agree that your contributions will be licensed under the Apache License, Version 2.0, and you grant the patent license described therein.
