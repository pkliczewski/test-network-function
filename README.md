# Test Network Function ![build](https://github.com/test-network-function/test-network-function/actions/workflows/merge.yaml/badge.svg) [![Go Report Card](https://goreportcard.com/badge/github.com/test-network-function/test-network-function)](https://goreportcard.com/report/github.com/test-network-function/test-network-function)

This repository contains a set of network function test cases and the framework to build more.  It also generates reports
(claim.json) on the result of a test run.
The tests and framework are intended to test the interaction of Cloud-Native Network Functions (CNFs) with OpenShift
Container Platform.

The suite is provided here in part so that CNF Developers can use the suite to test their CNFs readiness for
certification.  Please see "CNF Developers" below for more information.

## Container

### Running

A container that can be used to run the tests is available from [quay.io](https://quay.io/repository/testnetworkfunction/test-network-function)

To pull the latest container and run the tests you use the following command. There are several required arguments:

* `-t` gives the local directory that contains tnf config files set up for the test.
* `-o` gives the local directory that the test results will be available in once the container exits.
* Finally, list the specs to be run must be specified, space-separated.

Optional arguments are:

* `-k` gives a path to one or more kubeconfig files to be used by the container to authenticate with the cluster. Paths must be separated by a colon.
* `-i` gives a name to a custom TNF container image. Supports local images, as well as images from external registries.

If `-k` is not specified, autodiscovery is performed.
The autodiscovery first looks for paths in the `$KUBECONFIG` environment variable on the host system, and if the variable is not set or is empty, the default configuration stored in `$HOME/.kube/config` is checked.

```shell-script
./run-container.sh -k ~/.kube/config -t ~/tnf/config -o ~/tnf/output diagnostic generic
```

*Note*: Tests must be specified after all other arguments!

### Building

You can build an image locally by using the command below. Use the value of `TNF_VERSION` to set a branch or tag that will be
installed into the image.

```shell-script
docker build -t test-network-function:v1.0.5 --build-arg TNF_VERSION=v1.0.5 .
```

To make `run-container.sh` use the newly built image, specify the custom TNF image using the `-i` parameter.

```shell-script
./run-container.sh -i test-network-function:v1.0.5 -t ~/tnf/config -o ~/tnf/output diagnostic generic
```

## Dependencies

At a minimum, the following dependencies must be installed *prior* to running `make install-tools`.

Dependency|Minimum Version
---|---
[GoLang](https://golang.org/dl/)|1.14
[golangci-lint](https://golangci-lint.run/usage/install/)|1.32.2
[jq](https://stedolan.github.io/jq/)|1.6
[OpenShift Client](https://docs.openshift.com/container-platform/4.4/welcome/index.html)|4.4

Other binary dependencies required to run tests can be installed using the following command:

```shell-script
make install-tools
```

Finally the source dependencies can be installed with

```shell-script
make update-deps
```

*Note*: You must also make sure that `$GOBIN` (default `$GOPATH/bin`) is on your `$PATH`.

*Note*:  Efforts to containerize this offering are considered a work in progress.

## Available Test Specs

There are two categories for CNF tests;  'General' and 'CNF-specific'.

The 'General' tests are designed to test any commodity CNF running on OpenShift, and include specifications such as
'Default' network connectivity.

'CNF-specific' tests are designed to test some unique aspects of the CNF under test are behaving correctly.  This could
include specifications such as issuing a `GET` request to a web server, or passing traffic through an IPSEC tunnel.

### General

The general-purpose category covers most tests.  It consists of multiple suites that can be run in any combination as is
appropriate for the CNF(s) under test:

Suite|Test Spec Description|Minimum OpenShift Version
---|---|---
diagnostic|The diagnostic test suite is used to gather node information from an OpenShift cluster.  The diagnostic test suite should be run whenever generating a claim.json file.|4.4.3
generic|The generic test suite is used to test `Default` network connectivity between containers.  It also checks that the base container image is based on `RHEL`.|4.4.3
multus|The multus test suite is used to test SR-IOV network connectivity between containers.|4.4.3
operator|The operator test suite is designed basic Kubernetes Operator functionality.|4.4.3
container|The container test suite is designed to test container functionality and configuration|4.4.3

Further information about the current offering for each test spec is included below.

### diagnostic tests

The `diagnostic` test spec issues commands to poll environment information which can be appended to the claim file.
This information is necessary to ensure a properly spec'd environment is used, and allows the claim to be reproduced.
As of today, the `diagnostic` test suite just polls `Node` information for all `Node`s in the cluster.  Future
iterations may consider running `lshw` or similar types of diagnostic tests.

### generic tests

The `generic` test spec tests:
1) `Default` network connectivity between containers.
2) That CNF container images are RHEL based.

To test `Default` network connectivity, a [test partner pod](https://github.com/test-network-function/cnf-certification-test-partner)
is installed on the network.  The test partner pod is instructed to issue ICMPv4 requests to each container listed in
the [test configuration](./test-network-function/tnf_config.yml), and vice versa.  The test asserts that
the test partner pod receives the correct number of replies, and vice versa.

In the future, other networking protocols aside from ICMPv4 should be tested.

### multus tests

Similar to the `generic` test spec, the `multus` test spec is utilized for CNFs that utilize multiple network
interfaces.  As of today, the `multus` test suite just tests that
a [test partner pod](https://github.com/test-network-function/cnf-certification-test-partner) can successfully ping the secondary
interface of the CNF containers.  Since SR-IOV is often utilized, and the secondary interface of a CNF cannot be
accessed in user space, the test is unidirectional.

### operator tests

Currently, the `operator` test spec is limited to three test cases called `OPERATOR_STATUS`.  `OPERATOR_STATUS`
checks that the `CSV` corresponding to the CNF Operator is properly installed and operator `Subscription` is available.
It checks whether operator is using privileged permissions by inspecting `clusterPermissions` specified in `CSV`. 

In the future, tests surrounding `Operational Lifecycle Management` will be added.

### container tests

The `container` test spec has the following test cases:

Test Name|Description
---|---
`HOST_NETWORK_CHECK`|Ensures that the CNF pods do not utilize host networking.  Note:  This test can be disabled for infrastructure CNFs that should utilize host networking.
`HOST_PORT_CHECK`|Ensures that the CNF pods do not utilize host ports.
`HOST_PATH_CHECK`|Ensures that the CNF pods do not utilize the host filesystem.
`HOST_IPC_CHECK`|Ensures that the CNF pods do not utilize host IPC namespace to access or control host processes.
`HOST_PID_CHECK`|Ensures that the CNF pods do not utilize host PID namespace to access or control host processes.
`CAPABILITY_CHECK`|Ensures that the CNF SCC is not configured to allow `NET_ADMIN` or `SYS_ADMIN`.
`ROOT_CHECK`|Ensure that the CNF pods are not run as `root`.
`PRIVILEGE_ESCALATION`|Ensure that the CNF SCC is not configured to allow privileged escalation.

In the future, we are considering additional tests to ensure aspects such as un-alteration of the container image.

## Performing Tests

Currently, all available tests are part of the "CNF Certification Test Suite" test suite, which serves as the entrypoint
to run all test specs.  `CNF Certification 1.0` is not containerized, and involves pulling, building, then running the
tests.

By default, `test-network-function` emits results to `test-network-function/cnf-certification-tests_junit.xml`.

The included default configuration is for running `generic` and `multus` suites on the trivial example at
[cnf-certification-test-partner](https://github.com/test-network-function/cnf-certification-test-partner).  To configure for your
own environment, please see the Test Configuration section, below.

### Pulling The Code

In order to pull the code, issue the following command:

```shell-script
mkdir ~/workspace
cd ~/workspace
git clone git@github.com:test-network-function/test-network-function.git
cd test-network-function
```

### Building the Tests

In order to build the test executable, first make sure you have satisfied the [dependencies](#dependencies).

```shell-script
make build-cnf-tests
```

*Gotcha:* The `make build*` commands run unit tests where appropriate. They do NOT test the CNF.

### Testing a CNF

Once the executable is built, a CNF can be tested by specifying which suites to run using the `run-cnf-suites.sh` helper
script.
Any combintation of the suites listed above can be run, e.g.

```shell-script
./run-cnf-suites.sh diagnostic
./run-cnf-suites.sh diagnostic generic
./run-cnf-suites.sh diagnostic generic multus
./run-cnf-suites.sh diagnostic operator
./run-cnf-suites.sh diagnostic generic multus container operator
```

By default the claim file will be output into the same location as the test executable. The `-o` argument for
`run-cnf-suites.sh` can be used to provide a new location that the output files will be saved to. For more detailed
control over the outputs, see the output of `test-network-function.test --help`.

```shell-script
cd test-network-function && ./test-network-function.test --help
```

*Gotcha:* The generic test suite requires that the CNF has both `ping` and `ip` binaries installed.  Please add them
manually if the CNF under test does not include these.  Automated installation of missing dependencies is targetted
for a future version.

## Test Configuration

Configuration is accomplished with `tnf_config.yml` by default. An alternative configuration can be provided using the
`TNF_CONFIGURATION_PATH` environment variable.

This config file contains several sections, each of which configures one or more test specs:

Config Section|Purpose
---|---
generic|Describes containers to be tested with the `generic` and `multus` specs, if they are run.
cnfs|Defines which containers are to be tested by the `container` spec.
operators|Defines which containers are to be tested by the `operator` spec.

`testconfigure.yml` defines roles, and which tests are appropriate for which roles. It should not be necessary to modify this.


### generic

The `generic` section contains three subsections:

* `containersUnderTest:` describes the CNFs that will be tested.  Each container is defined by the combination of its
`namespace`, `podName`, and `containerName`, which are also used to connect to the container when required.

  * Each entry for `containersUnderTest` must also define the `defaultNetworkDevice` of that container.  There is also
  an optional `multusIpAddresses` that can be omitted if the multus tests are not run.

* `partnerContainers:` describes the containers that support the testing.  Multiple `partnerContainers` allows
for more complex testing scenarios.  At the time of writing, only one is used, which will also be the test
orchestrator.

* `testOrchestrator:` references a partner containers that is used for the generic test suite.  The test partner is used
to send various types of traffic to each container under test.  For example the orchestrator is used to ping a container
under test, and to be the ping target of a container under test.

The [included default](test-network-function/tnf_config.yml) defines a single container to be tested,
and a single partner to do the testing.

### cnfs and operators

The `cnfs` and `operators` sections define the roles under which operators and containers are to be tested.

[The default config](test-network-function/tnf_config.yml) is set up with some examples of this:
It will run the `"OPERATOR_STATUS"` tests (as defined in `testconfigure.yml`) against an etcd operator, and the
`"PRIVILEGED_POD"` and `"PRIVILEGED_ROLE"` tests against an nginx container.

A more extensive example of these sections is provided in [example/example_config.yaml](example/example_config.yaml)

## Test Output

### Claim File

The test suite generates a "claim" file, which describes the system(s) under test, the tests that were run, and the
outcome of all of the tests.  This claim file is the proof of the test run that is evaluated by Red Hat when
"certified" status is being considered.  For more information about the contents of the claim file please see the
[schema](https://github.com/test-network-function/test-network-function-claim/blob/main/schemas/claim.schema.json).  You can
read more about the purpose of the claim file and CNF Certification in the
[Guide](https://redhat-connect.gitbook.io/openshift-badges/badges/cloud-native-network-functions-cnf).

### Adding Test Results for the CNF Validation Test Suite to a Claim File 
e.g. Adding a cnf platform test results to your existing claim file.

You can use the claim cli tool to append other related test suite results to your existing claim.json file.
The output of the tool will be an updated claim file.
```
go run cmd/tools/cmd/main.go claim-add --claimfile=claim.json --reportdir=/home/$USER/reports
```
 Args:  
`
--claimfile is an existing claim.json file`
`
--repordir :path to test results that you want to include.
`

 The tests result files from the given report dir will be appended under the result section of the claim file using file name as the key/value pair.
 The tool will ignore the test result, if the key name is already present under result section of the claim file.
```
 "results": {
 "cnf-certification-tests_junit": {
 "testsuite": {
 "-errors": "0",
 "-failures": "2",
 "-name": "CNF Certification Test Suite",
 "-tests": "14",
```

### Command Line Output

When run the CNF test suite will output a report to the terminal that is primarily useful for Developers to evaluate and
address problems.  This output is similar to many testing tools.

Here's an example of a Test pass.  It shows the Test running a command to extract the contents of `/etc/redhat-release`
and using a regular expression to match allowed strings.  It also prints out the string that matched.:

```shell
------------------------------
generic when test(test) is checked for Red Hat version 
  Should report a proper Red Hat version
  /Users/$USER/cnf-cert/test-network-function/test-network-function/generic/suite.go:149
2020/12/15 15:27:49 Sent: "if [ -e /etc/redhat-release ]; then cat /etc/redhat-release; else echo \"Unknown Base Image\"; fi\n"
2020/12/15 15:27:49 Match for RE: "(?m)Red Hat Enterprise Linux Server release (\\d+\\.\\d+) \\(\\w+\\)" found: ["Red Hat Enterprise Linux Server release 7.9 (Maipo)" "7.9"] Buffer: "Red Hat Enterprise Linux Server release 7.9 (Maipo)\n"
•
```

The following is the output from a Test failure.  In this case, the test is checking that a CSV (ClusterServiceVersion)
is installed correctly, but does not find it (the operator was not present on the cluster under test):

```shell
------------------------------
operator Runs test on operators when under test is: my-etcd/etcdoperator.v0.9.4  
  tests for: CSV_INSTALLED
  /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:122
2020/12/15 15:28:19 Sent: "oc get csv etcdoperator.v0.9.4 -n my-etcd -o json | jq -r '.status.phase'\n"

• Failure [10.002 seconds]
operator
/Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:58
  Runs test on operators
  /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:71
    when under test is: my-etcd/etcdoperator.v0.9.4 
    /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:121
      tests for: CSV_INSTALLED [It]
      /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:122

      Expected
          <int>: 0
      to equal
          <int>: 1

```

The following is the output from a Test failure.  In this case, the test is checking that a Subscription
is installed correctly, but does not find it (the operator was not present on the cluster under test):

```shell
------------------------------
operator Runs test on operators when under test is: my-etcd/etcd
  tests for: SUBSCRIPTION_INSTALLED
  /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:129
2021/04/09 12:37:10 Sent: "oc get subscription etcd -n my-etcd -ojson | jq -r '.spec.name'\n"

• Failure [10.000 seconds]
operator
/Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:55
  Runs test on operators
  /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:68
    when under test is: default/etcdoperator.v0.9.4 
    /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:128
      tests for: SUBSCRIPTION_INSTALLED [It]
      /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:129

      Expected
          <int>: 0
      to equal
          <int>: 1

```

The following is the output from a Test failure.  In this case, the test is checking clusterPermissions for
specific CSV, but does not find it (the operator was not present on the cluster under test):

```shell
------------------------------
operator Runs test on operators when under test is: my-etcd/etcdoperator.v0.9.4  
  tests for: CSV_SCC
  /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:129
2021/04/20 14:47:52 Sent: "oc get csv etcdoperator.v0.9.4 -n my-etcd -o json | jq -r 'if .spec.install.spec.clusterPermissions == null then null else . end | if . == null then \"EMPTY\" else .spec.install.spec.clusterPermissions[].rules[].resourceNames end'\n"

• Failure [10.001 seconds]
operator
/Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:55
  Runs test on operators
  /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:68
    when under test is: my-etcd/etcdoperator.v0.9.4 
    /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:128
      tests for: CSV_SCC [It]
      /Users/$USER/cnf-cert/test-network-function/test-network-function/operator/suite.go:129

      Expected
          <int>: 0
      to equal
          <int>: 1

```

## CNF Developers

Developers of CNFs, particularly those targeting 
[CNF Certification with Red Hat on OpenShift](https://redhat-connect.gitbook.io/openshift-badges/badges/cloud-native-network-functions-cnf),
can use this suite to test the interaction of their CNF with OpenShift.  If you are interested in CNF Certification
please contact [Red Hat](https://redhat-connect.gitbook.io/red-hat-partner-connect-general-guide/managing-your-account/getting-help/technology-partner-success-desk).

Refer to the rest of the documentation in this file to see how to install and run the tests as well as how to
interpret the results.

You will need an [OpenShift 4.4 installation](https://docs.openshift.com/container-platform/4.4/welcome/index.html)
running your CNF, and at least one other machine available to host the test suite.  The
[cnf-certification-test-partner](https://github.com/test-network-function/cnf-certification-test-partner) repository has a very
simple example of this you can model your setup on.

# Known Issues

## Issue #146:  Shell Output larger than 16KB requires specification of the TNF_DEFAULT_BUFFER_SIZE environment variable

When dealing with large output, you may occasionally overrun the default buffer size. The manifestation of this issue is
a `json.SyntaxError`, and may look similar to the following:

```shell script
    Expected
        <*json.SyntaxError | 0xc0002bc020>: {
            msg: "unexpected end of JSON input",
            Offset: 660,
        }
    to be nil
```

In such cases, you will need to set the TNF_DEFAULT_BUFFER_SIZE to a sufficient size (in bytes) to handle the expected
output.

For example:

```shell script
TNF_DEFAULT_BUFFER_SIZE=32768 ./run-cnf-suites.sh diagnostic generic
```

## Issue-161 Some containers under test do nto contain `ping` or `ip` binary utilities

In some cases, containers do not provide ping or ip binary utilities. Since these binaries are required for the
connectivity tests, we must exclude such containers from the connectivity test suite.  In order to exclude these
containers, please issue add the following to `test-network-function/generic_test_configuration.yaml`:

```yaml
excludeContainersFromConnectivityTests:
  - namespace: <namespace>
    podName: <podName>
    containerName: <containerName>
```

Note:  Future work may involve installing missing binary dependencies.
