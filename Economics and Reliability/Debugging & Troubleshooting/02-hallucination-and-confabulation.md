## Hallucination and Confabulation in AI Output

**Related to:** [Debugging & Troubleshooting Overview](00-overview.md) — Debugging Area 2 · [QA & Testing: AI-Generated Test Coverage](../QA%20&%20Testing/02-ai-generated-test-coverage.md) · [Security: Threat Modeling](../Security/01-threat-modeling.md) · [Workflows: Verification-Driven Development](../Workflows/05-verification-driven-development.md)

---

## Overview

Hallucination — the generation of confident, plausible-sounding output that is factually incorrect — is the failure mode that makes AI-generated code most dangerous to ship without verification. Unlike obvious errors that fail tests or generate runtime exceptions, hallucinated output often passes automated checks, reads correctly to a casual reviewer, and fails only in production under conditions the engineer did not anticipate. The confidence with which language models present incorrect information is one of their most consequential properties: there is no correlation between model certainty and model accuracy for claims about external facts.[^1]

The engineering context introduces specific hallucination categories that general AI guidance does not adequately address. An AI that confabulates a historical date is harmless; an AI that confabulates a function signature produces code that fails at runtime; an AI that confabulates a library's security behavior produces code that is exploitable in production. For engineering teams, the stakes of hallucination are directly proportional to the criticality of the code the AI is generating, and the mitigation must be calibrated to those stakes rather than applied uniformly.

Veracode's Spring 2026 analysis found that 45% of AI-generated code fails at least one security test — a failure rate driven partly by hallucinated assumptions about how security-relevant APIs and frameworks behave. These failures survive code review at high rates because they are structurally plausible: the code looks like code that should work, uses the right abstractions, and implements the right overall pattern. The hallucination is in the details that make the difference between secure and insecure behavior.[^3]

---

## Section 1: High-Risk Hallucination Categories

**Description:** Hallucination is not uniformly distributed across task types. Some categories of AI output are reliably accurate and require minimal verification; others are systematically prone to confabulation and require independent verification before use. Understanding which categories are high-risk allows engineers to allocate verification effort where it generates the most risk reduction rather than applying uniform skepticism to all AI output regardless of its accuracy profile.[^1]

The highest-risk hallucination categories for engineering work are: external API signatures and behavior (the model's training data may include outdated versions or incorrect documentation that was present in training), security API semantics (the distinction between secure and insecure usage of cryptographic or authentication libraries is subtle and frequently confused in training data), framework-specific behavior under edge conditions (how a framework behaves on error, on timeout, or under concurrent load is frequently hallucinated), and version-specific compatibility claims (the model cannot know the current vulnerability or API state of a library at inference time).[^4]

**Recommended Practice:**
- Maintain a team-specific high-risk hallucination list in CLAUDE.md: the specific external libraries, APIs, and frameworks that the team's experience or external research has identified as frequently hallucinated in this codebase. Update this list when a hallucination is discovered and add a corresponding verification requirement for that category.[^5]
- For any AI-generated code that uses an external library not already present in the session context, verify the specific function signatures, method names, and parameter types against the current library documentation before accepting the output. This verification takes two to five minutes per library reference and prevents the class of hallucination that is most common in AI-generated code.[^1]
- Treat AI-generated security-critical code — authentication handlers, authorization checks, cryptographic operations, input sanitization — as requiring human expert review regardless of how correct it appears. The Veracode 45% security test failure rate is concentrated in these categories; casual code review is not sufficient to catch the hallucination patterns that produce security vulnerabilities in this code.[^3]
- When the model generates code using an API in a way that seems surprising or unusually elegant, verify it specifically. Counter-intuitive code that is correct is uncommon; counter-intuitive AI-generated code that is hallucinated is not. The elegance heuristic — "this is too clean to be right" — applies to AI output in contexts where the engineer has enough domain knowledge to find the response surprising.

---

## Section 2: The "Read Before Trust" Verification Pattern

**Description:** The "read before trust" pattern is the primary operational mitigation for hallucination: before accepting any AI-generated claim about a function, method, API, or library behavior, read the authoritative source for that claim. For internal code, the authoritative source is the source file. For external libraries, it is the current version documentation. For framework behavior, it is the framework's own documentation or test suite. Reading the source is the only reliable verification — the model's confident assertion about what a function does is not verification.[^6]

The pattern is not as burdensome as it sounds for the majority of AI-generated code. Most AI output references functions and libraries that the engineer already knows — the verification step is either trivially quick (confirming a known API) or provides valuable information (discovering that an API has changed since the engineer last used it). The expensive verification cases are external libraries the engineer is not familiar with, where "read before trust" requires enough reading to actually understand what the library does — exactly the case where hallucination is most likely and most consequential.

**Recommended Practice:**
- Apply "read before trust" categorically to: any function or method the engineer would not recognize without the AI's assistance, any security-relevant API usage, and any library or framework version the session has not explicitly been given documentation for. These are the categories where hallucination frequency and impact are both highest.[^6]
- Automate the verification step where possible: for internal code, use IDE navigation to jump to the definition of any AI-generated function call and confirm its signature matches the usage. For external libraries, configure the IDE's type checking to flag mismatched signatures immediately rather than requiring manual verification. Type errors from hallucinated signatures are easier to catch than logical errors from hallucinated behavior.[^4]
- For AI-generated test code, apply "read before trust" to the assertions: verify that each test assertion actually tests what the test name claims it tests. An AI-generated test that asserts `result!== null` when the test is named `"should return the correct user ID"` is passing while providing false coverage — a hallucination of test correctness rather than of implementation.[^7]
- Document "read before trust" invocations in code review: when reviewing AI-generated code, note which external references were verified and against which source. This documentation creates a verification record that makes hallucinated output less likely to survive the review process and creates accountability for the verification step.

---

## Section 3: Reducing Hallucination Through Context Injection

**Description:** Hallucination is more likely when the model is generating content about information not present in its current context — it is constructing an answer from training data rather than from provided facts. The most effective way to reduce hallucination for specific claims is to provide the authoritative information as session context: inject the current library documentation, the actual function signature, or the relevant framework configuration. When the correct answer is in the context, the model is more likely to use it rather than construct a plausible alternative from training data.[^5]

This is a more targeted approach than skepticism about all model output. It identifies the specific claims most likely to be hallucinated, injects the authoritative information for those claims, and verifies the output against that information. The combination of context injection and targeted verification is more efficient than general skepticism applied uniformly, because it focuses effort on the high-risk categories while allowing trust in lower-risk output.[^1]

**Recommended Practice:**
- Before sessions involving unfamiliar external libraries, inject the library's README, its primary API documentation, and any version-specific migration notes as session context. This investment in pre-session context preparation substantially reduces hallucination for that library's usage throughout the session.[^5]
- For sessions involving security-critical libraries (OAuth clients, JWT libraries, encryption utilities), inject the library's security documentation and any published security advisories as explicit context. This is the most important application of context injection for hallucination reduction — the stakes of hallucination in security-critical libraries are highest, and the model's training data for security library usage is most likely to include examples of insecure usage.[^3]
- Use the CLAUDE.md library section to document the team's current dependency versions and link to their documentation. A CLAUDE.md that includes "We use express 4.21.2 (docs: [link]) and jsonwebtoken 9.0.0 (docs: [link])" gives the model version-specific anchors that reduce version hallucination for those libraries.[^4]
- After a hallucination is discovered, add the correct information to CLAUDE.md or the relevant task spec as a direct constraint: "The `createToken` function in our auth module takes (userId: string, role: Role, expiresIn: number) — not (payload: object)." Inline corrections in documentation prevent recurrence and act as verification references for future sessions.

---

## Section 4: Test Hallucination as a Specific Risk Category

**Description:** Test hallucination — AI-generated tests that assert the wrong things, test the wrong code paths, or provide false coverage confidence — is a distinct and underappreciated risk in AI-assisted development. Unlike implementation hallucination, which fails at runtime or in code review, test hallucination passes the CI pipeline by definition: a hallucinated test that asserts the wrong condition passes when the wrong condition is met, providing green CI status while covering nothing useful. The result is a codebase with apparent test coverage and actual coverage gaps that only appear when the uncovered path fails in production.[^7]

Sonar's 2026 research found that the verification gap — the proportion of AI-generated code that engineers accept without adequate review — is particularly large for test code. Engineers who would carefully review AI-generated production code accept AI-generated tests more readily, on the assumption that tests are inherently lower-risk. This assumption is wrong when the test is hallucinated: a wrong production code path that is "covered" by a hallucinated test is as unprotected as if no test existed.[^8]

**Recommended Practice:**
- For every AI-generated test, read the assertion body specifically: does the assertion expression test what the test name says it tests? Does the assertion use the right comparison operator, the right expected value, and the right actual value? A test named `"should reject invalid tokens"` that asserts `response.status === 200` is a hallucinated test that will pass while covering nothing.[^7]
- Mutation test AI-generated test suites: run a mutation testing tool (Stryker for JavaScript/TypeScript, mutmut for Python, PITest for Java) on the module covered by AI-generated tests and verify that the mutation score is consistent with the stated coverage. A test suite that achieves 80% line coverage but 30% mutation score has a hallucination problem — the tests are passing but not detecting defects.[^7]
- Request that AI-generated tests include a brief justification for each assertion: "I assert X because [specific behavior the assertion verifies]." This prompting pattern forces the model to articulate why each assertion is correct, which surfaces hallucinated assertions that the model cannot justify without constructing an implausible rationale.[^4]
- Apply the "read before trust" pattern to test assertions with the same rigor as production code: verify that the function or method the test is calling matches the actual implementation signature, that the expected output of the function matches the actual implementation's behavior, and that edge cases covered by test names are actually exercised by the test body.

---

## Summary of Recommended Practices

| Practice | Immediate Action | Owner |
|---|---|---|
| High-Risk Category Awareness | Build team high-risk hallucination list; add to CLAUDE.md | Architect |
| Read Before Trust | Establish verification habit for external library usage; document in code review norms | Engineering team |
| Context Injection | Inject current library docs for external dependencies in relevant sessions | Engineering team |
| Test Hallucination | Add assertion review to AI-generated test acceptance criteria; evaluate mutation testing | QA engineer |

---

[^1]: Simon Willison — "LLM Hallucination: A Practical Framework for 2026," simonwillison.net, March 2026. https://simonwillison.net/2026/Mar/llm-hallucination-practical-framework/
 Hallucination as a structural property of language model generation; confidence-accuracy independence; high-risk hallucination categories for engineering work; "read before trust" as the primary operational mitigation.

[^3]: Veracode — "Spring 2026 GenAI Code Security Update: Despite Claims, AI Models Are Still Failing Security," March 24, 2026. https://www.veracode.com/blog/spring-2026-genai-code-security/
 45% security test failure rate for AI-generated code; hallucination concentration in security-critical code categories; security API semantic hallucination as the primary failure mechanism.

[^4]: Anthropic — "Best Practices for Claude Code," Claude Code Documentation, 2026. https://code.claude.com/docs/en/best-practices
 CLAUDE.md library version documentation; hallucination-reducing constraint patterns; assertion justification prompting; version-specific anchors in session context.

[^5]: Anthropic — "Common Workflows," Claude Code Documentation, 2026. https://code.claude.com/docs/en/common-workflows
 Context injection workflow for external library sessions; security library documentation injection; CLAUDE.md as a version-anchor document; pre-session context preparation for hallucination reduction.

[^6]: Boris Cherny — "How Boris Uses Claude Code," January 2026. https://howborisusesclaudecode.com
 "Read before trust" pattern implementation in daily engineering practice; verification categories and their appropriate sources; the efficiency case for targeted rather than uniform verification.

[^7]: CodeRabbit — "State of AI Code Generation: AI vs. Human Code Report," December 17, 2025. https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report
 Test hallucination as a distinct risk category; false coverage confidence from hallucinated assertions; mutation testing as the verification approach for AI-generated test suites; the verification gap for test code.

[^8]: Sonar (SonarSource) — "Sonar Data Reveals Critical 'Verification Gap' in AI Coding," press release, January 8, 2026. https://www.sonarsource.com/company/press-releases/sonar-data-reveals-critical-verification-gap-in-ai-coding/
 The verification gap in test code specifically; engineer acceptance rates for AI-generated tests vs. production code; the misconception that tests are inherently lower-risk to accept without review.
