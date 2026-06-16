# Lab 3 — CI/CD PR Gate

## 1. Selected Path
I selected **GitHub Actions** because I have access to GitHub.com and it provides easy CI/CD integration with pull requests.


## 2. Green CI Run
Link to successful CI run:
PASTE YOUR PR LINK HERE (Checks must be green)

Example:
https://github.com/daliquido/DevOps-Intro/pull/


## 3. Failed Run (1.5 proof)

### Failed CI run:
PASTE LINK OR SCREENSHOT HERE

### Fix commit:
COMMIT HASH OR MESSAGE:
fix: restore passing tests

---

## 4. Branch Protection Screenshot
PASTE SCREENSHOT OF:
Settings → Branches → main rule

Must show:
- Require status checks
- vet / test / lint selected

---

## 5. Design Questions (1.2)

### a) Why pin ubuntu version instead of ubuntu-latest?
Because ubuntu-latest is a moving target. GitHub changes it without warning (e.g. 22.04 → 24.04). That can break builds unexpectedly due to:

* different Go preinstalled versions
* different system libraries
* different shell/tools behavior

Pinning ensures reproducibility and stable CI behavior.

### b) Why split vet, test, lint?
Splitting gives:

* parallel execution (faster CI)
* clearer failure reporting
* isolation (lint failure doesn’t hide test failure)
* better caching and scalability

If combined:

* everything runs sequentially
* one failure may stop later checks
* slower feedback loop
* harder to diagnose issues

### c) Why SHA pinning in GitHub Actions?
SHA pinning prevents supply chain attacks via GitHub Actions tampering.

If you use a tag like @v4, that tag can be moved or compromised. A malicious update could inject code into your CI pipeline.

SHA pinning ensures:

* exact immutable action version is used
* prevents “tag hijacking”

Example incident:

* 2022-03 GitHub Actions supply chain risk awareness (reviewdog / tj-actions ecosystem warnings and compromise discussions widely highlighted in March 2022 security advisories)

Core idea: attackers can modify tags or upstream actions → CI becomes entry point.

### d) What is permissions:?
It defines least-privilege access for workflows, reducing security risk.
permissions: defines what the workflow is allowed to do.

Example:

permissions:
 contents: read
### e) GitLab difference between stage and job?
Stages define execution order, jobs are units of work. dependencies controls artifact flow, not execution order.