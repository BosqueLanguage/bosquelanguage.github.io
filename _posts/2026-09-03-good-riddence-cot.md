---
title: "Good Riddance Chain-of-Thought"
date: 2026-09-03
layout: post
---

Well Chain-of-Thought (CoT) has been, and perhaps the end of trying to monitor it, has been a topic of conversation today, and honestly, I am happy to say good riddance to this approach to trying to ensure safety and correctness. Trying to read the inner monolouge (if that is even what CoT faithfully represents) and use that information to drive correct behavior is fundamentally a round-about and fragile approach to achieving what we want -- a correct outcome.

In fact we have many-many years of experience on how to get correct outcomes out of systems that get distracted, overlook things, and sometimes even are misaligned with our goals! Humans have all of the risks of LLM agents and yet, rather than trying to accomplish correctness by monitoring or shaping inner thoughts, we have developed much more robust methods for ensuring desired outcomes. We ask for _proof of satisfaction_ in addition to simply producing an answer or artifact!

This approach is so deeply ingrained in many experiences that it often feels like an inherent part of the thought process itself but in fact it is a strategy for achieving reliable outcomes that can be documented and applied independently. 

Consider some examples where there is an answer (or artifact) produced _and_ also some proof of validity provided to demonstrate that the answer or artifact meets the desired criteria:

1. **A Certificate of Correctness** In the best case (such as formal verification of software), a certificate of correctness can be checked quickly and demonstrates the correctness of the software with respect to its specification. Regardless of the thought process, intents, or intermediate actions needed to produce it.
2. **Checklist** A checklist can serve as a lightweight form of proof, ensuring that all necessary steps or criteria have been addressed. By verifying each item on the checklist -- think CI pipeline, a security review, test set run, perf-lab, linters, etc. -- one can gain confidence in the correctness of the outcome without needing to rely on the internal generation process.
3. **Best Practice and Derivation** Informal demonstrations that an answer or artifact conforms to best practices or is derivable using an established methodology can provide confidence in its correctness. Think show your work on an exam or an engineering document showing the architecture matches a standard design pattern -- the reality of getting to the answer might not have mattered, but as long as we know the answer can be derived correctly we can be confident in it.
4. **Consistency Checks** Ensuring that an answer or artifact is consistent with known facts, previous results, or established constraints can serve as a form of proof. By verifying consistency, with say a comprehensive test suite or independetly derived results, one can gain confidence in the correctness of the outcome without needing to rely on the internal thought process.

Getting to robustness without trust on the perfection of each internal step is really our goal -- both for systems that involve humans and for AI agents. Ideally we want both internally aligned (conciencious) workers _and_ externally verifiable outcomes, but if we have to choose, the latter provides a more reliable path to correctness and safety. Experiences in software development show that over focusing on internal mental alignment, e.g. depending on developer skill over validation and CI checklists, has proven less effective than ensuring that actions and outcomes can be verified and trusted independently.

So, if we are at the end of CoT, it is an opportunity to focus more on externally verifiable outcomes and robust methods for ensuring correctness as a core part of the system, rather than relying on a perfect process running every time.