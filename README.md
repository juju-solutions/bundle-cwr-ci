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

    watch -c juju status --color

> Note: Once the charms indicate they are ready, use `Ctrl-c` to terminate the
`watch` command and proceed with the following instructions.

Set a password for Jenkins:

    juju config jenkins password=<yourpassword>

CWR needs access to your controller(s) to create models and allocate resources
needed to run charm/bundle tests. Grant this by creating a user on
your bootstrapped controller(s) with appropriate permissions:

    juju add-user ciuser
    juju grant ciuser add-model

Now register your controller with CWR by calling the `register-controller`
action. Provide the controller name and the registration token from the above
`juju add-user` command:

    juju run-action cwr/0 register-controller name=<controller-name> \
        token=<registration-token>

If you're using a cloud that requires credentials (i.e., anything other than
the LXD provider), you will need to provide those credentials as base64-encoded
YAML. You can find your credentials in `~/.local/share/juju/credentials.yaml`,
but you may want to extract and share just the portions that will be
used with the CI system.  In the future, Juju should provide a way to
share access to the credentials without having to share the credentials
themselves. Until then, inform CWR of your controller credentials with the
`set-credentials` action:

    juju run-action cwr/0 set-credentials cloud=<cloud-name> \
        credentials="$(base64 credentials.yaml)"

Finally, you may setup a session with the charm store that allows CWR to
release charms to your namespace. To do this, call the `store-login` action
and provide the base64 representation of an existing auth token:

    charm login
    .........
    juju run-action cwr/0 store-login \
        charmstore-usso-token="$(base64 ~/.local/share/juju/store-usso-token)"

The charm store session will remain active while the CI system is online. To
terminate the session, run the `store-logout` action:

    juju run-action cwr/0 store-logout

At this point, you have the foundation for a powerful charm/bundle CI system.
Workflows that leverage this system are described in the next section.


# Workflows

## CI Repository Changes

### Description
The goal is to build and test a charm/bundle every time code is committed or
tagged in a repository. In light of a successful test, the resulting
charm/bundle is pushed to the store and released in the specified channel.

The rationale of this workflow is that you want charm/bundle updates released
to appropriate channels as soon as you are confident that things are working as
expected. With good tests, this CI system can give you that confidence and
automatically handle the release process from a source repo to a channel in the
charm store.

### Prerequisites
  * A charm/top charm layer, e.g.: `awesome-charm`
  * Source repository, e.g.: `http://github.com/myself/my-awesome-charm`
  * A reference bundle for exercising the charm to be tested

### Procedure
To include `awesome-charm` in a **commit**-based CI pipeline, call the
`build-on-commit` action:

    juju run-action cwr/0 build-on-commit \
        repo=http://github.com/myself/my-awesome-charm \
        charm-name=awesome-charm \
        reference-bundle=~awesome-team/awesome-bundle

To include `awesome-charm` in a **tag**-based CI pipeline, call the
`build-on-release` action:

    juju run-action cwr/0 build-on-release \
        repo=http://github.com/myself/my-awesome-charm \
        charm-name=awesome-charm \
        reference-bundle=~awesome-team/awesome-bundle


By default, these actions will output a url that must be set as a hook in your
repository (*webhooks* in Github parlance).

Both of these actions will instruct CWR to setup a Jenkins job that will test
`awesome-charm` on all registered controllers. This works by cloning the
given `repo`, building a local copy of `charm-name`, deploying the given
`reference-bundle` with the locally built charm, and executing the charm/bundle
tests.

Both actions support the following parameters:

- `repo`: The url of the charm source repository.
- `charm-name`: The name of the charm to test.
- `reference-bundle`: Charm store location of a bundle to use to test the
  given charm (e.g.: `~awesome-team/awesome-bundle`).
  If not specified, the charm must specify a `reference-bundle` in its
  `tests.yaml`.
- `branch` (optional): The repository branch to extract when testing this
  charm.
- `controller` (optional): Name of the registered controller to use for tests.
  If not specified, tests will run on all registered controllers.
- `repo-access` (optional): `webhook` or `poll`. By default, the action will
  produce a URL to be used as a hook that will trigger the build on commit/tag.
  Set this to `poll` to periodically (every 5 minutes) poll the repository
  for changes.
- `push-to-channel` (optional): Charm Store channel (e.g.: edge, beta,
  candidate, stable) for releasing the charm if tests succeed.
- `lp-id` (optional): The launchpad ID/namespace you want the charm released
  under (e.g.: `~awesome-team`).

## CI Charm Store Changes

### Description
The goal is to test a bundle every time an included charm is updated in the
Charm Store. In light of a successful test, the resulting bundle is pushed to
the store and released in the specified channel.

The rationale of this workflow is that you want your bundle updated with the
most recent charms from the store, and you want to release a new bundle to the
appropriate channel as soon as you are confident that things are working as
expected.

### Prerequisites
  * A bundle, e.g.: `awesome-bundle`, with `ci-info.yaml` (details below)
  * Source repository, e.g.: `http://github.com/myself/my-awesome-bundle`

Example `ci-info.yaml`:

```
bundle:
  name: awesome-bundle
  namespace: awesome-team
  release: true
  to-channel: beta
charm-upgrade:
  awesome-charm:
    from-channel: edge
    release: true
    to-channel: beta
```

Under the `bundle` key, set the bundle `name`. If you wish to release the
bundle upon a successful update and test cycle, set `release` to true and
provide the desired `namespace` and `to-channel` where you want to release
the bundle.

Under the `charm-upgrade` key, specify the charm(s) you want the CI system to
monitor. For each charm, specify the channel to watch for new revisions
(`from-channel`) and optionally the channel to release any upgraded charms
(`to-channel`).

### Procedure
To include `awesome-bundle` in a **charm store**-based CI pipeline, call the
`build-bundle` action:

    juju run-action cwr/0 build-bundle \
      repo=http://github.com/myself/my-awesome-bundle \
      bundle-name=awesome-bundle

This action will create a Jenkins job that will clone your bundle source
repository, inspect the `bundle.yaml`, and determine if there are any charms
that can be updated. If an update is possible, a local `bundle.yaml` will be
created with updated charm revisions and the bundle tests will be executed. If
the tests are successful, this job can also release the bundle and updated
charms to the specified channel in the store. This job will run periodically
(every 10 minutes).

This action supports the following parameters:

  - `repo`: The url of the bundle source repository.
  - `bundle-name`: The name of the bundle to test.
  - `branch` (optional): The repository branch to extract when testing this
    bundle.
  - `controller` (optional): Name of the registered controller to use for tests.
    If not specified, tests will run on all registered controllers.




# Summary

We have described two example workflows that can leverage the charm/bundle CI
system provided by this bundle. Do you have ideas or other workflows related
to charm/bundle CI? Please let us know by contacting us on the mailing list
below.

# Resources

- [Juju mailing list](https://lists.ubuntu.com/mailman/listinfo/juju)
- [Juju community](https://jujucharms.com/community)
