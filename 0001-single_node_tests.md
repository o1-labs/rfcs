# RFC title

Single/Pair of nodes tests

## Summary

Idea for implementing missing part of test levels (Single node and pair of nodes tests).

## Motivation

While unit tests and Lucy framework were sufficient solution so far, we are inevitable approaching phase in development which need to have more targeted verification while preserving assurance. Current solution is good in assuring there is no critical bugs in crucial parts of system. In other words unit tests are good for reliable and fast quality verification but does not test built components. From the other side Lucy tests have a great assurance as they are using components in the same (or almost the same) fashion as our potential users/customers. However, they are slow and flaky while not giving developers good experience while analysis test results. There are some differences in debugging production issues and Lucy test cases. However, it is still problematic to discover underlying issue in test, since we are testing product / terraform config and gcloud infrastructure . 

While above opinions are sufficient for existing codebase. It could be problematic and costly to expand test automation suite. This RFC is outlining strategy for complement testing levels for unit and Lucy tests

## Detailed design

My proposal is to implement pyramid of tests like on diagram below:


          ┌───────────────────┐
          │                   │
          │     ZKAPP E2E     │
        ┌─┴───────────────────┴──┐
        │                        │
        │       Lucy Tests       │
        │                        │
      ┌─┴────────────────────────┴─┐
      │                            │
      │     Pair of nodes tests    │
      │                            │
    ┌─┴────────────────────────────┴─┐
    │                                │
    │         Single node test       │
    │                                │
  ┌─┴────────────────────────────────┴──┐
  │                                     │
  │             API Tests               │
  │                                     │
┌─┴─────────────────────────────────────┴─┐
│                                         │
│               Unit tests                │
│                                         │
└─────────────────────────────────────────┘

Where:

- Lucy tests are current test executive tests with full blown proofs and setup on google cloud
- Pair of nodes will be new tailored version of lucy tests with lightweight mina which exercises p2p layer and block/tx propagation
- Single node test will be new tailored version of lucy with focus on node stability/transaction and mempool quality plus various negative cases. It is also recommended to use demo mode.

While **UNIT** and **E2E** and **Lucy** levels are covered by dev-unit tests, lucy framework and zkapp e2e tests project, there are only 2 test cases on **Single node** level. Component level testing give as opportunity to fill gaps in unit tests, which are mostly concerned around modules and **E2E** and **Lucy**, which goal is to give us assurance that there are no severe bugs, like network halt on basic operations. However, they are not catching more sophisticated issues, like for example recent issue with low block production frequency or banning node for old block propagation. What’s more there is no way to cheaply implement automation for such bugs higher than UNIT level. 

Therefore we need to expand automation framework in direction of local single node setup or two nodes communication. It is possible to do it in existing lucy framework, but still more and more gcloud resources will be used causing increase of cost.

Therefore we need to expand automation framework in direction of local single node setup or two nodes communication. It is possible to do it in existing lucy framework, but still more and more gcloud resources will be used causing increase of cost.

There are some ideas to migrate tests to , like here:
[POC Draft - Local Test Engine Integration Test Framework ](https://www.notion.so/POC-Draft-Local-Test-Engine-Integration-Test-Framework-20a0624377ea4ad8a28a1c2892f7029f?pvs=21) 

While i agree with such investment , i have to notice that it is a lot of work with small ROI. Such investment is unsatisfactory as we will receive the same test suite as before, the only change will be a decreased cost of gcloud usage.

Having in mind above, my suggestion is to start building localhost framework initially for single and pair of nodes cherry picking existing lucy codebase, for example:

- Extracting initial test executive network description
- Moving all transaction building and sending to zkapp_test_executive tool
- Moving graphql logic to separate project
- Removing stack driver log engine.

Then make `single_node_tests` project, which is capable to start node, to implement initially simple operations of sending transactions and manipulate node. This solution can satisfy two goals:

- Replace test executive tests which can be migrated to smaller network (like payment test). Eventually leaving just one or two which are necessary to assure critical features (like network bootstrap etc.)
- While building such framework we can automate new test cases for current development need.

**Evergreen, wide-sweeping Details**

We still are using test framework in OCaml using Lucy codebase

## Drawbacks
[drawbacks]: #drawbacks

This change require a lot of investment in test framework.

## Rationale and alternatives

Some alternative could be hybrid solution in which we will only implement tooling in ocaml with cli interfaces and then consuming them in some more friendly language.

