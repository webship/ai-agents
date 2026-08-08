# Policy on the use of AI when contributing to Drupal — saved summary

Source: <https://www.drupal.org/docs/develop/issues/issue-procedures-and-etiquette/policy-on-the-use-of-ai-when-contributing-to-drupal> (page last updated 11 June 2026; saved here 2026-08-03). The live page is the source of truth — re-read it before the first contribution of a session. Related: the [Drupal.org Terms of Service](https://www.drupal.org/terms), the [Drupal coding standards](https://www.drupal.org/docs/develop/standards), and the [commit message standard](https://www.drupal.org/node/3586390).

## Why the policy exists

AI makes it cheap to produce a lot of code and text, and that lands as pressure on the people who review and maintain Drupal. The policy is not about which tools you use. It is about every contribution being something a real person stands behind.

## Be a collaborator first — this applies whether or not AI is involved

**Above everything else: be a good listener and collaborator** with the project maintainer, the original issue reporter and the other contributors on the issue. A **drive-by contribution** — one where you do not follow up on feedback, or you ignore the previous discussion on the issue — **will likely result in an account ban**.

Understanding the issue and collaboratively finding the right solution is the critical part of contributing. Writing the code is just executing that solution, and it only succeeds on top of that foundation. AI does not change this.

## Core principle: you are responsible for what you submit

- If a reviewer asks you to explain a decision or a piece of logic, **you must be able to answer**. "The AI wrote it" is grounds for immediately closing the contribution.
- You are fully responsible for the integrity of the submission. AI can hallucinate nonexistent packages (a supply-chain risk), introduce subtle security vulnerabilities, and produce unhelpful refactors that dump review burden on others. **Verify every dependency, the logic, and the security implications before submitting.**

## Copyright and licensing

Models sometimes emit verbatim code from other copyrighted projects. You alone are responsible for making sure AI-generated code infringes no third-party copyright and is fully GPL-compatible, including the copyright rules of your own jurisdiction. Not knowing where the code came from is not a defence.

## Contributions that do not meet the standard

- Dumping code into an issue without reading the thread or acknowledging earlier attempts to solve the problem.
- Posting an MR whose automated checks fail and leaving others to fix it.
- Adding AI-generated code to someone else's MR without their knowledge and without disclosure.
- Dumping a large AI patch, then abandoning the issue when human feedback is requested.
- Submitting code that ignores the conclusions of prior architectural discussion.
- Proposing a full rewrite of a module off the back of an AI review, without first engaging the existing maintainers.
- Using AI to generate issue summaries, comments or reviews that carry no independently verified technical insight (for example, summarizing a thread just to collect contribution credit).
- Posting issue comments, MR descriptions or forum posts that are **unreviewed AI output rather than your own words**.

## Disclosure

Mandatory when AI generated a **significant** portion of the code or text — entire functions, classes, architectural scaffolding, extensive documentation blocks. Required **regardless of how thoroughly you reviewed the output**. Not required for minor uses: single-line autocomplete, basic syntax corrections.

Use the template's AI-disclosure section when there is one. Otherwise append a clear, human-written line at the end of the issue summary, comment or MR description:

```
AI-Generated: Yes (Used GitHub Copilot to help generate the boilerplate for this feature).
```

The policy's own second example shows the same shape for prose:

```
AI was used in the drafting of this policy, to help review for clarity, clean up language, and check grammar.
```

Disclose on the **commit message AND the MR/PR description**.

## Enforcement

Education first where possible. A **temporary ban** may be used so guidance can be delivered and confirmed as read before the account is restored. Contributors who show good intent and respond constructively may not need one. **Clear disregard for the policy, or disrespect toward maintainers or other contributors, can bring a permanent ban.** Drupal Association staff and site moderators have limited capacity for the volume, so patience is expected while moderation scales.

## Policy vs. Terms of Service

This is a **contribution policy**: contributor behaviour, changeable through normal community governance. The **Terms of Service** is the legal framework for all use of Drupal.org, and only the Drupal Association can change it. The intent is to establish this as community norms and issue-queue etiquette first, then reinforce it through TOS updates. One area the TOS still needs to address explicitly: **automated agents acting on behalf of contributors.** Expect clearer rules there as tooling gets more capable.

## Google Summer of Code

GSoC is a learning experience. The policy strongly recommends **not** using AI, so you learn the foundations — which serves you better later even when you do use AI, because you will understand how to prompt it and how to read what it returns.
