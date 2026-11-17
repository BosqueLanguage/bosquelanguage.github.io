---
title: "Searle-iously Capable AI Agents"
date: 2026-11-17
layout: post
---

Have been thinking about the nature of intelligence and the distinction between intrinsic and extrinsic knowledge in AI systems. In many ways this split is widely (if implicitly) assumed, as a fundamental assumption in using extrinsic knowledge such as a skill, md directive, harness, or workflow, and the belief that one flavor or design is better than another. However, we also see a push toward lighter weight systems that lean on, perhaps the bitter lesson, of just getting a bigger/better model and then relying on the intrinsic knowledge/capabilities of the model itself. 

Personally, I favor the view that the extrinsic knowledge is a fundamental part of agentic systems. Thus, for practical reasons, I want to develop some quantitative estimate of the contribution of extrinsic vs intrinsic knowledge in these systems. However, this is also a very fundamental question over the nature of intelligence itself, and the interplay between what an agent (AI or even human) "knows" internally versus what it can leverage from the extrinsic knowledge encoded in the system it operates in.

The following blog post elaborates a bit on this view and thoughts on a methodology for studying the interplay between intrinsic and extrinsic knowledge in agentic AI systems.

# Background
The [Chinese Room thought experiment](https://doi.org/10.1017/S0140525X00005756), proposed by John Searle, explores the limits of systems and intelligence. In this experiment a translator, who does not know Chinese, is in a room filled with books on translations to/from Chinese. Despite the fact that neither component of the system (the translator or the books) understands Chinese, the system as a whole can still read and answer questions that are given to it in Chinese. Although originally posed as a experiment to explore the relation between mere symbolic manipulation, regardless of how fluent, and true understanding or consciousness, it also contains an interesting question about the nature of intelligence & the question of where the system's capabilities come from -- the agent or the room? 

Just as humans use writing to move intrinsic knowledge of tools and their use into external systematized check-lists, run-books, and instruction manuals, we can also offload capabilities from the LLM into strong extrinsic knowledge as well. This idea is implicit in the motivation for many developments in agentic AI systems, e.g. [Skills](https://www.anthropic.com/news/skills) and [MCP](https://modelcontextprotocol.io/), are motivated by the idea of offloading capabilities from the LLM into structured external knowledge but the explicit study and formalization of this shift is still lacking. Thus, this proposal aims to fill this gap by explicitly framing this intrinsic vs. extrinsic system knowledge concept and performing a rigorous empirical evaluation in the space.

From this framing we want to study this issue empirically by performing a sweep over the expressive capabilities of the environment and the semantic richness of what knowledge can be encoded as part of the systems extrinsic intelligence. Starting with known baselines, using simple command line tools (or JSON-RPC) and a range of agentic tasks, we can systematically lift the tool (instruction) knowledge staring from RESTful specs like [Smithy](https://smithy.io/), through dependently typed specifications [Lean](https://lean-lang.org/) and explicitly agentic aware [BAPI Specifications](https://dl.acm.org/doi/abs/10.1145/3842652.3843194) to evaluate the impact of this shift on agentic system performance and safety. 

Importantly, these evaluations will also allow us to monitor reward-hacking behaviors via out-of-policy actions, impacts of tool specification languages on generalization to unseen tasks/tools, and engineering concerns such as model size and token efficiency. We expect these results to provide insight into the fundamental question of intrinsic vs. extrinsic agentic system knowledge as well as to provide practical guidance for the design of future agentic systems -- including post-training, specs, tools, and 
testing or validation based safety systems.

## Methods
The proposed approach to exploring this hypothesis involves systematically shifting increasingly rich extrinsic knowledge and capabilities into structured external specifications, and then, evaluating the impact on agentic safety and performance. We also intend to evaluate how the resulting agents are able to generalize when working in new and unfamiliar domains.

For the purposes of this study we propose to take [AppWorld](https://appworld.dev/) as the target for the evaluation of extrinsic agentic system knowledge. This benchmark is widely used (600+ citations) and provides a comprehensive set of tasks and scenarios for evaluating agentic system knowledge. It also contains a fully documented set of tool APIs, using an OpenAPI spec for all RESTful operations, covering 750 tasks which make use of ~457 operations. We can then extend this with an out-of-distribution domain and task, based on [investment banking scenarios](https://link.springer.com/chapter/10.1007/978-3-030-94583-1_6), which based on prior experience even frontier LLM models have limited knowledge of.

Starting with the sparse(r) specification provided by the OpenAPI specs in AppWorld the plan is to write incrementally richer versions in more expresive extrinsic tool/spec description languages and then RL fine-tune agents with these richer specifications to evaluate how they impact learning rates and final capabilities. We will then apply these agents in our unseen domains to evaluate their generalization performance, robustness, and safety behaviors in out-of-distribution scenarios.

For illustration consider the Venmo payment API in AppWorld. As described in the paper ``We do not use [the OpenAPI specs] in our experiments, as they are extremely verbose making them hard to fit in the model context limits''. Thus, although we plan to revisit them as a baseline, for the purposes of this proposal we start with the (simplified) [Smithy](https://smithy.io/) version of the API specification:

```
@documentation("Send money to a user.")
@http(method: "POST", uri: "/venmo/transactions", code: 200)
operation transaction {
    input: transactionInput
    output: transactionOutput
}

structure transactionInput {
    @required
    @jsonName("receiver")
    @pattern("^[^\\s]+@[^\\s]+\\.[^\\s]+$")
    @documentation("Email address of the receiver.")
    rcvr: String

    @required
    @range(min: 0.01)
    @documentation("Must be greater than 0.")
    amt: BigDecimal

    @required
    @jsonName("token")
    @documentation("Access token obtained from Venmo app login.")
    token: String

    @jsonName("note")
    @documentation("Description of or note about the transaction.")
    note: String
}
```

Even compared to simple normative tool descriptions (or Skills) this specification already allows us to encode and strongly enforce key aspects of the API, such as the regex data format for emails and the lower bound on the amount. 

However, if we move to a more expressive type system, such as the [Lean](https://lean-lang.org/) language, we can encode these constraints and other more complex capabilities explicitly in the type system of the declarations. These can then be used to enforce correctness at plan generation time as well as, by their explicit nature, to provide more extensive guidance on the capabilities and usage of the API.

```
-- POST /venmo/transactions
def isEmailRegex : Regex := re! r"^[^\s]+@[^\s]+\.[^\s]+$"

abbrev Email := { value : String // isEmailRegex.test value }
abbrev MinimumTransactionAmount := { value : Rat // 0.01 <= value }

-- Valid access tokens are obtained from Venmo app login `Venmo.login`
opaque validToken : String -> Prop
def Token := { value: String // validToken value }

structure transactionInput where
  rcvr : Email
  amt : MinimumTransactionAmount
  token : Token
  note : Option String

-- Send money to a user.
opaque transaction :
  transactionInput -> IO (Option String)
```

The Lean declaration mirrors the structure and constraints of the API specification with the explicit encoding of the email format and minimum transaction amount. However, it is also able to attach (aka brand) a logical condition on the AccessToken, and enforce that it is only satisfied when the token is produced as the result of a valid login processes. Thus, ensuring that an agent using this API cannot forge an access token string (or use one that it happened to see during pre-training!) and must use a valid authentication flow. Further, as this information, and the reference to how to get a valid accessToken are now explicitly encoded in the tool signature information the agent has an explicit signal for how to (safely) use this API!

Although the Lean supports encoding complex constraints, like the API usage relation between acquiring an access token via a specific API before calling the transaction API, it relies on a "trick" of branding a type with an opaque predicate to indirectly enforce the temporal condition. Although this works it also obscures the intent of the condition -- the relationship between API calls is encoded indirectly rather than explicitly and requires an agent to examine multiple types and predicates to understand the (implicit) intent. Instead [Bosque](https://github.com/BosqueLanguage/BosqueCore) (with [BAPI specifications](https://dl.acm.org/doi/abs/10.1145/3842652.3843194)) provides a more direct way to encode and enforce such temporal and sequential constraints through its resource specifications and event log mechanism. This allows us to write the API as:

```
type Email = CString of /[^%t;%n; ]+@[^%t;%n; ]+\.[%t;%n; ]+/
type USD = Decimal;

type Token = CString{1,};

%** Send money to a user. **%
api transaction(rcvr: Email, amt: USD, token: Token, note: Option<String>): APIResult<String>
    env={
        CurrentUser: Email
    }
    requires amt > 0<USD>;
    requires $events.contains(LoginSuccess{|id=env.CurrentUser, token=token|});
;
```

In this example Bosque uses an explit event log to enumerate prior actions and allows the API to specify preconditions (similar to [Dogwood](https://aws.amazon.com/blogs/opensource/introducing-dogwood-runtime-verification-for-ai-agents)) over these actions that must be satisfied for this API to be usable. In addition to strong constraints on usage, Bosque also provides explicit (and token efficient) mechanisms for specifying other common API requirements and recommendations including needed environment variables, accessed resources, and reccomended best-practice (as opposed to mandatory) usage patterns.

## Research Outcomes

### Validity and Extent of the Intrinsic and Extrinsic Knowledge Hypothesis:
The formalization of this hypothesis and a comprehensive empirical evaluation on agent performance including token efficiency, success rates, and policy compliance is the key research question. We expect the formalization and quantification of the ability to use enriched extrinsic knowledge in agentic systems to have a substantial impact on training and construction -- possibly representing a novel ``scaling axis'' for agentic system design.

### Practical Implications for Agent Tools, Specs, and Context:
The quantitative results of this study will be of immediate impact and use in the design and implementation of tools, specifications, and frameworks for agentic systems.  Specifically, we plan to measure token efficiency, LLM inference costs, and policy violation rates, in addition to overall agent success rates. Time and cost permitting we also plan to perform ablations over model sizes to understand the limits of extrinsic knowledge offloading.

### Policies and Verification over Agent Trajectories:
The use of the validation/verification focused languages, Lean and Bosque, provide direct insights into the practical applicability and effectiveness of these languages for verification of agent trajectories. This information will provide a roadmap for both the construction of specification/policy languages and the development of verification tools to enable the scalable verification of agent trajectories and ability to provide strong guarantees on agentic behaviors.