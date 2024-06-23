## Summary
[summary]: #summary

This RFC describe proposal for solution for measuring test coverage for ocaml code (both unit and integration tests) 

## Motivation
[motivation]: #motivation

Test coverage is a critical component of software testing. It refers to the extent to which a software application is tested. Test coverage is usually measured as a percentage of code or functionality tested. The higher the test coverage, the more comprehensive the testing and the better the quality of the software.

There are several ways to measure test coverage. One of the most common methods is code coverage, which measures the percentage of code that has been executed during testing. Code coverage can be measured at different levels, such as function, statement, and branch coverage.

## Detailed design
[detailed-design]: #detailed-design

Test coverage solution usually consist of two steps:

* Gathering test coverage in from UAT
* Reporting

In below section i will present in details each step of such process

### Toolset

Main requirement is to generate line coverage report using native OCaml toolset and used  reporting tools with well established name and preferably free for open source projects. 

In order to track gain/losses of tests coverage measurement there is a need to allow report generation per commit, so developer is aware how changes influence overall test coverage. Based on current stack the best choice would be to use https://github.com/aantron/bisect_ppx project as it is written in Ocaml and is already used by our dev team. Bisect ppx has already working integration with https://coveralls.io/ and https://about.codecov.io/ tools. For open source projects both solution are free. However codecov requires too extreme permissions on private repositories (https://community.codecov.com/t/full-control-of-private-repositories/3442).
What’s more coveralls supports parallel builds (https://docs.coveralls.io/parallel-builds) which would be very useful based on how we run unit tests (split across of couple of buildkite jobs). 

On top of that we are using github, therefore it would be nice if creator of PR could learn about drop or increase of test coverage without navigating to separate tool. 

Buildkite is our main CI tool. Currently the only acceptable solution should be incorporated in existing CI framework. 


### Test cases

Test cases should cover all the requirements identified in the previous step. Currently we have two levels of tests in ocaml:

- Unit tests
- Test Executive integration tests (from ocaml perspective)

In our solution we will measure combined coverage of both levels.

> 💡 It is important to differentiate both levels. Unit tests are extremely helpful for quick feedback. They are quite fast and reliable. However, they are not excercising product on client environment (or similar). Integration tests, from the other side are filling this gap and can prove that software is working in conditions similar to production usage. However test of that kind usually are slow, costly and less reliable. Therefor we need to understand that combined measurement can mislead about test coverage. 


### Method of measuring test coverage

#### **Unit tests**

1. Run unit tests with 

```bash
dune runtest --instrument-with bisect_ppx
```

Above will generate `bisect#####.coverage`  files 

1. Build coverage report file

```bash
opam install bisect-ppx
opam exec -- bisect-ppx-report  coveralls --git --service-name buildkite --service-job-id $BUILDKITE_JOB_ID --coverage-path . --parallel --repo-token=$COVERALLS_TOKEN coverage.json
```

1. Upload coverage to coveralls:

```bash
curl -X POST https://coveralls.io/api/v1/jobs -F 'json_file=@coverage.json'
```

#### Test executive

1. Build mina with instrumentation flags:

```bash
dune build --instrument-with bisect_ppx
```

2. Create docker out of above mina exec in buildkite (requires new job)
3. Run test executive with images from step2 with additional parameter for downloading coverage 

We need to implement in test-executive tear down step in which we will download all coverage files from pods. We need yet another flag to control behavior and also we need to have persistent storage attached to pods to avoid clearing volume before accessing coverage files. Example command to be utilized:

```bash
test_executive.exe  -- cloud --mina-image gcr.io/o1labs-192920/mina-daemon-instrumented --coverage --debug medium-bootstrap
```

As a result we should have coverage files downloaded locally.

1. We follow steps similar to unit tests

##### Closing parallel upload and trigger report generation

According to docs of coveralls we need to have step in our buildkite system to run web request at the end of pipeline:

```bash
curl -X POST \
  -H 'Content-Type: application/json' \
  -d '{
      "payload": {    
          "build_num": "4193273239",    
          "status": "done"    
      }    
  }' \
  'https://coveralls.io/webhook?repo_token=XXXXX'
```

This step should have dependency to all tests jobs we want to wait for. More details here:
https://buildkite.com/docs/pipelines/dependencies

###### Optional: Add checks to PRs

As a last improvements we can add badge to mina repository providing information about current test coverage. It’s only configuration problem with editing MD files etc.
1. Badge Live Demo: 
https://github.com/coverallsapp/coveralls-node-demo/pull/19#partial-pull-merging

2. Alerts on PR level. Coveralls can define alert on particular percentage of missing coverage.


## Drawbacks
[drawbacks]: #drawbacks

Test coverage touches a lot of moving parts. From altering build scripts (for deb/dockers), through new dhall command and pipelines, till finally enhancing test executive (a.k.a Lucy) with module for downloading coverage raw files.


## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

I don't see currently any alternative for overall strategy. We can always argue about particular step though.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

* Above solution does not differentiate unit and integration level. Do we want to measure coverage in separation (unit vs integration) ?