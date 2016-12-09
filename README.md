<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
# Overview

This bundle deploys a reference platform for building, testing, and releasing
Juju Charms and Bundles into the Juju Charm Store. The CI used in this bundle
is Jenkins, paired with the following charm-specific tools:

  * [Cloud Weather Report (CWR)][]
  * [Bundletester][]
  * [Matrix][]

[Cloud Weather Report (CWR)]: https://github.com/juju-solutions/cloud-weather-report
[Bundletester]: https://github.com/juju-solutions/bundletester
[Matrix]: https://github.com/juju-solutions/matrix

The cornerstone of this bundle is Cloud Weather Report (CWR). The `cwr` charm
handles CI requests (e.g. from from a webhook), manages the necessary models on
your controller(s), dispatches jobs to Jenkins, provides job status to the
requester, and can automatically release charms to your namespace in the charm
store.

> Note: We have a variation of this bundle called [cwr-rq][] that includes all
of the same components plus the Review Queue. If you are interested in a
charm/bundle CI system that includes a source review application, we
recommend you have a look at [cwr-rq][].

[cwr-rq]: https://github.com/juju-solutions/bundle-cwr-rq

## Bundle Composition
The charms that comprise this bundle are colocated on 1 machine. Additional
information about these charms can by found in the following linked READMEs:

  * [jenkins][]
  * [cwr][] (colocated on the jenkins machine)

[jenkins]: https://jujucharms.com/jenkins/xenial
[cwr]: https://jujucharms.com/u/kos.tsakalozos/cwr


# Getting Started

> Note: A bootstrapped Juju controller is required. If Juju is not yet set up,
please follow the [getting-started][] instructions prior to deploying this
bundle.

[getting-started]: https://jujucharms.com/docs/stable/getting-started

Deploy this bundle from the charm store:

    juju deploy cwr-ci

Charms in this bundle provide status messages to indicate their readiness.
Monitor the progress of the deployment with:

    watch juju status

> Note: Once the charms indicate they are ready, use `Ctrl-c` to terminate the
`watch` command and proceed with the following instructions.

Set a password for Jenkins:

    juju config jenkins password=<yourpassword>

CWR needs access to your controller(s) to create models and allocate resources
needed to run charm/bundle tests. Grant this by creating a user on
your bootstrapped controller(s) with appropriate permissions:

    juju add-user ciuser
    juju grant ciuser add-model

To register the controller with the CWR charm, you will need to call the
`register-controller` action and provide a human-friendly name and the
registration token from the above `juju add-user` command.

    juju run-action cwr/0 register-controller name=<controller-name> \
        token=<registration-token>

You should also setup a session with the charm store to allow CWR to release
charms to your namespace. To do this, call the `store-login` action and provide
the base64 representation of an existing auth token. For example:

    charm login
    .........
    export TOKEN=`base64 ~/.local/share/juju/store-usso-token`
    juju run-action cwr/0 store-login charmstore-usso-token="$TOKEN"

At this point, you have the foundation for a powerful charm/bundle CI system.
Workflows that leverage this system are described in the next section.


# Workflows

## Manage the Charm Release Cycle from Github

### Description
Our goal is to build and test a charm/bundle every time code is committed to a
repository. In light of a successful test, the resulting charm/bundle is pushed
to the store and released in the `edge` channel. Similarly, releases to the
`stable` channel can be made by tagging the code in Github when ready.

The rationale of this workflow is that you want charm/bundle updates released as
soon as you are confident that things are working as expected. With good tests,
the CI system can give you that confidence and automatically handle the release
process from a source repo to an `edge` or `stable` channel in the charm store.

### Prerequisites
  * A charm/top charm layer, e.g.: `awesome-charm`
  * Source repository on Github, e.g.: `http://github.com/myself/my-awesome-charm`
  * A charm store namespace, e.g.: `awesome-team`

### Procedure
To include `awesome-charm` in our CI pipeline, we need to call the
`build-on-commit` action:

    juju run-action cwr/0 build-on-commit \
        repo=http://github.com/myself/my-awesome-charm \
        charm-name=awesome-charm \
        push-to-channel=edge \
        lp-id=awesome-team \
        controller=lxd

This will instruct CWR to run a Jenkins job to test `awesome-charm` on your
`lxd` controller and release it to the **edge** channel each time you
**commit** to your repo.

For releasing `awesome-charm` to the stable channel, we need a similar call to
the `build-on-release` action:

    juju run-action cwr/0 build-on-release \
        repo=http://github.com/myself/my-awesome-charm \
        charm-name=awesome-charm \
        push-to-channel=stable \
        lp-id=awesome-team \
        controller=aws

This will instruct CWR to run a Jenkins job to test `awesome-charm` on your
`aws` controller and release it to the **stable** channel any time you
**tag** your source with a release tag.


# Summary

We have described an example workflow that can leverage the charm/bundle CI system
provided by this bundle. Do you have ideas or other workflows built around
CWR? Please let us know by contacting us on the mailing list below.

# Resources

- [Juju mailing list](https://lists.ubuntu.com/mailman/listinfo/juju)
- [Juju community](https://jujucharms.com/community)
