---
layout: post
title: "Signed Is Not Trustworthy: Grading the Evidence Behind a Robot's Action"
date: 2026-06-30 09:00:00 +0200
categories: [runtime, embodied-ai]
tags: [embodied-ai, robotics, agentrust, evidence-quality, attestation, issuer-independence, eu-ai-act]
description: >-
  Every post in this series produced a signed record. None of them is worth the same. One scale to read agent-to-robot evidence honestly: how close the signer sat to the machine, whether anyone independent could contradict it, and what hardware stood behind the key.
canonical_url: "https://governance.ai-mvp.com/2026/06/30/grading-robot-action-evidence/"
---

*Every post in this series ended with a signature. A signed manifest, a signed audit trail, a signed sensor record. Here is the uncomfortable part: a valid signature tells you the record was not altered after it was signed. It tells you almost nothing about whether the record is worth trusting. This post is one scale for the difference.*

> Series: The Engineering of AI Governance, Post 9
> Continues from [Trusting the Skills You Didn't Write](https://governance.ai-mvp.com/2026/06/19/trusting-the-skills-you-didnt-write/)
> Status: this consolidates vocabulary introduced across Posts 4 to 6 into one grading model. It extends AgenTrust's existing primitives (manifest, cMCP, TRACE); the specific grading scheme here is a design proposal for discussion, not a shipped feature.

---

Across the last five posts I introduced graded-evidence language three separate times without ever tying it together. Post 4 split trust into governance layers, mission, machine control, functional safety. Post 5 introduced a per-link **assurance class** for hardware backing, `gpu-cc-attested` down to `software-only`. Post 6 made a robot's sensors testify and called the readings "physical truth." Each was locally correct. Together they leave a reader unable to answer the only question an investigator actually asks: *given this signed record in front of me, how much should I believe it?*

So this post does one thing. It collapses that scattered vocabulary into a single way to read an evidence record, and it is deliberately built so that a high score never quietly becomes a claim the evidence cannot support.

## First, which layer is even talking?

Before grading a record, know what kind of claim it is making. Post 4 named three layers, and they do not substitute for each other:

| Layer | Question it answers | Who owns it |
|---|---|---|
| Mission governance | Was this request allowed, by which agent, under which policy? | cMCP + Agent Manifest + Cedar |
| Machine control | Was this motion acceptable against the *current* physical state? | Independent controller |
| Functional safety | How is harm physically prevented when everything above fails? | Certified systems (out of scope, on purpose) |

A record from the mission layer that says `allow` is not evidence about motion. A record from the controller is not proof of physical safety. The first mistake in reading agent-to-robot evidence is letting a claim from one layer answer a question that belongs to another. Keep them apart, then grade whichever one you are holding.

## Then, grade the link with three questions

Any single evidence link, an authorization, an outcome, a sensor frame, can be graded on three independent axes. They do not collapse into one number, and pretending they do is how "signed" gets mistaken for "trustworthy."

**1. Proximity: how close did the signer sit to the machine?**

The further the signer is from the controller boundary, the more can go unobserved between the claim and the metal.

| Level | Signer | What it means |
|---|---|---|
| E0 | The agent or application itself | Self-reported. A valid signature on the accused's own account. |
| E1 | A gateway that saw the response | Observed, but no controller-side party attested it. |
| E2 | A separate observer | An independent watcher signs what it saw. |
| E3 | A controller-side adapter | A signer sitting at the action-server or PLC boundary. |
| E4 | The controller's own key | The controller, OEM module, or safety authority signs under its own key. |

An E0 self-report is not a weak green. It is red. A perfectly valid signature from the party with the most reason to lie is the *lowest* grade of evidence, not a passing one.

**2. Independence: could anyone contradict it?**

Proximity is worthless if the same party controls both ends. A record where the requester and the outcome signer are the same key, or the same company, cannot corroborate itself, no matter how close to the machine it sat.

So every outcome link should declare its relationship to the request signer: `same_key`, `same_party`, `same_contract_group`, or `independent_party`. An E3 adapter run entirely by the operator is still same-party evidence. That distinction has to be on the face of the record, not inferred later by a tired investigator.

**3. Assurance class: what hardware stood behind the key?**

This is Post 5's axis, and it is genuinely separate from the first two. A signature is only as honest as the environment that held the key. From Post 5, per link: `software-only`, `trustzone-signed`, `cpu-tee-attested`, `gpu-cc-attested`, plus what it protected (confidentiality, integrity, identity) and which authority attested it. A `software-only` signature can be forged by a compromised runtime before signing. A hardware-attested one cannot, without breaking the hardware.

Three axes, orthogonal. A record can be close to the machine (E4) and still be same-party and still be `software-only`. Read all three, or you have not read the record.

## The verdict, and the line it must never cross

Combine the axes into one human-readable verdict so a reviewer does not have to do the arithmetic under pressure: an E4, `independent_party`, `cpu-tee-attested` outcome reads *high-integrity, independent, hardware-backed*. An E0, `same_party`, `software-only` outcome reads *self-reported, not audit-grade*, and displays red.

But here is the spine of the whole scheme, and where I have to correct my own Post 6. However high a record scores, two claims stay off the table by default:

- **`physical_completion_claim: none`.** A controller reporting `success` means the controller reported success under its own semantics. Sensor readings raise the cost of lying, they do not become "physical truth." I called them that in Post 6, and it was a word too strong. Even an E4, hardware-backed sensor record is graded evidence that a motion was *witnessed*, not proof that the world changed the way the agent intended.
- **`completeness_claim: not_proven`.** Two signed receipts prove the integrity of what was presented. They do not prove nothing was withheld. Per-issuer hash chains and gap detection are what let a reviewer see when a record is missing, and their absence is itself a finding.

The discipline is the product. A grading scale that let a high score imply physical completion or completeness would be worse than no scale, because it would launder confidence the evidence never earned.

## Reading a record, end to end

Put it together on the asymmetric chain from Post 5. A cloud reasoning step signs its decision `gpu-cc-attested`, `independent_party` at the datacenter. The plan crosses to the robot. The edge actuation signs `trustzone-signed`, and its outcome link is E3, adapter-attested, but `same_party`, because the operator runs the adapter. The sensor witnesses from Post 6 attach at E2 to E3 depending on who holds their keys.

A reader who grades each link sees the honest shape immediately: strong and independent in the cloud, stepping down to same-party and software-rooted at the hand, with no physical-completion claim anywhere. That is not a failure of the system. That is the system telling the truth about itself, which is the only reason anyone outside the operator should believe a word of it.

## What this means under EU law

The same requirements from earlier posts bite here. **Article 12** wants automatically logged events and **Article 19** wants them kept, but a log an operator can trivially forge is record-keeping in name only. **Article 15** asks for assurance that the deployed system is the approved one. A grading scale that surfaces proximity, independence, and assurance class is exactly how you answer an auditor without hand-waving: not "we have logs," but "here is what each record can and cannot support, on its face."

## Where I want to be wrong

This scale is a proposal, and the axes are the part I most want stress-tested. If you review agent-driven robot incidents, or build controllers, or sign attestations for a living: does grading proximity, independence, and assurance class separately match how you actually weigh evidence, or is there a fourth axis I am folding in by accident? The [example is public](https://github.com/agentrust-io/examples/tree/main/industrial-embodied-ai), and the objections are the point.
