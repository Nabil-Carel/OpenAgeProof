# OpenAgeProof

*An open protocol for privacy-preserving age assurance.*

## The crossfire

Age verification laws are passing in Canada, the UK, the US, the EU, and Australia. The framing is consistent: protect kids from social media, restrict adult content, hold platforms accountable. The public broadly supports these goals. Three out of four Canadians support an under-16 social media ban. Similar majorities support adult content gating in the US and UK.

Adults supporting these laws assume the laws are about kids. They're not personally affected. They're adults, the laws are for minors, and protecting kids feels like an obvious good.

But verification doesn't work that way. To prove a user isn't a kid, platforms have to verify every user's age. The "ban for minors" becomes an "ID check for everyone." Adults who supported the goal find themselves uploading documents, scanning their faces, handing data to third-party verifiers they've never heard of. The implementation arrived in their lap, and it looks nothing like what they thought they were voting for.

## The compliance options on offer are bad

The methods being deployed solve the verification problem by surveilling everyone. Government ID upload to a third-party verifier. Face scans by estimation algorithms. Biometric data handed to whoever's running the latest digital wallet pilot.

In every case, the cost of proving you're an adult is creating a permanent, breachable record that you obtained an age verification credential, a record that doesn't need to exist for the verification itself to work. The UK's Online Safety Act enforcement triggered a sharp spike in VPN downloads. People supported the goal. They didn't sign up for the implementation.

Every system being built today optimizes for compliance, not for users. When the next incident happens, and it will, the affected users will have no recourse. They didn't choose the vendor. They didn't consent to the data retention. They just wanted to read an article or join a community, and now their face or government ID is in a leaked database.

## What OpenAgeProof does

OpenAgeProof is an open protocol that asks: what's the minimum information needed to prove someone is an adult, and how do we deliver that without building surveillance infrastructure?

The answer is a credit card.

Banks already do the work of verifying adulthood. Before issuing a credit card, the bank checks identity, runs credit history, validates the application against fraud databases, and confirms the applicant meets the legal age requirement. Card networks audit this process. Regulators enforce it. The infrastructure to confirm "this person is an adult" exists; it's just locked inside the banking system.

Holding an active credit card is a strong proxy for adulthood. The card is the credential. The protocol's job is just to confirm the user has access to one without revealing which one or who they are. The user pays a small fee to cover operational costs and receives an anonymous token that proves "this person had access to an adult-underwritten card." Nothing more.

The issuer doesn't know which token belongs to which card. The relying party doesn't know who you are. No record exists anywhere that connects you to the credential. This is enforced by cryptography, not policy.

## How it works

OpenAgeProof separates the act of getting a credential from the act of using it.

Once, when you first set up: you visit an OpenAgeProof issuer's website. You enter your card details and complete a small charge, a few dollars, comparable to any online purchase. The issuer produces a token and gives it to you. The token is valid for one year. You're done with the issuer until next year.

You store the token. The protocol doesn't dictate how. You can save it to a password manager, write it down, keep it in a text file. Better storage and presentation tools (browser extensions, mobile apps) can be built on top of the protocol later. None exist today.

When you visit a website that requires age verification: you provide the token. The website checks it locally against the issuer's published verification key. The website doesn't contact the issuer. The verification confirms the token is valid, was issued by a recognized issuer, and hasn't expired.

What the issuer sees and stores: when you obtain a token, the issuer receives a card fingerprint from the payment processor, an opaque identifier that does not reveal your card number, your name, or any other personal information. The issuer ensures that one card produces only one token, ever. After issuance, the issuer's database contains only a fingerprint and an expiration date. It does not contain your token. It cannot link the fingerprint to the token in the wild.

What the website sees and stores: the website receives your token, verifies it, and accepts you as an adult. It sees a random token identifier and an expiration date. Nothing else. The token contains no information about you, your card, or which issuer's service you used.

What the third parties see: the payment processor sees a small charge, just like any online purchase. The issuer never sees you presenting the token at a website. The website never contacts the issuer when verifying.

The token cannot be revoked. This is by design: the issuer doesn't know who you are or which token you carry, so there is no mechanism by which the issuer could identify and invalidate a specific token even under legal compulsion. If your card is ever cancelled, your existing token continues to work until its expiration date. No new token can be issued on that card. The damage window is bounded by the token's natural expiration.

The result: every party in the chain holds the minimum information needed for their step. The payment processor knows you bought something. The issuer knows your card was used to obtain a token. The website knows you have a valid token. No party knows enough to identify you to anyone else.

## The breaches have already started

A short, partial list of identity verification incidents in the last twenty months:

**June 2024:** AU10TIX, used by Uber, TikTok, and Bumble for identity verification, had administrator credentials exposed for over a year, granting access to user verification data including ID images.

**July 2025:** The Tea app, marketed for women's safety verification, was breached. Selfies and driver's license images were leaked.

**October 2025:** Discord's age verification vendor 5CA was breached. Roughly 70,000 government ID images of Discord users were exposed.

**December 2025:** Veriff was compromised, leaking Total Wireless customer verification data.

**February 2026:** An unsecured database tied to identity verification provider IDMerit exposed approximately one billion records across 26 countries: names, dates of birth, addresses, national ID numbers. IDMerit disputes the framing, attributing the data to upstream sources rather than their own systems.

**April 2026:** A California cannabis delivery service leaked 40,000+ customer photo IDs, selfies, and personal details.

**April 2026:** The EU's official age verification app was demonstrably broken in under two minutes by a security consultant who showed PIN bypass and credential extraction.

Some of these are confirmed breaches with leaked data circulating. Others are disputed exposures or architectural failures with no proven exfiltration. The point isn't any single incident. It's the consistency. This is what the first twenty months of the new age verification regime look like, and the laws are still rolling out.

Every age verification vendor handling sensitive data will eventually have an incident. Vendors holding government IDs and biometric data are among the most valuable targets on the internet.

## Cards rotate. Faces don't.

Here's what makes biometric incidents different from every other kind of data breach: *you cannot rotate your face.* When a password leaks, you change it. When a credit card is stolen, you cancel it and a new one arrives in days. When your face leaks, you have it for the rest of your life. When your government ID image leaks, the underlying ID stays valid for years. When your fingerprint leaks, it's gone forever.

Credit cards are the only widely-held credential designed for routine rotation. The banking system assumes cards will be compromised. That's why fraud monitoring exists, why chargebacks exist, why a new card arrives in three to five business days when you call the bank. If a card you used with OpenAgeProof is ever compromised through any channel (phishing, skimming, a merchant breach), you cancel it the way you would anyway. The fingerprint that anchored your token is now associated with a dead card. You get a new card, mint a new token, move on.

No other age verification system has this property. Faces don't rotate. Government IDs change rarely, painfully, and incompletely. Biometric templates, once leaked, follow you forever. OpenAgeProof anchors to the only identity-adjacent credential that users can actually replace when something goes wrong.

## If an OpenAgeProof issuer is breached, you are not affected

The protocol is designed so the issuer doesn't possess data worth leaking. We don't store your card number. We don't store your token. We don't know who you are. There is nothing about you to leak.

No action is required on your part. Your token is still valid. Continue using it as normal.

This is the property no other age verification system can offer. When AU10TIX, 5CA, Veriff, or the next vendor is breached, affected users have to scramble. Change passwords, freeze credit, monitor for identity theft, accept that a leaked face or government ID will follow them forever. With OpenAgeProof, the breach has nothing to leak. The strongest privacy guarantee is the one where the issuer doesn't possess the information in the first place.

## Why now

The laws are passing. We're not going to stop them. The question is what gets built to enforce them.

If the answer is the current crop of identity verification vendors, we're building permanent surveillance infrastructure that will be used for purposes beyond age verification: by governments, by data brokers, by whoever can compel or breach the verifiers' databases. The laws being passed today are the foundation. The infrastructure being deployed will outlast them.

There's a window, while regulatory frameworks are still settling and platforms haven't all picked vendors, to put an alternative on the table. OpenAgeProof is that alternative. It's not the only possible privacy-preserving design, but it's a working one, buildable today, with existing payment infrastructure and existing cryptographic primitives.

## A call to platforms

If you're operating a platform that needs to comply with age verification regulations: the existing vendors are a liability. Every incident is a regulatory inquiry waiting to happen. Every leaked face is a class action. Every "we use industry-leading verification" press release is a hostage to fortune.

Discord swapped vendors in 2025 after one breach. They were already swapping again by early 2026 after Persona's frontend exposure raised questions about that vendor's surveillance posture. So will everyone else. The cycle repeats because the underlying architecture, centralized databases of biometric and identity data, is incompatible with not-leaking-eventually.

Adopt a protocol where the breach has nothing to leak. Where the verifier doesn't know who used it. Where the worst case is "users got a token they don't need anymore" rather than "users' identity documents are circulating on a darknet forum."

## A call to developers

OpenAgeProof is open source. It needs people to make it real.

If you're a backend developer: help build and harden the reference issuer implementation. Improve the API, add support for more payment processors, write integration libraries for the websites that need to verify tokens.

If you're a frontend developer: build the user-facing components that make obtaining and presenting tokens feel less manual than they currently do. Browser extensions, mobile apps, password manager integrations. None of this exists yet, and it should.

If you're a cryptographer: review the protocol design, audit the implementation, find the holes before adversaries do. The cryptography behind this is not exotic, but it has to be right.

If you're a security researcher: stress-test the threat model. Try to break it. Tell us what you find.

If you're a technical writer: improve the spec, write integration guides, translate documents into other languages.

If you're a designer: the user experience matters and is currently underdeveloped. The protocol is for everyone, but right now it's only usable by people comfortable with technical setup. That has to change for adoption.

If you have regulatory or policy expertise: help map the protocol to compliance frameworks across jurisdictions. Help engage with regulators considering approved verification methods.

A privacy-preserving age verification protocol that ships on time and gets adopted is more valuable than a perfect one that doesn't. The work covers the full stack: code, cryptography, design, documentation, advocacy. There is no piece that doesn't need attention.

## A call to users

If a platform asks you to upload your ID or scan your face for age verification, ask why. Ask what they keep, who their vendor is, what the vendor's track record is, and what happens when the vendor is breached.

Then ask whether they support OpenAgeProof. Pressure on platforms is what creates room for alternatives. The vendors lobbying for the current architecture have business reasons to keep things as they are. Users asking for something else is what changes the calculus.

## What OpenAgeProof is not

OpenAgeProof does not solve under-16 social media bans. Detecting minors on a platform is a different problem requiring different infrastructure. The protocol is for adult-content gating where the legal question is "is this user an adult."

OpenAgeProof does not provide identity verification. It does not prove who you are, only that you had access to an adult-underwritten payment credential. This is sufficient for "highly effective" age verification under regulatory frameworks like the UK's, but does not satisfy KYC, AML, or other identity-binding requirements.

OpenAgeProof is not a commercial product. It's an open specification with a reference implementation. Anyone can run an issuer. Anyone can verify tokens. The goal isn't to capture a market. It's to put a non-extractive option on the table before the alternatives lock in.

---

The spec, reference implementation, and design reasoning are open at [openageproof.org](https://openageproof.org). Read the spec. Run an issuer. Build a relying party. Pressure the platforms you use. Adopt the protocol. The point is the infrastructure being built right now will outlast the laws that demanded it. Make sure something privacy-preserving is in the mix before the alternatives are the only thing left.
