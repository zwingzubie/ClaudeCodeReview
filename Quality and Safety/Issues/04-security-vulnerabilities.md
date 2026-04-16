## Security Vulnerability Accumulation: The Threat Surface AI Is Building

**Related to:** [Issues Overview](overview.md) — Issue 4 · [Security: Threat Modeling](../Security/01-threat-modeling.md)[^a] · [Security: SAST and DAST Integration](../Security/02-sast-dast-integration.md)[^b] · [Ethics: Security Responsibility](../Ethics/03-security-responsibility.md)[^c] · [Metrics: Security Vulnerability Trends](../Metrics/04-security-vulnerability-trends.md)[^d]

---

### The Structural Security Problem

AI coding tools generate security vulnerabilities at a measurably higher rate than human developers. This is not a temporary limitation of current models — it is a structural characteristic of how LLMs generate code. AI tools infer patterns statistically from training data; they do not reason about whether those patterns are secure in the context of a specific system, a specific threat model, or a specific data type. The result is code that looks correct and passes tests but contains security flaws that a comprehending engineer would have caught.

For a team without a dedicated security engineer, this is a compounding risk. Vulnerabilities introduced by AI tools fall diffusely across PRs that no single person is responsible for reviewing with a security lens. By the time they are discovered — in an audit, a penetration test, or an incident — they have been in production for months and may have propagated across multiple dependent modules.

---

### The Numbers

The scope of the problem is documented across multiple independent analyses from 2025 and 2026.

**Veracode's 2025 GenAI Code Security Report** tested over 100 LLMs across Java, JavaScript, Python, and C# using 80 standardized tasks mapped directly to OWASP Top 10 Common Weakness Enumerations. **45% of all AI-generated code samples failed security tests.** Java had the worst failure rate at 72%. LLMs failed to prevent Cross-Site Scripting (CWE-80) in 86% of cases and Log Injection (CWE-117) in 88% of cases. Critically, models improved at syntactic correctness over years of evaluation — but showed no improvement in security.[^3]

**Apiiro's empirical analysis** of tens of thousands of repositories across Fortune 50 enterprises found that AI-assisted developers produce 3–4x more commits but introduce **10x more security findings**. By June 2025, AI code was adding over 10,000 new security findings per month — a 10x spike from December 2024. Privilege escalation paths increased 322%. Azure credential exposure nearly doubled. Syntax errors dropped 76% — masking the severity of the underlying security regression.[^2]

**AppSec Santa's 2026 study** tested GPT-5.2, Claude Opus 4.6, Gemini 2.5 Pro, DeepSeek V3, Llama 4 Maverick, and Grok 4 against the OWASP Top 10:2021 standard using five open-source SAST tools. **25.1% of generated code contained confirmed vulnerabilities.** Server-Side Request Forgery (SSRF) was the most common single finding at 32 confirmed instances. Critically, **78% of confirmed vulnerabilities were detected by only one of the five SAST tools** — no single scanner provides comprehensive coverage of AI-generated code.[^3]

**DryRun Security's Agentic Coding Security Report** (March 2026) built two real applications with Claude Code, OpenAI Codex, and Gemini — from MVP through production. **87% of PRs contained at least one vulnerability**. 143 security issues were found across 38 scans. No agent produced a fully secure application. Systemic authentication failures persisted in every final codebase.[^11]

---

### The Hardcoded Credentials Epidemic

One of the most concrete manifestations of AI security vulnerability is the growth of hardcoded credentials in committed code. GitGuardian's **State of Secrets Sprawl 2026** report found that 28.65 million new hardcoded secrets appeared in public GitHub in 2025 — a **34% year-over-year increase**, the largest single-year jump on record.[^7]

AI-assisted commits leak secrets at **3.2% vs. a 1.5% baseline** — more than double the human-written rate. AI service credentials (LLM API keys, vector database tokens) reached 1.28 million on public GitHub, up 81% year-over-year. Eight of the ten fastest-growing secret detector categories were AI service credentials — meaning the very tools being adopted to build securely are themselves introducing secret sprawl.

The Cloud Security Alliance's March 2026 research note on credential sprawl documents an additional novel attack surface: 24,008 unique secrets were found hardcoded in MCP (Model Context Protocol) configuration files, with 2,117 confirmed valid and live.[^6]

---

### The Hallucinated Package Problem

A vulnerability class unique to AI-generated code is the hallucinated dependency — a package name that does not exist but sounds plausible, which an attacker can register and weaponize. Researchers term this "slopsquatting."

A peer-reviewed study accepted at USENIX Security 2025 analyzed 576,000 code samples across 16 LLMs and found that open-source models hallucinate package names at a **21.7% rate**; commercial models at **5.2%**.[^1] The 205,474 unique hallucinated package names identified represent a persistent, systemic attack vector. Any team that does not audit AI-suggested dependencies before installing them is building supply chain vulnerabilities into their project.

---

### The CVE Record

Through March 2026, Georgia Tech's Vibe Security Radar project documented **74 confirmed CVEs traceable to AI tools** — 35 in March 2026 alone, up from 6 in January and 15 in February. Researchers estimate the true figure is **5–10x higher** due to attribution gaps. The Cloud Security Alliance estimates approximately 80% of developers believe AI produces more secure code than humans — a belief that directly contradicts the empirical record.[^5]

Fortune's December 2025 reporting documented the first confirmed real-world exploitation of AI coding tools as a distinct attack surface. Amazon Q's VS Code extension was compromised via prompt injection — a malicious version directing the tool to wipe local files passed Amazon's verification and stayed live for two days. Critical prompt-injection vulnerabilities were independently discovered in Cursor, GitHub Copilot, and Google Gemini CLI in 2025.[^12]

---

### Why Teams Without Security Engineers Are Most at Risk

Escape.tech's active DAST scan of 5,600 publicly available applications built on vibe coding platforms found **2,000+ vulnerabilities, 400+ exposed secrets, and 175 instances of PII** including medical records, IBANs, and phone numbers. Over 10% of applications had critical row-level security disabled by default. The researchers note these are conservative estimates from passive scanning — aggressive testing would yield more findings.[^10]

The common thread: these applications were built by teams and developers who had no dedicated security function. Responsibility for security was diffuse or absent. AI tools produced functional applications; nobody was positioned to evaluate whether those applications were secure.

IT Pro's January 2026 survey found that 17% of organizations have no control policies over AI use in development, and 98% of security professionals say their teams "lack adequate visibility into how GenAI is used in development workflows."[^13]

---

### The OWASP LLM Framework

OWASP's 2025 Top 10 for LLM Applications — the authoritative enumeration of LLM-specific security risks — covers 10 categories not present in standard OWASP web guidance: Prompt Injection (LLM01), Sensitive Information Disclosure (LLM02), Supply Chain (LLM03), Improper Output Handling (LLM06), Excessive Agency (LLM08), and others including Vector/Embedding Weaknesses and System Prompt Leakage.[^9] These risks "do not map cleanly to legacy web categories" and require a distinct security review process.

The OpenSSF's August 2025 security guidance for AI code assistants establishes that developers remain "legally and ethically responsible for all AI-generated code" and cannot defer to AI as a security shortcut. It mandates alignment with OWASP Top 10, ASVS, SAFECode, and CWE/SANS and explicitly addresses the 19.7% hallucination rate for non-existent packages.[^8]

---

### Proposed Solutions

**1. Automated SAST in CI/CD — Before Review, Not After**

Every PR containing AI-generated code should pass automated static analysis before it reaches a human reviewer. The AppSec Santa study found that 78% of confirmed vulnerabilities in AI-generated code were caught by only one SAST tool — meaning no single scanner provides comprehensive coverage.[^3] Running multiple complementary tools (Semgrep, Snyk Code, CodeQL) is required to approximate full coverage, particularly for the injection flaws, hardcoded credentials, and SSRF patterns most common in AI-generated code.

**2. Dependency Auditing for AI-Suggested Packages**

Every dependency suggested by AI tooling must be verified before installation: does the package exist? Is it the legitimate version? Does it match the AI's stated description? The USENIX Security study found 5.2% hallucination rates even in commercial models — on a team generating dozens of AI-assisted features per month, this is a meaningful exposure.[^1] Add a CI step that audits new dependencies against a known-good registry.

**3. Security Review Rotation for AI-Generated Code**

Assign backend engineers ownership of a security review rotation — one engineer per sprint specifically reviews AI-generated authentication, data access, and external API code against the OWASP LLM Top 10. The DryRun study found that 87% of AI-generated PRs contained at least one vulnerability.[^11] Without a named reviewer with a security lens, these will not be caught in standard implementation review.

**4. Prompt the AI to Audit Itself**

Before finalizing any security-sensitive AI-generated feature, prompt Claude Code explicitly: "What are the potential security vulnerabilities in this implementation? What would an attacker target? How would you rewrite the most vulnerable parts?" The Cloud Security Alliance research confirms this as an effective practice when used consistently — AI tools are capable of identifying vulnerabilities in their own output when directly prompted to do so.[^6]

**5. Secrets Scanning at Commit Time**

Configure pre-commit hooks and CI to scan for hardcoded secrets before code is pushed. GitGuardian's data shows AI-assisted commits leak secrets at 2x the human rate.[^7] This is a low-effort, high-value control that prevents the most common credential exposure pattern in AI-generated code.

---

### Summary

AI-generated code introduces security vulnerabilities at structurally higher rates than human-written code, across all major model families and all major language ecosystems. The vulnerability types are not exotic — they are the OWASP Top 10 classes that every engineering team has been taught to prevent for two decades. The difference is that AI generates them faster, in larger volumes, and in ways that pass code review by teams who are not explicitly looking for them. On a team without a dedicated security engineer, this is a governance problem as much as a technical one. The solution is not to distrust AI entirely but to build the structural controls — SAST, secrets scanning, dependency auditing, security rotation — that ensure AI-generated code receives the security scrutiny that its vulnerability rate demands.

---

[^1]: Joseph Spracklen et al. — "We Have a Package for You! A Comprehensive Analysis of Package Hallucinations by Code Generating LLMs," arXiv:2406.10279, USENIX Security 2025. https://arxiv.org/abs/2406.10279
    Analysis of 576,000 code samples across 16 LLMs. Open-source models hallucinate package names at 21.7%; commercial models at 5.2%. 205,474 unique hallucinated package names — a persistent supply chain attack vector ("slopsquatting").

[^2]: Itay Nussbaum (Apiiro) — "4x Velocity, 10x Vulnerabilities: AI Coding Assistants Are Shipping More Risks," September 4, 2025. https://apiiro.com/blog/4x-velocity-10x-vulnerabilities-ai-coding-assistants-are-shipping-more-risks/
    Empirical analysis across Fortune 50 enterprises. AI code: 10x more security findings, 322% more privilege escalation paths. AI adds 10,000+ security findings per month by June 2025 — a 10x spike from December 2024.

[^3]: Veracode Research — "2025 GenAI Code Security Report," July 30, 2025. https://www.veracode.com/blog/genai-code-security-report/
    100+ LLMs tested across 80 OWASP-mapped tasks. 45% of AI-generated code failed security tests. Java: 72% failure rate. XSS prevented in only 14% of cases; Log Injection in only 12%. No improvement in security despite years of model capability improvements.


[^5]: Cloud Security Alliance — "Vibe Coding's Security Debt: The AI-Generated CVE Surge," April 4, 2026. https://labs.cloudsecurityalliance.org/research/csa-research-note-ai-generated-code-vulnerability-surge-2026/
    74 confirmed CVEs traceable to AI tools through March 2026, with 35 in March alone. True figure estimated 5–10x higher. ~80% of developers believe AI produces more secure code — contradicted by all empirical evidence.

[^6]: Cloud Security Alliance — "Vibe Coding Security Crisis: Credential Sprawl and SDLC Debt," March 31, 2026. https://labs.cloudsecurityalliance.org/research/csa-research-note-ai-generated-code-security-vibe-coding-202/
    34% YoY increase in hardcoded credentials on public GitHub in 2025. AI service credential leaks +81% YoY. Documents "Rules File Backdoor" and "slopsquatting" as novel enumerated attack surfaces. 24,008 unique secrets in MCP configuration files.

[^7]: Anna Nabiullina and Carole Winqwist (GitGuardian) — "The State of Secrets Sprawl 2026," March 17, 2026. https://blog.gitguardian.com/the-state-of-secrets-sprawl-2026/
    28.65 million new hardcoded secrets on public GitHub in 2025 (34% YoY). Claude Code-assisted commits show 3.2% secret-leak rate vs. 1.5% human baseline. AI service credentials: 1.28 million, +81% YoY.

[^8]: OpenSSF (Avishay Balter, David A. Wheeler et al.) — "Security-Focused Guide for AI Code Assistant Instructions," August 1, 2025. https://best.openssf.org/Security-Focused-Guide-for-AI-Code-Assistant-Instructions
    Industry-standard guidance: AI models "often perpetuate insecure patterns." 19.7% of AI-suggested packages do not exist. Developers remain legally and ethically responsible for all AI-generated code.

[^9]: OWASP GenAI Security Project — "OWASP Top 10 for LLM Applications 2025." https://genai.owasp.org/resource/owasp-top-10-for-llm-applications-2025/
    Authoritative enumeration of LLM-specific risks including Prompt Injection (LLM01), Supply Chain (LLM03), Excessive Agency (LLM08). Introduces Vector/Embedding Weaknesses and System Prompt Leakage. Risks "do not map cleanly to legacy web categories."

[^10]: Nohé Hinniger-Foray et al. (Escape.tech) — "Methodology: How We Discovered Over 2k High-Impact Vulnerabilities in Apps Built with Vibe Coding Platforms," October 29, 2025. https://escape.tech/blog/methodology-how-we-discovered-vulnerabilities-apps-built-with-vibe-coding/
    DAST scan of 5,600 publicly available apps built on Lovable, Bolt, and similar platforms. 2,000+ vulnerabilities, 400+ secrets, 175 PII exposures. 10%+ of apps had critical row-level security disabled by default.

[^11]: DryRun Security — "New Research: Claude Generates the Most Unresolved Security Flaws in AI-Built Applications," March 11, 2026. https://www.globenewswire.com/news-release/2026/03/11/3253696/0/en/New-DryRun-Security-Research-Anthropic-s-Claude-Generates-the-Most-Unresolved-Security-Flaws-in-AI-Built-Applications.html
    Three AI agents building two real applications from MVP to production. 87% of PRs contained at least one vulnerability; 143 security issues across 38 scans. No agent produced a fully secure application. Authentication failures persisted in every final codebase.

[^12]: Sage Lazzaro — "AI Coding Tools Exploded in 2025. The First Security Exploits Show What Could Go Wrong," *Fortune*, December 15, 2025. https://fortune.com/2025/12/15/ai-coding-tools-security-exploit-software/
    Documents first confirmed real-world exploitations of AI coding tools. Amazon Q extension compromised via prompt injection; stayed live for two days. Prompt-injection vulnerabilities confirmed in Cursor, Copilot, and Gemini CLI.

[^13]: Nicole Kobie — "Nearly Half of Software Developers Don't Check AI-Generated Code," *IT Pro*, January 9, 2026. https://www.itpro.com/software/development/software-developers-not-checking-ai-generated-code-verification-debt
    Nearly 50% of developers do not verify AI code before committing. 17% of organizations have no AI use control policies. 98% of security professionals say their teams lack visibility into how GenAI is used in development.

[^a]: [Security: Threat Modeling](../Security/01-threat-modeling.md) — threat modeling provides the framework for identifying which vulnerability classes AI-generated code is most likely to introduce; these are the risks this document describes.

[^b]: [Security: SAST and DAST Integration](../Security/02-sast-dast-integration.md) — SAST/DAST integration is the primary detection mechanism for the vulnerabilities described here; automated scanning in the pipeline is the operational countermeasure.

[^c]: [Ethics: Security Responsibility](../Ethics/03-security-responsibility.md) — security responsibility documents the accountability structure when AI-generated vulnerabilities reach production; this document describes what accumulates, that one describes who owns it.

[^d]: [Metrics: Security Vulnerability Trends](../Metrics/04-security-vulnerability-trends.md) — vulnerability trend metrics operationalize the risk described here; tracking trends over time converts a qualitative concern into a measurable governance signal.
