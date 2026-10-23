# Fraud Detector — Scam & Phishing Detection System

A modular fraud detection system I built to catch scam messages and phishing links — the kind that impersonate a bank, threaten account suspension, or promise an unrealistic investment return. It looks at a message, a URL, or both, and gives back a risk score, a verdict, and plain-language reasons for why.

🔗 **Live demo:** [frauddetector.in](https://frauddetector.in)
👤 **Built by:** Piyush Verma — [LinkedIn](https://www.linkedin.com/in/-piyush-verma) · [GitHub](https://github.com/Piyush-Verma-07)

> The production codebase lives in a private repository, since this is a live product I'm continuing to run and develop. This repo is the public-facing writeup — what it does, how it's built, and what I learned building it. Happy to grant read access or walk through the code with anyone reviewing it for a hiring decision.

---

## Why I built it this way

A message containing the word "verify" isn't automatically a scam. A weird-looking URL isn't automatically malicious either. Most of the simple detectors I looked at before starting this just keyword-match and call it a day, which means they're wrong a lot — in both directions.

So instead of one big if/else chain, Fraud Detector treats detection as evidence-gathering: several independent checks run against the input, each contributing weak or strong evidence, and a separate aggregation step combines everything into a score and a verdict. No single signal decides the outcome on its own.

It's also deliberately **not** an LLM wrapped in a chat UI. Everything is rule-based and deterministic, which means every verdict is explainable and reproducible — ask it twice, get the same answer, and get a real reason why, not a hallucinated one.

## See it in action

| You paste this | You get this |
|---|---|
| `Hey, are we still meeting for lunch tomorrow at 1pm?` | **Risk Score: 0 — SAFE** |
| `You have won a free gift card! Claim it now at www.prize-rewards-today.com before it expires` | **Risk Score: 75 — HIGH RISK** — urgency language, prize/reward scam pattern, suspicious URL |
| `Dear customer, your KYC has expired. Click here to verify immediately... http://bit.ly/verify-kyc-now` | **Risk Score: 93 — HIGH RISK** — fear-based language, urgency, account-verification pattern, shortened link, plain HTTP, uncommon TLD |

### Screenshots

**Homepage**
![Homepage](screenshots/01-homepage.jpg)

**A safe message, left alone**
![Safe result](screenshots/02-safe-result.jpg)

**A prize scam, flagged**
![High risk prize scam](screenshots/03-high-risk-prize-scam.jpg)

**KYC phishing with a shortened link, flagged**
![High risk KYC phishing](screenshots/04-high-risk-kyc-phishing.jpg)

---

## How a request actually flows through it

```text
             User Input
                 |
                 v
          Input Validation
                 |
                 v
          Analysis Engine
                 |
         +-------+-------+
         |               |
         v               v
    Text Analysis    URL Analysis
         |               |
         +-------+-------+
                 |
                 v
      External Threat Intelligence
                 |
                 v
        Evidence Aggregation
                 |
                 v
          Risk Assessment
                 |
         +-------+-------+
         |               |
         v               v
     Risk Verdict   Analysis Status
         |               |
         +-------+-------+
                 |
                 v
          Structured Result
```

**Text analysis** looks for patterns tied to urgency and pressure tactics, account-suspension threats, verification/credential requests, delivery scams, and investment schemes — each one contributes evidence rather than deciding the result by itself.

**URL analysis** checks domain structure, typosquatting and brand-impersonation indicators, suspicious TLDs, and redirect behavior — redirect destinations get checked before the app follows them, with a hop limit, so it doesn't end up making unsafe requests on a user's behalf.

**External threat intelligence** supplements the local checks against Google Safe Browsing, AbuseIPDB, and RDAP for domain registration data. A couple of additional providers are wired in for development use only. None of these are load-bearing on their own — if one is down, the app notices and says so (more on that below) instead of quietly pretending nothing's wrong.

**Evidence aggregation** is the part I spent the most time getting right. Different checks can fire on the same message for different reasons, and treating every signal as equally authoritative produces a lot of false positives. So evidence carries weight — some of it's weak on its own and needs support, some of it is strong enough to move the score a lot by itself.

## The idea I'm most proud of: separating risk from confidence

Early on I kept running into the same problem: what should the app say when Google Safe Browsing times out, or AbuseIPDB is rate-limited? The lazy answer is to just skip that check and still mark the message "SAFE" if nothing else fired — but that's quietly lying about how thorough the check actually was.

So the app tracks **analysis completeness** as its own thing, separate from the risk score. If local text/URL analysis still found something suspicious, that evidence isn't thrown away just because one external provider didn't respond — and if the overall check was limited, the result says so plainly instead of implying a full, clean bill of health.

## A lesson the evaluation corpus taught me

I started with a small, curated set of obvious scam examples, and the detector handled them well — which told me almost nothing. Once I expanded the test corpus to a wider, messier set of real scam and legitimate messages, previously-hidden false negatives showed up immediately. Patterns I thought were covered weren't; some legitimate messages I assumed were safe tripped signals I hadn't considered.

That's the actual lesson: passing a small hand-picked test set doesn't tell you much about real-world performance. The evaluation process now leans on precision/recall across categories and treats every missed case as a prompt to go fix the underlying signal, not just patch the one example.

## Architecture, at a glance

- **API layer** — receives and validates requests (message length, URL count, URL length, request size are all capped before anything reaches the detection logic)
- **Orchestration** — coordinates which detection components run
- **Text + URL detectors** — independent, individually testable
- **Threat intelligence integrations** — isolated from the core, so a provider outage can't take down the whole pipeline
- **Evidence aggregation** — combines everything into one assessment
- **Result generation** — risk score, verdict, human-readable reasons, completeness status

One backend, three client surfaces (web app, Chrome extension, and an earlier Android/TWA client) — all talking to the same engine through the same API, so detection logic can't quietly drift between them.

*(I've kept the actual evidence weighting and scoring internals out of this writeup on purpose — that's the part of the product I'd rather not hand to anyone for free. Glad to walk through it live.)*

## Tech stack

| | |
|---|---|
| Python | core detection logic and backend |
| Flask | API framework |
| Gunicorn | production WSGI server |
| tldextract | domain parsing |
| dnspython | DNS lookups |
| python-dotenv | environment/config |

## Privacy, briefly

Raw messages aren't stored. Logs are written to avoid raw content. External providers only get what they need for their specific check (a URL to a URL-reputation service, not the whole message). Request tracing works off a request ID, not message content.

## Where it still falls short

A SAFE verdict isn't a guarantee — it means nothing currently implemented fired, not that nothing's wrong. New scam patterns show up before the detector's been taught to catch them. The risk score is a ranking signal, not a calibrated probability — a 70 doesn't mean "70% chance this is fraud." And it's only as good as the providers it depends on, which have their own coverage gaps.

I'd rather say that plainly here than have someone find it out the hard way.

## What's next

Mostly: feeding more real (and more varied) data through the evaluation corpus and fixing what it finds. I'm also curious whether a lightweight ML layer on top of the rule engine would help with borderline cases, but I haven't found a case for it yet that the rule-based approach doesn't already handle reasonably well.

---

**Want to see the actual source?** It's private while this is a live product I'm still building on. If you're evaluating this as part of a hiring process, reach out and I'll grant access or walk through it on a call.
