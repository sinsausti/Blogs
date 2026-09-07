---
title: "How to Build a Cybersecurity Risk Register"
description: "Build a useful cybersecurity risk register with defined scope, consistent likelihood and impact criteria, ownership, and treatment tracking."
author: "Sebastian Insausti"
date: "2026-08-22"
tags: ["Infrastructure", "Security"]
canonical_url: "https://insaustis.com/blog/cybersecurity-risk-register.html"
---

# How to Build a Cybersecurity Risk Register

A risk register connects technical exposure to business decisions. It is not a vulnerability list: each entry should describe a plausible threat event, its consequences, existing controls, ownership, and an agreed response.

## 1. Define the Method First

Document the assessment purpose, scope, assumptions, information sources, likelihood and impact scales, risk model, review frequency, and approval authority. Without shared criteria, two teams may score the same situation differently.

Choose a practical scope such as a service, business process, cloud account, or database platform. Include dependencies and third parties that can materially affect it.

## 2. Use the Right Fields

```
Risk ID
Service / process and asset owner
Threat source and threat event
Vulnerability or predisposing condition
Potential business impact
Existing controls and evidence
Likelihood and impact ratings
Current risk rating
Risk owner
Selected response and treatment actions
Action owner and due date
Target or residual risk
Status, last review, and next review
```

The risk owner accepts accountability for the business risk. The action owner performs a treatment task. They may be different people.

## 3. Write Risk Statements Clearly

A useful format is:

> Because [threat] could exploit [condition], [business consequence] may occur, affecting [service, data, or objective].

Example: “Because the backup account shares administrative credentials with production, an attacker who compromises production could delete both systems, preventing database recovery within the approved RTO.”

## 4. Rate Likelihood and Impact Consistently

Define each rating with observable criteria. Likelihood can consider threat capability, intent, exposure, history, and control effectiveness. Impact can include operations, financial loss, safety, privacy, legal obligations, reputation, and dependencies.

```
Likelihood
1 Rare       — exceptional circumstances
2 Unlikely   — possible but not expected
3 Possible   — credible during the review period
4 Likely     — expected or repeatedly observed
5 Almost certain — frequent or imminent

Impact
1 Negligible — minimal disruption
2 Minor      — limited, locally recoverable effect
3 Moderate   — material service or business effect
4 Major      — serious operational or legal effect
5 Severe     — threatens critical objectives or safety
```

These definitions are examples and must be adapted to the organization. If a matrix multiplies ordinal ratings, treat the result as a prioritization aid—not a precise probability or expected monetary loss. Document how ties and high-impact exceptions are handled.

## 5. Choose and Track a Response

- **Mitigate:** reduce likelihood or impact with additional controls.
- **Avoid:** stop the activity that creates the risk.
- **Transfer or share:** allocate part of the consequence through contracts, insurance, or another party; accountability and residual risk remain.
- **Accept:** formally acknowledge the risk within defined authority and review conditions.

Every treatment needs an owner, due date, resources, expected result, and evidence of completion. Record the risk expected after treatment and reassess it when the work is complete.

## 6. Keep the Register Alive

Review risks on a defined schedule and when major changes occur: incidents, migrations, new suppliers, architecture changes, vulnerabilities, regulatory changes, or control failures. Close a risk only when the stated condition no longer exists or an authorized owner accepts the remaining exposure.

Useful reporting includes overdue high risks, treatments without owners, accepted risks approaching review, repeated control failures, and changes in exposure. Avoid rewarding teams for simply lowering scores.

## References

Primary guidance: [NIST SP 800-30 Rev. 1](https://csrc.nist.gov/pubs/sp/800/30/r1/final), [NIST SP 800-39](https://csrc.nist.gov/pubs/sp/800/39/final), and the [NIST Cybersecurity Framework 2.0](https://www.nist.gov/cyberframework).
