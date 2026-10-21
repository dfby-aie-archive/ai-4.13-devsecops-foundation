# [4.13] DevSecOps Foundations

## Lesson Overview
![Title](./assets/images/infographic-4.13-devsecops-foundations.png)

## Dependencies
- [Self Studies](./studies.md)
- [Lesson](./lesson.md)
- [Assignment](./assignment.md)

## Lesson Objectives
* **Explain** what DevSecOps is and how it differs from traditional DevOps
* **Describe** the shift-left security approach and its benefits
* **Identify** common security risks in CI/CD pipelines and dependency vulnerabilities
* **Apply** secrets management best practices to protect sensitive credentials
* **Explain** why pipelines pin tools by immutable hashes (2026 Trivy supply-chain attack)
* **Scan** dependencies with Snyk on the Snyk website and as a CircleCI security gate

## Lesson Plan

| Duration | What | How or Why |
|----------|------|------------|
| 10 min | Warm up | Intro and lesson overview, why security matters in DevOps |
| 15 min | Part 1: What is DevSecOps? | Definition, traditional vs DevSecOps, principles, real-world Equifax breach |
| 10 min | Activity 1 — Real-world discussion | Discuss Equifax breach and what DevSecOps could have prevented |
| 15 min | Part 2: Shift-left security | What shift-left means, cost table, benefits, shift-left in practice, examples |
| 15 min | Part 3: Pipeline security risks | Hardcoded secrets, vulnerable dependencies, insecure images, attack scenarios |
| 15 min | Part 4: Secrets management | What secrets are, horror stories, environment variables, best practices |
| 10 min | Activity 2 — Spot the security mistake | Learners identify security mistakes in code and config snippets |
| 15 min | Part 5: Dependency vulnerabilities | CVEs, Log4Shell example, transitive dependencies, best practices |
| 10 min | Part 6: Security scanning types | SAST, DAST, SCA and container scanning table; tools used in 4.13 and 4.14 |
| 15 min | Part 7: Case study — Trivy supply-chain attack | What happened, tags vs hashes, pinning by commit SHA / digest / checksum, least privilege |
| 45 min | Part 8: Activity 3 — Snyk | Part A: import devops-demo on the Snyk website and read results; Part B: `snyk_scan` CircleCI job gating publish, break and fix with commons-text 1.9 |
| 5 min | Recap and wrap up | Key takeaways, preview Lesson 4.14 (OSV-Scanner, OWASP ZAP), Q&A |
| **Total** | | **180 min** |