# Failures in AI-assisted development: a working record

Fahad Mohammed Gibrine

I build and operate a hotspot billing and RADIUS platform that runs in production and
serves paying customers. Most of the code was written with model assistance, and every
change referenced here carries a `Co-Authored-By` trailer naming the model. This file
records the cases where the generated work was wrong, what made it look right, and how
it was caught.

Entries are drawn from commit history in the project's own repository, which is
private because it describes live infrastructure. Hashes and the underlying diffs can
be shown on request. Hostnames, addresses and anything describing the current state of
the deployment are left out here on purpose.

---

## 1. Captive portal bypass in a generated installer script

Security impact: high.

**What I asked for.** A generator that emits a RouterOS script to configure a
customer's router for hotspot billing.

**What came back.** A script that bound the hotspot to the wireless interface only. It
read as complete and used the right command vocabulary.

**What was wrong.** Every Ethernet port was left on the WAN side with open internet.
Any customer connecting through a downstream access point never passed through the
captive portal, so they got service without authenticating or paying. The script was
not broken. It worked exactly as written, and what was written was the wrong network
topology.

**Why it looked right.** Syntactically valid output, and the hotspot genuinely came up
on the wireless interface. On a bench test with a laptop on Wi-Fi it passed. The
failure only appears with a downstream AP on an Ethernet port, which is the normal
deployment.

**How I caught it.** Testing the topology customers actually use rather than the one
in front of me.

**What I changed.** Rewrote the generator to put WAN on its own port, bridge the
remaining Ethernet ports with the wireless interface, and attach the hotspot to the
bridge. I also cut the generator's configurable fields from a long list to three,
because the subnet, pool and DNS name were options that existed only to be set wrongly.

**Lesson.** A model will produce a configuration that satisfies the words of the
request while silently assuming a topology nobody stated. The assumption is the
dangerous part and it is invisible in the output. Ask what the generated config assumes
about everything it does not touch.

---

## 2. Credentials hardcoded into client-side source and committed

Security impact: high.

**What was wrong.** A RADIUS shared secret and a server address were set as default
values on an admin form. That put them in the repository and served them to every
browser that loaded the admin page. The same credential was also printed in two setup
guides.

**Why it looked right.** Prefilled defaults are a reasonable usability choice, and each
file in isolation looked like sensible work. Nothing was malformed. The problem was a
property of where the value ended up, which no single file shows you.

**How I caught it.** Reading the committed output as an attacker would rather than as
the author, asking what an unauthenticated visitor to the admin page receives.

**What I changed.** Fields are empty and the operator enters the value once, held in
that browser's local storage. Guides use placeholders. I verified the generated script
was byte-for-byte unchanged by hashing both versions' output on identical inputs, so
the security fix could not have altered behaviour. Remediation of the exposed value
and of related storage decisions is tracked in the project's own security
documentation.

**Lesson.** Secret-in-source is the most common security defect I have hit in
model-assisted work, and it is never flagged, because the model is optimising for a
form that works on first try.

---

## 3. A confident diagnosis that was wrong, and a change that would have taken a third party offline

Availability impact: high, for a third party.

**What was wrong.** An earlier change deleted the reverse-proxy server blocks for a
partner company's sites, on the stated reasoning that this platform does not host other
parties' sites. It does. Deploying that change would have taken the partner's portal
offline.

**The second error.** The same work claimed those blocks had already taken the whole
portal down. They had not. Production was up throughout, because the containers those
blocks referenced do exist on the production host, so the proxy resolved them there
without trouble. The defect was real but latent: it breaks any host that lacks those
containers, meaning every fresh deployment, laptop and CI run. So the diagnosis named a
real problem and got both its mechanism and its blast radius wrong.

**Why it looked right.** The reasoning was coherent and the conclusion followed from
its premise. The premise, that this server hosts only its own sites, was never checked
against DNS or the running host.

**How I caught it.** Resolving the hostname and looking at what was actually running in
production, instead of accepting a plausible account of it.

**What I changed.** Restored both sites in their own file, installed only where the
partner's certificate is present, with upstreams behind variables so a stopped
container returns 502 for that one site rather than preventing the proxy from starting.
Verified three ways: certificate absent, certificate present with the containers
absent, and the live local stack.

**Lesson.** The failure mode that costs most is not bad code, it is a confident causal
story about production that nobody verified against production. Fluent reasoning from
an unchecked premise is harder to catch than a syntax error, because there is nothing
to catch until it ships.

---

## 4. Config that only starts when the environment is wrong

Availability impact: medium.

**What was wrong.** A TLS config template proxied to containers that do not exist in
this project, behind certificates never issued for it. The proxy resolves targets at
config load, so an unresolvable target is fatal. The stack came up only as long as no
certificate existed, and broke the moment one did.

**Lesson.** Generated infrastructure config is tested against the state the machine
happens to be in. Ask which state the config has never been run in, because that is
where it fails, and here that state was "working correctly".

---

## 5. Escaping bug that made a repair tool do nothing, silently

Impact: low, but a clean example of silent failure.

**What was wrong.** One function joined its output lines with a literal `\n` rather
than a newline, while the function directly beside it did the right thing. The result
was a single 1931-character line with literal backslash-n between commands. The target
device parsed it as invalid syntax and the repair silently did nothing. The
neighbouring button was fine, which is why nobody noticed.

**Lesson.** Two functions written in the same session, side by side, one correct and
one not. Consistency within generated code is not a safe assumption, so near-identical
code needs checking separately rather than by inspection of its twin.

---

## 6. Line endings: correct on the authoring machine, unrunnable on the target

Impact: low, deploy blocker.

**What was wrong.** With `core.autocrlf=true`, the Windows default, `git archive`
rewrote the shell scripts to CRLF on export. The shipped deploy script arrived with a
`#!/bin/bash\r` shebang and the server reported "cannot execute: required file not
found", an error that says nothing about line endings.

**What I changed.** Pinned `core.autocrlf=false` for the archive so the export is LF
regardless of the machine's git config.

**Lesson.** The code was correct and the toolchain broke it in transit. Cross-platform
assumptions are a standing blind spot in generated build and deploy scripts, and the
resulting error points nowhere near the cause.

---

## Patterns so far

Six entries is not a sample worth generalising from, but the distribution is already
lopsided in a way I did not expect.

What I expected to find was broken code. Almost none of these are that. Entries 1, 3
and 4 are code that runs correctly and encodes a wrong assumption about the
environment: a topology nobody specified, a server believed to host only its own sites,
a certificate believed never to exist. Entry 2 is correct code putting a secret
somewhere it should not be. Only entries 5 and 6 are defects in the ordinary sense.

Two things follow for how I work.

Nothing generated goes out without being tested in the configuration it will actually
meet, not the one on my bench. Entries 1, 3 and 4 were all invisible to local testing
and all visible immediately to a test that matched production.

A causal explanation gets checked harder than code does. Code that is wrong usually
announces itself eventually. A wrong explanation of why something broke, delivered
fluently, propagates into the next three decisions before anything contradicts it.
Entry 3 is the one that would have cost a third party real downtime, and it was a
reasoning failure, not a coding one.
