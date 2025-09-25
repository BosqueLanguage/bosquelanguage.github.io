---
title: "Trustworthy-by-Construction Agentic APIs"
date: 2025-09-26
layout: post
---

The previous post briefly reviewed the state of our ability to translate Bosque code into (decidable) logical formulas for formal analysis of program behavior using SMT solvers. A natural application of this capability is to symbolically validate, in a formal sense, the behavior of an application via 
[property-based-testing or proving the absence of certain classes of bugs](https://bosquelanguage.github.io/2025/09/18/small-models.html).

However there is another space where the ability to validate the safety of a set of actions (expressed in code) is crucial -- agentic AI systems. These systems are intended to operate in, and interact with, the real world to accomplish tasks on our behalf. This includes tasks that are mundane, like searching for a recipe, but also tasks that involve destructive (or irreversible) actions, tasks that involve sensitive data, or tasks that entail financial or reputational costs to rectify if they are done poorly. In this world merely saying that a system is likely to behave correctly is not sufficient. Existing tool use frameworks (such as MCP) provide agents unchecked access to information and systems, which if misused, can have serious consequences. Guarantees about behavior are limited to a statistical statement that, under normal operating conditions, the system will, with high likelihood, behave as expected and this is insufficient for applications which handle sensitive data or perform important operations and, in the presence of malicious actors who seek to confuse or mislead an agent, this risk is unacceptable. 

Earlier this month we proposed a new research direction in an NSF proposal to address this challenge: [Trustworthy-by-Construction Agentic APIs](https://bosquelanguage.github.io/assets/research/papers/TrustWorthyAgents.pdf). This proposal aims to provide a framework for creating trustworthy-by-construction agentic APIs that allow AI agents to interact with the world in a safe and predictable manner even under unexpected (or adversarial) operating conditions!

## The Essence of Trustworthy Agents

Our objective is to provide a framework for creating reliable-by-construction agentic APIs that allow AI agents to interact with the world in a safe and predictable manner under unexpected operating conditions and/or adversarial situations. We take the position that a foundational aspect of AI Agents is the interface they use to interact with the world. Specifically:
1. Agents use **software APIs** to interact with the world.
2. Agents **must be able to reason**, probabilistically and formally, about their actions and the potential consequences.
3. The design and specification of **APIs must be optimized for Agentic use and analysis**. 

Based on [previous work](https://dl.acm.org/doi/abs/10.1145/3622758.3622895) and experience while we hypothesize that, these three components are closely linked. Fundamentally, an API that is easy for a human to use, and for formal verification tooling to reason about, is also an API that is easy for an Large Language Model (LLM) AI Agent to use.

## Creating an API Suitable for a Trustworthy-by-Construction Agent

The first step in creating a truly Agentic optimized programming API framework is to develop a suitable **action language and specification system**. Consider the task of paying a bill. The core API is as simple as:
```
type USD = Decimal ;

entity Account { ... }
entity Confirmation { ... }

api transfer (amt: USD, from: Account, payee: Account): Confirmation;
```

This API provides a syntactically explicit and useable description for an AI Agent. Bosque is designed to move information from implicit (API documentation pages) into explicit syntactic features of the code. In our example the type system allowing the type alias of USD to be used as a unit of currency instead of a simple number with a comment that it is a dollar amount. Rough experimentation shows that, even without specific training, a model like Gemini-2 or GPT-4 has a roughly 20% higher success rate of generating code using this API than the equivalent in TypeScript. 

Even with this significantly improved success rate the API is still ripe for accidental misuse or targeting by malicious actors seeking to confuse our agent. Today we could improve this API in Bosque by adding a simple pre-condition to the API:
```
api transfer(amt: USD, from: Account, payee: Account): Confirmation
  requires 0.0<USD> < amt;
  requires amt <= 100.0<USD>;
;
```
These simple pre-condition ensure the useful constraint that we should never pay a negative amount of money (also a payment of zero does not make sense) and that the API cannot be used to transfer more than $100.0 USD. However, this is not entirely satisfactory as an agent could still erroneously transfer money, even if the amount is small. Further, the dollar amount is hard-coded into the API and does not allow for situational flexibility when the agent is operating in a different contexts/environments.

Conveniently, programming languages have two well known concepts that support these needs – Scoped Dynamic Environments and Uniform Resource Identifiers (URIs). Scoped Dynamic Environments allow us to describe a set of ambient variables that are available to the agent during the execution of some code and URIs are a familiar and flexible (syntactic) way to describe permissions at a logical level – notably the underlying data representation is not exposed via the URI naming scheme. Thus, developers can use these to organically create a model of arbitrary resources and access controls (using Globs) that compactly describe the resources that
an agent can access (and that are accessed by any given API call).
```
api transfer(amt: USD, from: Account, payee: Account): Confirmation
  env ={
    PAYMENT_AUTHORIZATION: OAUTH_TOKEN,
    PAYMENT_LIMIT: USD
  }
  permissions ={
    \account:${from.routing}/${from.account}\
  }

  requires 0.0 < USD > < amt ;
  requires amt <= env . PAYMENT_LIMIT ;
;
```

The API now describes the context in which it can be used, including the required OAuth token for payment authorization and the maximum payment limit. The API also describes the resources the API accesses, in this case the account that the payment is being made from.

When comparing the structure of Model-Context-Protocol (MCP), and the underlying concepts from REST, which envision dynamic discoverability and use of APIs, the proposed Bosque specification language and extensions provide several improvements. As opposed to the verbose and limited type system provided by JSON Schema/OpenAPI, Bosque provides a rich type system that allows for more precise and expressive API and compact specifications. This increases the likelihood of successful correct use by an agent and reduces the number of context tokens used – which reduces the risk of dilution in the prompt. The structured nature of the components of the specification provides a strong structure for the underlying LLM to key on. In the payment example, the specifications make it explicit that the API enforces a payment limit, e.g. the requires clause and the code for the check, are strong signals to the agent that it needs to consider the relation between the payment amount and the limit before calling the API.

The explicit nature and structure of the specifications, as opposed to free form text structure in MCP, also simplifies the tasks of pre-selecting tools (APIs) via Resource Augmented Generation (RAG) and improves the ability of an agent to correctly select which tools (APIs) to use for a given task. The structured nature of the Bosque API specification allows for more precise and effective retrieval of relevant APIs. Once in the agent’s context, the structured nature of the API specification allows the agent to more easily reason about the APIs and which are the best options. In the payment example, the specification is explicit about the permissions, allowing the agent to quickly ignore APIs, that it does not have the needed permissions for.

## CheckFlow Logic for Multi-Step Plan Safety

This design allows us to express safety of single actions (calls) bu agentic tasks are often dynamic multi-step processes that involve sequences of actions and events. The APIs that are used often have constraints on the order of their use or other temporal properties that must be satisfied. For example, a payment API may require that a user confirmation has been made before a payment is sent, or before sending a purchase confirmation, that the reservation has been successfully made in the system. If we look at the example payment API we can extend it with a user confirmation requirement.

```
api transfer(amt: USD, from: Account, payee: Account): Confirmation
  env = {
    PAYMENT_AUTHORIZATION: OAUTH_TOKEN,
    PAYMENT_LIMIT: USD
  }
  permissions ={
    \account:${from.routing}/${from.account}\
  }

  requires 0.0<USD> < amt ;
  requires amt <= env.PAYMENT_LIMIT ||
           $events.contains(ExplicitUserApprove{|payee=payee, amt=amt|});
;
```

This API specification with a behavioral checkflow added the requirement that the payment amount must be less than the limit, or that an explicit user approval event has occurred. This ensures that if the agent is attempting to make a payment that exceeds the limit, it must first obtain explicit user approval, which is recorded as an event. This provides a safeguard against unauthorized or accidental large payments.

This direct and intuitive way to express these properties is expressed in terms of a (linearized) series of events that occur during the execution of a system. Conditions can then be specified over these sequence of events, such as ”the user must approve the payment before it is sent” or ”if a read etag matches the previous write etag then the contents of the read are equal to the contents written”. This has the benefit of being directly expressible as code in a manner a developer (or LLM based agent) is already familiar with and allows for the expression of arbitrary computable conditions over the sequence of events.

This form of logical predicates over sequences of events can also be used to express the effects of API calls as well. A sailboat rental API might have a sequence of events that includes checking the weather forecast and confirming inventory availability. In the example we have an API that allows an agent to rent a sailboat for a given day. The API has two permissions, one for the weather service and one for the internal sailboat availability service (which is otherwise opaque to us). The API is defined as ensuring that if the response is a success, then two events must occur during the execution of the API call:

1. There must have been an API call to check that the weather is safe for sailing that day.
2. There must have been a successful API call to ensure availability and reserve the sailboat inventory for the given quantity.

```
datatype Response = Success | Failure ;

api rentSailboat(quantity: Nat, day: LocalDateTime): Response
  permissions ={
    \web:api.weather.gov/\,
    \ms_service:sailboat_availability\
  },

  ensures Response === Success ==> $events.contains(SafeWeather{|day=day, response=Ok|});
  ensures Response === Success ==> $events.contains(ReserveInventory{|day=day, quantity=quantity, response=Approved|});
;
```

By directly integrating these specifications in the language, and automatically emitting the event generation as part of the runtime, the system can track and verify these properties. These properties are also made syntactically visible to the AI Agents so that they can identify key requirements and effects to reason about. This allows them to effectively identify which APIs are appropriate for accomplishing their objectives and to identify appropriate sequences of API calls to accomplish them (as well as what potential failures and recovery strategies are needed).

## Putting the Trust in Trustworthy Agents!
