## Summary
[summary]: #summary

One paragraph explanation of the feature.

## Motivation
[motivation]: #motivation

Reflect the will of the PRD (Product Requirements Document). If a linkable artifact, include it here. If not, summarize in your own words.

Be sure to include:

* What are the hypotheses behind how this change meets the needs of a PRD?
* What is the expected outcome or outcomes and what are the exit conditions along the way? Include assessment criteria and ideally measurements that will either verify this hypothesis or yield opportunities to halt and abandon the implementation.
* What are we optimizing for in this particular solution? Reflect on alternatives in a later section.
* What use cases does it support?

## Detailed design
[detailed-design]: #detailed-design

In general:
* Be specific. This document is meant to share intent to your colleagues. Share what you believe you will actually do.
* Be decisive. No maybes. Any uncertainty can be captured in the unresolved questions section at the end.

Beyond the design of the change itself, also include details around:
* Security implications
* Performance
* The impact of this change on other components or systems
* Dissect edge cases with examples

**Evergreen, wide-sweeping details**

Evergreen details are hypothesized to be true for the lifetime of this component or system. They are also hard to pinpoint a location in a spec as they are too widesweeping.

Evergreen details are included directly in the RFC in this section.

**Ephemeral details**

Ephemeral details must live in the spec so that they can evolve over time.

In this section, link to one or more lines of code in committed GitHub code or one or more lines within a PR or PR draft.

When in doubt, "the spec" can be a block comment in source code, but there are a few conventions for our existing systems:

In proof systems-related projects:
* The spec is inline comments which are generated via [cargo-spec](https://github.com/mimoo/cargo-specification)

For Mina Daemon:
* Large areas are captured by a [separate specs area](https://github.com/MinaProtocol/mina/tree/develop/docs/specs)
* Default to inline comments. Prefer [cargo-spec](https://github.com/mimoo/cargo-specification) format, so we can extract them later.

For SnarkyJS and other zkapps related projects:
* The spec is inline comments. Prefer [cargo-spec](https://github.com/mimoo/cargo-specification) format, so we can extract them later.

## Test Plan and Functional Requirements
[test-plan-and-functional-requirements]: #test-plan-and-functional-requirements

1. Testing Goals and Objectives: 
    * Specify the overall goals and objectives of testing for the proposed feature or project. This can help set the expectations for testing efforts once the implementation details are finalized.
2. Testing Approach: 
    * Outline the general approach or strategy for testing that will be followed. This can include mentioning the types of testing to be performed (e.g., unit testing, integration testing, performance testing) and any specific methodologies or tools that will be utilized.
3. Testing Scope: 
    * Define the scope of testing by identifying the key areas or functionalities that will be covered by testing efforts. 
4. Testing Requirements: 
    * Specify any specific testing requirements that need to be considered, such as compliance requirements, security testing, or specific user scenarios to be tested.
5. Testing Resources: 
    * Identify the resources required for testing, such as testing environments, test data, or any additional tools or infrastructure needed for effective testing.

## Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

* Why is this design the best in the space of possible designs?
* What other designs have been considered and what is the rationale for not choosing them?
* What is the impact of not doing this?

## Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

* What parts of the design do you expect to resolve through the RFC process before this gets merged?
* What parts of the design do you expect to resolve through the implementation of this feature before merge?
* What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
