# Add Minimal SnarkyJS Checks

## Summary
[summary]: #summary

We describe a minimal layer of SnarkyJS checks that won't hinder typical protocol development.

## Motivation
[motivation]: #motivation

Objective: Empower the protocol development to continue swiftly while preventing maximally complex SnarkyJS bitrot. Out-of-scope for this RFC: re-enabling the rest of those tests in SnarkyJS CI and changing the submodule layout within Mina.

Background: Currently, SnarkyJS and, transitively, SnarkyJS-bindings is included in the main Mina repo as a submodule. Until recently, extensive CI jobs ensured that SnarkyJS remains correct as Mina evolves. Unfortunately, this setup has many issues: (1) Mina is owned by the Mina Foundation and SnarkyJS is owned by O(1) Labs; it's semantically awkward to rely on Mina's CI for testing SnarkyJS, (2) the dependency graph feels a bit inverted; as SnarkyJS is a client of the Mina daemon's internals rather than Mina daemon's internals being a client of SnarkyJS, and most importantly (3) it's causing velocity issues with protocol development. Why? Many parts of SnarkyJS are tightly coupled to zkApp representations and behavior -- this means that even trivial renaming or reordering of fields requires SnarkyJS changes (albeit in a semi-automatic fashion).

Having said that, there are important reasons to keep some tests running on Mina's PRs: For example, many cryptography changes have the potential to break proof generation on SnarkyJS, and by far the best place to address that is the PR introducing the cryptography change.

As of this RFC, all the SnarkyJS tests are configured to "soft-fail" on Mina's CI; this was a quickfix to empower quick Mina development, but is creating opportunities for bitrot in these situations so this RFC aims to quickly reconcile this temporary situation.

Success Metric: Protocol developers are undisturbed by SnarkyJS tests in day-to-day development. However, when deep changes to cryptography are made, the following requirements apply.

Requirements:

* The bare-bones SnarkyJS tests (as described below) run as blocking jobs on Mina PRs
* Any SnarkyJS tests that are more tightly coupled to zkApps do not block Mina CI

Non-requirements (future work):

* The other SnarkyJS tests are executing automatically by CI somewhere (probably the SnarkyJS repo)
* The submodule structure is changed -- this should be done carefully in conjunction with moving Pickles and other cryptography layers out of Mina; perhaps we can remove safely remove it altogether


Hypotheses about how the change meets these needs:

* The tests presented below will not often break with day-to-day zkApp development
* The tests presented below will catch cryptography changes that would otherwise be hard to track and fix during day-to-day SnarkyJS development

In this RFC, we are optimizing for:

* Minimal changes needed to carefully re-enable the important SnarkyJS tests on Mina

## Detailed design
[detailed-design]: #detailed-design

To meet the requirements, it suffices to extract a single creation of an [end-to-end recursive proof](https://github.com/o1-labs/snarkyjs/pull/997/files#diff-32aa0e3ac39d1593084da877ed5ed544175d7058ebeeda1623157b31723b8a9bR40) into a separate CI job.

The existing SnarkyJS will remain unchanged in the "soft-fail" job on the Mina repo.

This can be executed in parallel with the other SnarkyJS tests, so it shouldn't meaningfully impact the PR-CI-time.

There are no other impacts with other components nor any edge cases.

## Test Plan and Functional Requirements
[test-plan-and-functional-requirements]: #test-plan-and-functional-requirements

As this RFC describes the implementation of a test, the test plan for this test, is minimal

1. Testing Goals and Objectives: 
    * The new ZkProgram CI job needs to be sufficient and not too-brittle
2. Testing Approach: 
    * Beyond simple manual testing during implementation, this new CI job needs to be observed carefully over the next few weeks to verify that it _is_ catching issues and _is not_ failing when benign protocol changes are created.
3. Testing Scope:  N/A
4. Testing Requirements: N/A
5. Testing Resources: 
    * A few minutes a week, to review CI jobs and ask core engineers if they are noticing any issues.

## Drawbacks
[drawbacks]: #drawbacks

We are keeping some friction for protocol development, but it seems like a reasonable compromise to merely keep the minimal set of changes.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could keep all the tests removed in the Mina repo and just catch even cryptography changes on the SnarkyJS-side; however, it's usually unclear what exactly is changed, so unlike the converse where a cryptography engineer can ask one of many SnarkyJS developers to help with a fix on their PR, it's likely that only one or even none of the cryptography developers will have an easy solution to proving errors that broke some arbitrary time in the past.

One alternative approach is to increase the scope of these changes to also ensure that the rest of the SnarkyJS tests are running somewhere and/or refactor the submodule structure in the Mina repo, but it's best to take baby-steps and work with minimal shippable chunks as they're ready.

Not doing anything increases the likelihood that SnarkyJS bitrots in complex ways (due to cryptography changes) which dramatically impacts the velocity of SnarkyJS development. Simple bitrotting due to zkApp shapes changing is easy to capture out-of-band of the PR.

## Prior art
[prior-art]: #prior-art

None that I'm aware of

## Unresolved questions
[unresolved-questions]: #unresolved-questions

None
