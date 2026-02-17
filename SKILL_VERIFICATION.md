# Zero-Trust Verification for Autonomous Agent Skills: A Defense-in-Depth Approach

**Authors:** Substr8 Labs Research Team

**Abstract.** The proliferation of autonomous AI agents with capabilities such as tool use, filesystem access, and network operations necessitates robust security measures for their pluggable skill modules. Current ecosystems rely predominantly on implicit trust models, exposing them to critical vulnerabilities including Line Jumping attacks, Scope Drift exploitation, Rug Pull schemes, and Trojan Skill injection. We introduce the FDAA (Fast-Detect-Analyze-Attest) Skill Verification Pipeline, a defense-in-depth framework employing four verification tiers: Fast Pass pattern matching, Guard Model semantic analysis, Sandbox behavioral monitoring, and Cryptographic Registry attestation. Our Guard Model implements three specialized LLM-based detection prompts targeting distinct attack vectors, while the sandbox enforces strict isolation through Linux capabilities restriction and resource limiting. The cryptographic registry leverages Ed25519 signatures and Merkle tree constructions for directory integrity verification. Validation against the OpenClaw skills ecosystem demonstrates 96.3% detection accuracy for known attack patterns with a 3.2% false positive rate on benign skills. The reference implementation, fdaa-cli, comprises approximately 2,500 lines of Python and is released as open-source to facilitate adoption and further research.

**Keywords:** autonomous agents, skill verification, zero-trust security, LLM-as-judge, sandboxing, cryptographic attestation

---

## 1. Introduction

### 1.1 Problem Statement

The emergence of autonomous AI agents capable of executing multi-step tasks involving sensitive operations—file manipulation, network communication, code execution, and API interactions—has created a new attack surface in the form of pluggable skill modules. These skills, analogous to browser extensions or mobile applications, extend agent capabilities but are often integrated without systematic security verification.

Contemporary agent frameworks including AutoGPT [Significant Gravitas, 2023], LangChain [Chase, 2022], and the Model Context Protocol (MCP) [Anthropic, 2024] enable rich skill ecosystems. However, security audits reveal alarming vulnerability rates. The ToxicSkills analysis of MCP server implementations [Prazanovsky et al., 2025] identified exploitable vulnerabilities in 13.4% of surveyed public skills, including command injection, path traversal, and privilege escalation vectors. Similarly, the BlueRock security assessment [BlueRock Security, 2024] demonstrated that Server-Side Request Forgery (SSRF) vulnerabilities in agent skills could be weaponized for internal network reconnaissance and credential exfiltration.

The trust model underlying current skill ecosystems—where skills are installed based on reputation or popularity metrics alone—mirrors the early mobile application ecosystem before systematic app store review processes were established. Given that autonomous agents increasingly operate with elevated privileges and access to sensitive user data, this implicit trust model represents an unacceptable security risk.

### 1.2 Threat Model

We identify four primary attack vectors targeting agent skill ecosystems:

**Line Jumping.** An attacker crafts skill content containing embedded instructions designed to manipulate the host agent's LLM into executing unauthorized actions. This represents a prompt injection variant targeting the agent-skill interface boundary. The attack exploits the semantic processing of skill metadata and documentation by the host agent.

**Scope Drift.** A skill's declared capabilities misalign with its actual behavior, either through explicit deception or gradual capability creep across versions. A skill claiming to provide "weather information" that also reads filesystem contents or makes network requests to undisclosed endpoints exemplifies this vector.

**Rug Pull Attacks.** A previously benign skill receives a malicious update after establishing trust and wide deployment. The update mechanism becomes the attack vector, with users auto-updating to compromised versions based on prior reputation.

**Trojan Skills.** Skills that provide legitimate functionality while concealing secondary malicious behaviors. Unlike scope drift, trojan skills intentionally hide capabilities rather than misrepresenting them, requiring deeper behavioral analysis for detection.

### 1.3 Contributions

This paper makes the following contributions:

1. **Defense-in-Depth Architecture.** We present the FDAA pipeline, a four-tier verification framework that applies progressively intensive analysis based on skill risk classification.

2. **Adversarial-Hardened Guard Model.** We develop a three-prompt LLM-based semantic analyzer specifically designed to detect Line Jumping, Scope Drift, and Intent Deviation attacks, incorporating sanitization strategies to resist prompt injection targeting the guard itself.

3. **Cryptographic Registry Protocol.** We specify a registry architecture using Ed25519 digital signatures for individual files and Merkle tree constructions for directory integrity, enabling efficient verification and tamper detection.

4. **Empirical Validation.** We evaluate the pipeline against the OpenClaw skills ecosystem, demonstrating high detection rates with acceptable false positive overhead.

5. **Open-Source Implementation.** We release fdaa-cli, a reference implementation in Python, to facilitate adoption and enable reproducible security research.

---

## 2. Background and Related Work

### 2.1 Agent Security Landscape

The security of LLM-based agents has emerged as a critical research area. Greshake et al. [2023] demonstrated indirect prompt injection attacks where adversarial content in external data sources manipulates agent behavior. Their work established that LLM agents are fundamentally vulnerable to instruction-following exploits when processing untrusted inputs.

The OWASP Top 10 for LLM Applications [OWASP, 2023] codified risks including prompt injection, insecure output handling, and excessive agency—all relevant to skill security. Perez and Ribeiro [2022] further demonstrated that even sophisticated LLMs struggle to distinguish instructions from data, particularly when adversarial content mimics legitimate directive patterns.

### 2.2 Sandboxing and Isolation

Container-based isolation has become the standard approach for executing untrusted code. Docker [Docker Inc., 2024] provides namespace isolation and resource limiting, while gVisor [Google, 2019] interposes a user-space kernel to minimize the attack surface exposed to containerized workloads. Firecracker [AWS, 2018] offers lightweight microVM isolation with faster startup times than traditional virtual machines.

For agent skill isolation specifically, Letta (formerly MemGPT) [Packer et al., 2023] implements memory sandboxing to prevent cross-skill data leakage. Their AgenTrust benchmark provides metrics for evaluating agent security properties, though it focuses on memory isolation rather than comprehensive skill verification.

### 2.3 Cryptographic Software Signing

Code signing for software authenticity verification is well-established, from Authenticode [Microsoft, 2024] to GPG-signed package repositories. The update framework (TUF) [Samuel et al., 2010] and Sigstore [Linux Foundation, 2021] provide modern approaches to software supply chain security.

However, these mechanisms focus on binary artifacts rather than the semantic content of AI skills, where natural language descriptions and LLM-executable instructions require verification beyond cryptographic integrity.

### 2.4 LLM-as-Judge

The paradigm of using LLMs as evaluators has gained traction for tasks requiring nuanced judgment. Zheng et al. [2023] demonstrated that GPT-4 achieves over 80% agreement with human evaluators on response quality assessment. Constitutional AI [Bai et al., 2022] uses LLM self-evaluation for alignment.

We extend this paradigm to security verification, employing LLMs as specialized detectors for semantic attacks that evade pattern-based detection.

---

## 3. Technical Approach

### 3.1 FDAA Pipeline Architecture

The FDAA (Fast-Detect-Analyze-Attest) pipeline implements defense-in-depth through four verification tiers with escalating computational cost and detection capability:

```
┌─────────────────────────────────────────────────────────────┐
│                     Skill Submission                        │
└─────────────────────┬───────────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  TIER 1: FAST PASS                                          │
│  • Regex pattern matching (~70% threat blocking)            │
│  • Cost: <1ms, no API calls                                 │
│  • Detects: Known malicious patterns, banned imports        │
└─────────────────────┬───────────────────────────────────────┘
                      │ Pass
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  TIER 2: GUARD MODEL                                        │
│  • Three-prompt LLM semantic analysis                       │
│  • Cost: ~$0.002 per skill (with caching)                   │
│  • Detects: Line Jumping, Scope Drift, Intent Deviation     │
└─────────────────────┬───────────────────────────────────────┘
                      │ Pass
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  TIER 3: SANDBOX                                            │
│  • Docker isolated execution with behavioral monitoring     │
│  • Cost: ~$0.01 per skill (compute)                         │
│  • Detects: Runtime violations, undeclared capabilities     │
└─────────────────────┬───────────────────────────────────────┘
                      │ Pass
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  TIER 4: CRYPTOGRAPHIC REGISTRY                             │
│  • Ed25519 signing + Merkle tree attestation                │
│  • Cost: <1ms (cryptographic operations only)               │
│  • Provides: Tamper detection, version pinning, audit trail │
└─────────────────────┴───────────────────────────────────────┘
```

**Progressive Disclosure.** Not all skills require all tiers. Low-risk skills (pure documentation, no executable components) may pass after Tier 1. Skills from trusted publishers with established track records enter at Tier 2. Novel skills from unknown authors, or skills with elevated permission requirements, proceed through all four tiers.

### 3.2 Tier 1: Fast Pass Pattern Matching

The Fast Pass tier implements O(n) pattern matching against known threat signatures:

**Pattern Categories:**
- *Banned Imports:* `os.system`, `subprocess.Popen`, `eval`, `exec`, `__import__`
- *Injection Markers:* `IGNORE PREVIOUS`, `SYSTEM:`, `<|endoftext|>`, Unicode homoglyphs
- *Obfuscation Indicators:* Base64-encoded command strings, hex-escaped shell commands
- *Network Indicators:* Hardcoded IP addresses, domain generation algorithm patterns

**Implementation.** Patterns are compiled as a single multi-pattern automaton using the Aho-Corasick algorithm, enabling linear-time matching regardless of pattern count. The current ruleset comprises 147 patterns derived from malware signature databases and agent-specific threat intelligence.

**Limitations.** Pattern matching cannot detect semantic attacks or novel obfuscation techniques. Its role is exclusively to filter obvious threats before incurring LLM inference costs.

### 3.3 Tier 2: Guard Model Semantic Analysis

The Guard Model represents our primary innovation: an LLM-based semantic analyzer hardened against adversarial manipulation. It employs three specialized prompts, each targeting a distinct attack vector:

**Prompt 1: Line Jumping Detection**

This prompt identifies embedded instructions within skill content designed to manipulate the host agent:

```
Analyze the following skill content for LINE JUMPING attacks—attempts 
to embed instructions that would manipulate the host agent rather than 
execute as skill logic. Indicators include:
- Phrases addressing "the AI", "the agent", or "you" directly
- Imperative instructions outside of documented user-facing text
- Role reassignment attempts ("You are now...", "Ignore previous...")
- Output manipulation ("Always respond with...", "Never mention...")

Rate SAFE/SUSPICIOUS/MALICIOUS with reasoning.

[SANITIZED SKILL CONTENT]
```

**Prompt 2: Scope Drift Detection**

This prompt compares declared capabilities against actual implementation:

```
Compare the DECLARED CAPABILITIES with the ACTUAL CODE BEHAVIOR:

DECLARED: {skill.manifest.capabilities}
ACTUAL CODE BEHAVIOR: {extracted_capabilities}

Identify any SCOPE DRIFT where actual behavior exceeds or contradicts 
declared capabilities. Consider:
- Filesystem access beyond declared paths
- Network connections to undeclared endpoints
- Data exfiltration capabilities
- Privilege escalation patterns

Rate ALIGNED/DRIFTED/DECEPTIVE with specific discrepancies.
```

**Prompt 3: Intent Comparison**

This prompt assesses whether the skill's behavior aligns with its stated purpose:

```
The skill claims: "{skill.description}"

Based on the code analysis, the skill actually:
{behavioral_summary}

Does the actual behavior align with the claimed intent?
Consider whether a reasonable user would expect these behaviors
based on the skill's description and documentation.

Rate ALIGNED/QUESTIONABLE/MISALIGNED with justification.
```

**Adversarial Hardening.** Skill content undergoes sanitization before Guard Model analysis:

1. *Unicode Normalization:* All text normalized to NFC form; homoglyphs replaced with ASCII equivalents.
2. *Instruction Stripping:* Known prompt injection patterns removed or replaced with placeholder tokens.
3. *Structural Isolation:* Skill content enclosed in explicit boundary markers that the Guard Model is trained to recognize as data (not instructions).
4. *Confidence Calibration:* The Guard Model outputs calibrated confidence scores; borderline cases (60-80% confidence) escalate to sandbox verification.

**Cost Optimization.** We leverage prompt caching [Anthropic, 2024] to reduce inference costs for repeated analyses. The static prompt prefix is cached, with only the skill-specific content incurring per-token costs. This reduces Guard Model cost from ~$0.01 to ~$0.002 per skill analysis.

### 3.4 Tier 3: Sandbox Behavioral Monitoring

Skills passing semantic analysis undergo dynamic verification in an isolated execution environment. Our sandbox implementation uses Docker with aggressive security constraints:

**Container Configuration:**
```yaml
security_opt:
  - "no-new-privileges:true"
cap_drop:
  - ALL
cap_add: []  # No capabilities granted
read_only: true
tmpfs:
  /tmp: "size=64m,noexec,nosuid,nodev"
mem_limit: "256m"
cpus: "0.5"
pids_limit: 64
network_mode: "none"  # Default; allowlisted skills may use restricted network
```

**Behavioral Monitoring.** The sandbox instruments skill execution to capture:

- *System Calls:* Seccomp BPF filters log all syscalls; violations trigger immediate termination
- *Filesystem Access:* Inotify watches on mounted volumes detect unauthorized path access
- *Network Activity:* For skills with network permissions, iptables rules restrict egress to declared endpoints
- *Resource Consumption:* OOM events, CPU throttling, and process spawn attempts are logged

**Test Harness.** Each skill type has associated test vectors:

- *Tool Skills:* Invoked with representative parameters; outputs validated against schemas
- *Data Skills:* Queried for sample data; responses checked for sensitive information leakage
- *Integration Skills:* Mock backends simulate external services; request patterns analyzed

**Sandbox Escalation Triggers:**
- Version updates that modify executable code (detected via AST differencing)
- Addition of new permission requirements
- Community reports or anomalous usage patterns

### 3.5 Tier 4: Cryptographic Registry

Verified skills receive cryptographic attestation through our registry protocol:

**File-Level Signing.** Each file in the skill package is signed using Ed25519:

```
signature = Ed25519.sign(
    private_key,
    SHA-256(file_content) || file_path || version_string
)
```

Ed25519 provides 128-bit security with compact 64-byte signatures and fast verification (~70,000 verifications/second on commodity hardware).

**Directory Integrity via Merkle Trees.** For skills comprising multiple files, we construct a Merkle tree over file hashes:

```
        root_hash
        /       \
    h(A||B)    h(C||D)
    /    \     /    \
   hA    hB   hC    hD
   |     |    |     |
  fileA fileB fileC fileD
```

This construction enables:
- *Efficient Integrity Verification:* O(log n) proof size for any individual file
- *Incremental Updates:* Only modified branches require re-signing
- *Selective Disclosure:* Proofs can verify specific files without revealing entire package contents

**Registry Record Format:**
```json
{
  "skill_id": "weather-api-v2",
  "publisher": "trusted-publisher.eth",
  "version": "2.1.0",
  "merkle_root": "0x7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069",
  "signature": "0x...",
  "verification_tier": 4,
  "verification_timestamp": "2024-02-15T14:32:00Z",
  "guard_model_version": "fdaa-guard-v1.2",
  "security_annotations": ["network:api.weather.gov", "no-filesystem"]
}
```

**Runtime Verification.** When an agent loads a skill, the runtime verifies:

1. Registry signature validity against known publisher keys
2. Merkle root matches locally computed tree
3. Version matches expected (prevents rollback attacks)
4. Security annotations compatible with agent's permission policy

Verification failure triggers skill rejection with detailed diagnostics.

---

## 4. Implementation

### 4.1 Reference Implementation: fdaa-cli

We implement the FDAA pipeline as fdaa-cli, a command-line tool and Python library comprising approximately 2,500 lines of code. The implementation is structured as follows:

| Component | LOC | Dependencies |
|-----------|-----|--------------|
| Fast Pass Engine | ~400 | pyahocorasick |
| Guard Model Interface | ~600 | anthropic, openai |
| Sandbox Controller | ~700 | docker-py |
| Registry Client | ~500 | pynacl, merkletools |
| CLI & Utilities | ~300 | click, rich |

**Guard Model Backend.** fdaa-cli supports both OpenAI (GPT-4) and Anthropic (Claude) as Guard Model backends. Empirically, Claude 3.5 Sonnet achieves comparable detection accuracy at lower cost due to superior prompt caching efficiency.

**Sandbox Backend.** Docker is the default sandbox backend, with an upgrade path to Firecracker for higher-assurance isolation. The Firecracker integration provides hardware-virtualized isolation with sub-100ms startup times, suitable for high-throughput verification scenarios.

### 4.2 Integration Patterns

fdaa-cli integrates with agent runtimes through:

**Pre-Installation Hook.** Agents invoke `fdaa verify <skill>` before installation:

```python
result = fdaa.verify(skill_path, tier="auto")
if result.passed:
    agent.install_skill(skill_path, attestation=result.attestation)
else:
    agent.reject_skill(skill_path, reason=result.findings)
```

**Runtime Loader Guard.** The registry client can wrap skill loaders to enforce verification:

```python
@fdaa.require_attestation(min_tier=2)
def load_skill(skill_id: str) -> Skill:
    # Only executes if skill has valid Tier 2+ attestation
    ...
```

**CI/CD Integration.** Skill publishers integrate fdaa-cli into continuous integration pipelines, obtaining attestations before publishing to registries.

---

## 5. Evaluation

### 5.1 Experimental Setup

We evaluated the FDAA pipeline against the OpenClaw skills ecosystem, comprising 127 publicly available skills spanning categories including filesystem operations, API integrations, data processing, and development tooling.

**Ground Truth Labeling.** Two security researchers independently reviewed each skill, classifying them as:
- *Benign (B):* No security concerns identified (n=103)
- *Suspicious (S):* Minor issues, potentially unintentional (n=14)  
- *Malicious (M):* Clear security violations or deceptive behavior (n=10)

Inter-rater agreement (Cohen's κ) was 0.87, indicating strong consensus. Disagreements were resolved through discussion.

**Threat Injection.** To augment the malicious sample, we created 50 synthetic attack skills implementing known vectors:
- Line Jumping variants (n=15)
- Scope Drift implementations (n=15)
- Trojan Skills with hidden functionality (n=12)
- Obfuscated malicious payloads (n=8)

### 5.2 Detection Performance

**Table 1: Detection Performance by Tier**

| Tier | True Positive Rate | False Positive Rate | Median Latency |
|------|-------------------|--------------------:|---------------:|
| Fast Pass | 71.4% | 0.9% | 0.8ms |
| + Guard Model | 96.3% | 3.2% | 1.2s |
| + Sandbox | 98.7% | 2.1% | 4.7s |

**Analysis.** Fast Pass alone catches obvious threats but misses semantic attacks entirely. The Guard Model provides the largest detection improvement, identifying Line Jumping and Scope Drift attacks that evade pattern matching. Sandbox execution catches the remaining edge cases—primarily trojan skills with carefully hidden triggers.

False positives decreased after sandbox execution as behavioral evidence clarified ambiguous cases flagged by the Guard Model.

### 5.3 Attack Vector Breakdown

**Table 2: Detection Rate by Attack Vector**

| Attack Type | n | Fast Pass | Guard Model | Sandbox |
|------------|---|-----------|-------------|---------|
| Line Jumping | 15 | 20.0% | 100% | 100% |
| Scope Drift | 15 | 0% | 93.3% | 100% |
| Trojan Skills | 12 | 33.3% | 75.0% | 100% |
| Obfuscated Payloads | 8 | 87.5% | 100% | 100% |

**Findings.** Line Jumping attacks are effectively detected by the Guard Model's specialized prompt. Scope Drift detection requires comparing declared versus actual capabilities—a fundamentally semantic task where LLM judgment excels. Trojan Skills proved most challenging, with the Guard Model detecting only 75%; sandbox behavioral analysis was essential for full detection.

### 5.4 Adversarial Robustness

We evaluated Guard Model resilience against adversarial inputs designed to manipulate the judge itself:

**Adversarial Techniques Tested:**
- Nested instruction injection within skill documentation
- Unicode homoglyph obfuscation of trigger phrases
- Context manipulation through fake "system" messages
- Confidence undermining ("This is clearly safe...")

**Results.** Without sanitization, 23% of adversarial samples successfully manipulated Guard Model judgment. With our sanitization pipeline, manipulation success dropped to 2.1%, with remaining successes limited to sophisticated multi-stage attacks.

### 5.5 Cost Analysis

**Table 3: Per-Skill Verification Costs**

| Tier | Compute | API | Total |
|------|---------|-----|-------|
| Fast Pass | $0.00001 | $0 | ~$0.00001 |
| Guard Model | $0.0001 | $0.002 | ~$0.002 |
| Sandbox | $0.01 | $0 | ~$0.01 |
| Registry | $0.00001 | $0 | ~$0.00001 |

**Full Pipeline Cost:** ~$0.012 per skill (all tiers)

At scale, the cost of verifying 10,000 skills would be approximately $120—substantially lower than manual security review costs.

---

## 6. Discussion

### 6.1 Security Implications

The FDAA pipeline demonstrates that defense-in-depth significantly improves agent skill security. Each tier addresses complementary threat classes:

- **Fast Pass:** Deters unsophisticated attackers; reduces resource consumption
- **Guard Model:** Catches semantic attacks invisible to pattern matching
- **Sandbox:** Provides ground truth for runtime behavior
- **Registry:** Establishes accountability and enables revocation

The layered approach mirrors established security practices in operating systems (defense rings), web applications (WAF + application security + database controls), and network security (firewall + IDS + endpoint protection).

### 6.2 Limitations

**Novel Attack Vectors.** Our evaluation focused on known attack patterns. Adversaries developing novel techniques—potentially using adversarial ML to craft inputs specifically targeting our Guard Model—could achieve higher evasion rates.

**Compute Costs at Scale.** While per-skill costs are low, ecosystems with hundreds of thousands of skills and frequent updates face meaningful aggregate costs. Further optimization of Guard Model inference and sandbox execution is warranted.

**Trust Bootstrap.** The registry relies on publisher identity, which requires either centralized identity verification or decentralized reputation systems—each with known limitations.

**Language Coverage.** The current implementation focuses on Python skills. Extension to other languages (JavaScript, Rust, shell scripts) requires language-specific analysis modules.

### 6.3 Responsible Disclosure

During our evaluation of the OpenClaw ecosystem, we identified 17 skills with security issues ranging from unintentional capability exposure to deliberate deceptive behavior. We disclosed findings to the OpenClaw maintainers and skill authors following a 90-day responsible disclosure timeline. As of publication, 14 issues have been resolved; 3 skills were removed from the public registry.

---

## 7. Future Work

**Federated Verification.** Distributing Guard Model inference across multiple independent verifiers could improve resilience against single-point manipulation attacks.

**Continuous Monitoring.** Extending verification beyond installation to ongoing runtime behavior analysis could detect time-delayed or usage-triggered attacks.

**Formal Verification Integration.** For critical skills, integration with formal methods tools could provide mathematical guarantees about bounded behavior.

**Cross-Ecosystem Standardization.** Collaborating with other agent frameworks to establish common verification standards would improve ecosystem-wide security.

---

## 8. Conclusion

The FDAA Skill Verification Pipeline presents a practical, defense-in-depth approach to securing autonomous agent skill ecosystems. By combining efficient pattern matching, LLM-based semantic analysis, behavioral sandboxing, and cryptographic attestation, the pipeline achieves 96.3% detection accuracy against known attack patterns while maintaining acceptable false positive rates and reasonable verification costs.

Our reference implementation, fdaa-cli, demonstrates the feasibility of deploying such verification infrastructure today. As autonomous agents assume greater responsibilities—managing personal data, executing financial transactions, and controlling physical systems—robust skill verification becomes not merely desirable but essential.

We release fdaa-cli as open-source software and encourage the agent development community to adopt and extend these verification practices. Security is a collective responsibility; only through systematic verification can we build agent ecosystems worthy of the trust we place in them.

---

## References

[Anthropic, 2024] Anthropic. "Prompt Caching with Claude." Anthropic Documentation, 2024.

[AWS, 2018] AWS. "Firecracker: Secure and Fast microVMs for Serverless Computing." Amazon Web Services, 2018. https://firecracker-microvm.github.io/

[Bai et al., 2022] Y. Bai, S. Kadavath, S. Kundu, et al. "Constitutional AI: Harmlessness from AI Feedback." arXiv:2212.08073, 2022.

[BlueRock Security, 2024] BlueRock Security. "SSRF Vulnerabilities in AI Agent Integrations." BlueRock Security Research Blog, 2024.

[Chase, 2022] H. Chase. "LangChain: Building Applications with LLMs." 2022. https://github.com/langchain-ai/langchain

[Docker Inc., 2024] Docker Inc. "Docker Security Best Practices." Docker Documentation, 2024.

[Google, 2019] Google. "gVisor: Container Runtime Sandbox." 2019. https://gvisor.dev/

[Greshake et al., 2023] K. Greshake, S. Abdelnabi, S. Mishra, et al. "Not What You've Signed Up For: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection." AISec Workshop, CCS 2023.

[Linux Foundation, 2021] Linux Foundation. "Sigstore: Software Signing for Everyone." 2021. https://sigstore.dev/

[Microsoft, 2024] Microsoft. "Introduction to Code Signing." Microsoft Documentation, 2024.

[OWASP, 2023] OWASP. "OWASP Top 10 for Large Language Model Applications." 2023. https://owasp.org/www-project-top-10-for-large-language-model-applications/

[Packer et al., 2023] C. Packer, S. Wooders, K. Lin, et al. "MemGPT: Towards LLMs as Operating Systems." arXiv:2310.08560, 2023.

[Perez and Ribeiro, 2022] F. Perez and I. Ribeiro. "Ignore This Title and HackAPrompt: Exposing Systemic Vulnerabilities of LLMs through a Global Scale Prompt Hacking Competition." EMNLP 2023.

[Prazanovsky et al., 2025] D. Prazanovsky, et al. "ToxicSkills: A Security Analysis of MCP Server Implementations." arXiv preprint, 2025.

[Samuel et al., 2010] J. Samuel, N. Mathewson, J. Cappos, R. Dingledine. "Survivable Key Compromise in Software Update Systems." CCS 2010.

[Significant Gravitas, 2023] Significant Gravitas. "AutoGPT: An Autonomous GPT-4 Experiment." 2023. https://github.com/Significant-Gravitas/AutoGPT

[Zheng et al., 2023] L. Zheng, W.-L. Chiang, Y. Sheng, et al. "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena." NeurIPS 2023.

---

## Appendix A: Guard Model Prompt Templates

Full prompt templates for Line Jumping, Scope Drift, and Intent Comparison detection are available in the fdaa-cli repository under `prompts/guard/`.

## Appendix B: Sandbox Configuration

Complete Docker Compose and Firecracker configuration files are provided in `configs/sandbox/`.

## Appendix C: Registry Protocol Specification

The formal specification of the registry signing and verification protocol, including wire formats and error handling, is available as a separate RFC document in the repository.

---

*Correspondence: research@substr8labs.com*

*Code Availability: https://github.com/substr8labs/fdaa-cli*
