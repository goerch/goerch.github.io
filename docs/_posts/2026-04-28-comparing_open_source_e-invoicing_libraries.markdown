---
layout: post
title:  "Measuring and Comparing Open Source E-Invoicing Libraries: A Cautious Look"
date:   2026-04-28 7:12:00 +0200
categories: e-invoicing open-source code-quality standards
---
*A developer's notes on code metrics, ZUGFeRD, XRechnung, and the libraries that implement them.*

---

I recently spent some time trying to get a sense of two open source e-invoicing libraries —
[mustangproject](https://github.com/ZUGFeRD/mustangproject) (Java) and
[ZUGFeRD-csharp](https://github.com/stephanstapel/ZUGFeRD-csharp) (.NET) — and came away with
more questions than answers, but also some useful perspective. This post summarises what I found.
I am not a domain expert in e-invoicing standards, so please treat the observations here with
appropriate caution and verify anything that matters to your situation.

---

## Measuring Code Size: scc and the COCOMO Trap

A natural starting point is asking "how big is this project?" A useful free tool for this is
[scc](https://github.com/boyter/scc) (Sloc, Cloc and Code), a fast Go-based line counter that
also computes cyclomatic complexity and COCOMO cost estimates per language. It is in many ways
the spiritual successor to David A. Wheeler's `sloccount`.

Running `scc` on both libraries reveals an immediate problem: the raw totals are dominated by
XML, XSD, and XSLT files, not handwritten application logic. For mustangproject the Extensible
Stylesheet (XSL) category alone accounts for roughly 688,000 lines; for ZUGFeRD-csharp the XML
category runs to around 753,000 lines. Both of these are largely **normative artefacts from
standards bodies** — XSLT stylesheets and Schematron rules from
[KoSIT](https://xeinkauf.de/xrechnung/) and [FeRD](https://www.ferd-net.de/), not original code.

When you filter to just the handwritten logic the picture becomes more comparable:

| Library | Language | Code lines | Cyclomatic complexity |
|---|---|---|---|
| mustangproject | Java | ~22,000 | ~3,586 |
| ZUGFeRD-csharp | C# | ~21,500 | ~1,972 |

Two libraries implementing roughly equivalent functionality, at almost identical size — but the
Java is nearly twice as complex per line. Whether that reflects Java's more verbose OOP patterns
or accumulated conditional handling over a longer project lifetime is hard to say without deeper
analysis.

### On COCOMO estimates

`scc` produces COCOMO "organic" cost and schedule estimates. For mustangproject it produced a
figure in the tens of millions of dollars. This is not meaningful here. The COCOMO model was
calibrated on large 1980s-era institutional projects with significant process overhead. It has no
concept of "this file was transcribed from a normative standards document." The relative
complexity ratios within COCOMO (more branching → more effort) are still roughly valid as
*trends*, but the absolute dollar figures should be ignored unless you are specifically modelling
labour costs in a comparable institutional context.

The honest conclusion is that `scc` is useful for getting a rough sense of *where the mass of a
project lives* — how much is handwritten logic versus bundled artefacts, which language carries
the complexity — but effort estimation from static metrics alone is essentially hopeless for
projects of this kind. The moment a codebase bundles normative standards documents, the numbers
stop reflecting developer effort in any meaningful way.

---

## The KoSIT Test Suite: A Genuine Quality Signal

A more useful quality indicator than line counts is test coverage against normative reference
cases. For XRechnung, KoSIT maintains an
[official test suite](https://github.com/itplr-kosit/xrechnung-testsuite) of real-world business
case invoices and technical edge cases, versioned alongside each XRechnung release. This is a
government-mandated artefact — KoSIT operates on behalf of the
[IT-Planungsrat](https://www.it-planungsrat.de/), the joint federal-state IT standards body.

The ~1,200 XML files found in the ZUGFeRD-csharp repository are almost certainly these KoSIT
test instances, bundled as integration tests. That is a good sign — it means the library is
testing its output against the normative reference. However, it also means those files should
be entirely excluded from any LOC or effort analysis.

**For ZUGFeRD there is no equivalent official test suite.** The closest thing is the
[ZUGFeRD/corpus](https://github.com/ZUGFeRD/corpus) repository, which is maintained by the
Mustang team rather than FeRD, and is explicitly described in its own README as unofficial and
makeshift. This asymmetry reflects the different governance structures: KoSIT has a legal mandate
and publishes rigorous versioned artefacts; FeRD is an industry forum with no equivalent
obligation.

---

## ZUGFeRD and XRechnung: Not Synonyms

It is easy to treat these two terms as interchangeable, but they are architecturally quite
different things built on the same foundation.

Both are implementations of the European standard **EN 16931**, which defines a semantic invoice
data model. On top of that:

- **XRechnung** is a pure XML file — a German CIUS (Core Invoice Usage Specification) that
  tightens EN 16931 for public sector use, notably requiring a
  [Leitweg-ID](https://www.xeinkauf.de/xrechnung/faq/) for routing to the correct government
  inbox. It has no visual component. It is governed by
  [KoSIT on behalf of the IT-Planungsrat](https://xeinkauf.de/xrechnung/).

- **ZUGFeRD** is a hybrid format: a PDF/A-3 file with an invoice XML embedded as an attachment.
  A human can read it in any PDF viewer; the embedded XML can also be extracted and processed
  automatically. It was designed primarily for B2B use. It is governed by
  [FeRD (Forum elektronische Rechnung Deutschland)](https://www.ferd-net.de/).

They overlap because ZUGFeRD 2.1.1 introduced a dedicated XRechnung reference profile, meaning
a ZUGFeRD envelope can carry XRechnung-compliant XML. But they remain distinct standards with
distinct governance bodies.

### The legal timeline

The obligations created by these standards are real and time-bound:

- **B2G (suppliers to federal government):** mandatory since
  [27 November 2020](https://www.e-rechnung-bund.de/faq/xrechnung/) at federal level; most
  Bundesländer have since followed.
- **B2B receiving:** mandatory since [1 January 2025](https://www.bundesfinanzministerium.de/)
  under §14 UStG (Wachstumschancengesetz) — every German B2B company must be able to process
  EN 16931-compliant e-invoices.
- **B2B sending:** mandatory from [1 January 2027](https://www.bundesfinanzministerium.de/)
  (with transitional rules).

There is **no KoSIT certification scheme for software vendors**. Conformance is defined
operationally: if your output passes the KoSIT validator, it is conformant. Invoices that fail
the validator are automatically rejected by government portals, which is a stronger forcing
function than any formal certification could be.

---

## The Libraries: Mustangproject vs ZUGFeRD-csharp

### mustangproject

[mustangproject](https://github.com/ZUGFeRD/mustangproject) is a Java library, Maven artifact,
command-line tool, and REST server. It has been in development since the early days of ZUGFeRD
and has accumulated broad format support: ZUGFeRD 1 and 2.x, Factur-X, and XRechnung (CII and
UBL syntaxes).

A key architectural decision is that **validation is built in**. The library bundles the KoSIT
XRechnung Schematron rules directly (as listed in its
[NOTICE file](https://github.com/ZUGFeRD/mustangproject/blob/master/NOTICE)), as well as EN
16931 validation artefacts from the CEN. This makes mustangproject the de facto reference
implementation for validation in the German e-invoicing open source ecosystem.

The project's popularity in the Java ecosystem is partly structural: Java has a long history in
German ERP and accounting software (SAP, Lexware, Sage), so a library with a clean Maven
dependency fits naturally into existing stacks. The command-line tool also broadened the audience
beyond Java developers.

One caveat worth noting: an
[open bug](https://github.com/ZUGFeRD/mustangproject/issues/661) shows a case where
mustangproject rejects XML that the KoSIT validator accepts — meaning it applies stricter rules
than the official reference in some edge cases. This may or may not be a problem depending on
your use case.

### ZUGFeRD-csharp

[ZUGFeRD-csharp](https://github.com/stephanstapel/ZUGFeRD-csharp) started in 2013 as a hobby
project and grew to be used by hundreds of companies. It remains the primary .NET option for
this domain.

However, it has recently undergone a significant licensing change.
[Starting with version 18.0.0](https://github.com/stephanstapel/ZUGFeRD-csharp/releases), the
project moves to an open-core model: the base library remains open source for stability and bug
fixes, but new features and — notably — **validation** are now part of the commercial
[FactoorSharp](https://www.factoorsharp.de/) product.

The commercial validation component is advertised as using Mustang, Valitool, and VeraPDF as
backends. This is worth noting for two reasons:

1. It means the .NET ecosystem's primary validation path delegates to the Java library — Mustang
   again acting as the ground truth.
2. From a purely commercial perspective, advertising that your paid feature calls a free
   competitor may not be the strongest value proposition, though the integration work and
   convenience still has value.

For teams currently using ZUGFeRD-csharp, version 18 remains available as open source, but
future XRechnung version support and new features will likely appear first or only in the
commercial tier. This is a risk worth factoring into dependency decisions.

---

## Why .NET Validation Delegates to Java: The XSLT 2.0 Gap

The decision to call Mustang from the commercial .NET validation tier is not arbitrary — it
reflects a structural platform limitation that is worth understanding.

The KoSIT Schematron validation rules are compiled to **XSLT 2.0**. The built-in .NET XML stack
(`XslCompiledTransform`) only supports XSLT 1.0. As one developer put it plainly in the
ZUGFeRD-csharp issue tracker, `document-uri()` — an XSLT 2.0 function used in the KoSIT
stylesheets — is simply not recognised, making native .NET validation impossible without an
external processor.

The natural solution is [Saxon-HE](https://github.com/Saxonica/Saxon-HE), the open source XSLT
3.0 processor from Saxonica, available as a NuGet package. However, here the platform
fragmentation becomes painful:

- **Saxon-HE for .NET Framework** works and has been demonstrated in community contributions
  to ZUGFeRD-csharp, but ZUGFeRD-csharp targets **.NET Standard**, not .NET Framework.
- **SaxonCS** — the modern .NET Standard/Core-compatible version — is a **commercial product**,
  not open source, and was described in the ZUGFeRD-csharp community as unaffordably expensive
  for an open source library dependency.

This is corroborated by our own experiments, which reached the same conclusion independently.

The net result is a somewhat uncomfortable situation for the .NET e-invoicing ecosystem: the
only practical path to free, standards-compliant XRechnung validation on modern .NET is to
shell out to a Java process running Mustang — which is exactly what the commercial FactoorSharp
tier does. There is also
[concern in the community](https://github.com/stephanstapel/ZUGFeRD-csharp/discussions/316)
that Saxonica's near-monopoly on XSLT 2.0/3.0 processing creates a long-term open source risk,
since the Java ecosystem's Saxon-HE (also MPL-2.0) could in principle face the same commercial
pressure at any time.

For Java, this problem does not arise: Saxon-HE on the JVM is fully open source, supports XSLT
3.0, and integrates cleanly — which is one more structural reason why mustangproject can offer
validation as a first-class feature while ZUGFeRD-csharp cannot, at least not without
commercial dependencies.

---

## Summary

| Topic | Observation |
|---|---|
| scc / LOC | Useful for locating where project mass lives; useless for effort estimation when standards artefacts dominate |
| COCOMO estimates | Not meaningful here; treat as noise |
| KoSIT test suite | Normative, versioned, government-backed — exists for XRechnung only |
| ZUGFeRD test suite | No official equivalent; community corpus exists but is unofficial |
| mustangproject | Java, fully open source, validation built in via KoSIT Schematron |
| ZUGFeRD-csharp | .NET, moving to open-core from v18; validation in commercial tier |
| XSLT 2.0 on .NET | Built-in stack is XSLT 1.0 only; Saxon-HE requires .NET Framework; SaxonCS is commercial — structural barrier to free validation on modern .NET |
| XRechnung vs ZUGFeRD | Different formats, different governance; XRechnung is legally mandatory for B2G |
| B2B obligation | Receiving mandatory since Jan 2025; sending from Jan 2027 |
| "Certified by KoSIT" | No such scheme exists; passing the KoSIT validator is the conformance test |

---

*Corrections and additions welcome. This post reflects a learning process rather than expert
knowledge, and the e-invoicing standards landscape changes at least once a year with each KoSIT
release cycle.*