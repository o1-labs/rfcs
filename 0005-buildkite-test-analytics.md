# RFC title

Proposal for using buildkite test analytics module for automation test reporting. 

## Summary

Test Analytics helps track and analyze the steps in that pipeline that involve tests:

* Reports and Visualizes test statuses 
* Works with any continuous integration
* Monitors test suite performance and reliability

We can try to integrate with it in order to gain more quality visibility.


## Detailed design

Test analytics module consume test report in form of junit.xml or internal json format. Unfortunately there is no out of the box integration ready for Ocaml. Therefore, there is a need to create test reporter module which will prepare such file. 

Idea is to parse `test.log` file which is an artifact of test executive test run. Test reporter could be an CLI which convert log to report file. Thanks to that we won't break SPOR rule for test executive. This will also help us to expand solution for other CIs without touching Lucy code directly. An example of execution in our CI system can be like below:

```
./test_reporter.exe test-result generate --log-file "$TEST_NAME.test.log" --format junit --output-file test_result.xml
```

In occasion of introducing test reporter we may also refactor test result evaluation mechanism and extract it to separate module. In order to nicely parse test log, we need also couple of improvements to log structure. Currently there is no indication in the log that test result evaluation is started. We need to know which line in the log indicates start of test result log. This part of the log is also a bit complex as we report contextualized exceptions (failed assertions) / non-contextualized exceptions (errors in logs). There is a proposal to include new level of log:
`Span` which will indicate that there is an section in the log which has start and end. Thanks to that we can nicely parse such log without hackish approach (like know which unrelated to test result log entry is its begin)

Having `test_result.xml` file we need to upload it to buildkite rest api. For example like below:

```
curl -X POST https://analytics-api.buildkite.com/v1/uploads -sv \
  -H "Authorization: Token token=\"$BUILDKITE_TEST_ANALYTICS_TOKEN\"" \
  -F "data=@test_result.xml" \
  -F "format=junit" \
  -F "run_env[CI]=buildkite" \
  -F "run_env[key]=$BUILDKITE_BUILD_ID" \
  -F "run_env[number]=$BUILDKITE_BUILD_NUMBER" \
  -F "run_env[branch]=$BUILDKITE_BRANCH" \
  -F "run_env[commit_sha]=$BUILDKITE_COMMIT" \
  -F "run_env[url]=$BUILDKITE_BUILD_URL" \
  -F "run_env[message]=$BUILDKITE_MESSAGE"
```

As seen above we need to have `BUILDKITE_TEST_ANALYTICS_TOKEN` added to our buildkite agent. This also means that we need to update agents with such token.


**Evergreen, wide-sweeping Details**

Buildkite should still support and expand test analytics module. In the same time we should maintain CI solution within buildkite infrastructure. Also our considerations are only for test executive (a.k.a Lucy) integration tests.
However, once we will blaze trails, we can implement test reports for other kind of tests

## Test plan and functional requirements

1. Testing goals and objectives: 
    * We want to ensure, that statuses of each test is reported to buildkite
2. Testing approach: 
    * run CI job and check buildkite test analytics module

## Drawbacks
[drawbacks]: #drawbacks

We should not do this if we want to switch CI system

## Rationale and alternatives

There is not alternative for Test analytics module in Buildkite. We can try to build our own test management solution though.


## Unresolved questions

* There is an open question if we want to report test failures related to instability of test framework or cloud issues (like not enough nodes provisioned)
