<!-- markdownlint-disable-next-line -->
# <img src="https://opentelemetry.io/img/logos/opentelemetry-logo-nav.png" alt="OTel logo" width="45"> OpenTelemetry Demo

[![Slack](https://img.shields.io/badge/slack-@cncf/otel/demo-brightgreen.svg?logo=slack)](https://cloud-native.slack.com/archives/C03B4CWV4DA)
[![Version](https://img.shields.io/github/v/release/open-telemetry/opentelemetry-demo?color=blueviolet)](https://github.com/open-telemetry/opentelemetry-demo/releases)
[![Commits](https://img.shields.io/github/commits-since/open-telemetry/opentelemetry-demo/latest?color=ff69b4&include_prereleases)](https://github.com/open-telemetry/opentelemetry-demo/graphs/commit-activity)
[![Downloads](https://img.shields.io/docker/pulls/otel/demo)](https://hub.docker.com/r/otel/demo)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg?color=red)](https://github.com/open-telemetry/opentelemetry-demo/blob/main/LICENSE)
[![Telemetry Tests](https://github.com/open-telemetry/opentelemetry-demo/actions/workflows/run-telemetry-tests.yml/badge.svg)](https://github.com/open-telemetry/opentelemetry-demo/actions/workflows/run-telemetry-tests.yml)
[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/opentelemetry-demo)](https://artifacthub.io/packages/helm/opentelemetry-helm/opentelemetry-demo)
[![FOSSA Status](https://app.fossa.com/api/projects/custom%2B162%2Fgithub.com%2Fopen-telemetry%2Fopentelemetry-demo.svg?type=shield&issueType=license)](https://app.fossa.com/projects/custom%2B162%2Fgithub.com%2Fopen-telemetry%2Fopentelemetry-demo?ref=badge_shield&issueType=license)
[![FOSSA Status](https://app.fossa.com/api/projects/custom%2B162%2Fgithub.com%2Fopen-telemetry%2Fopentelemetry-demo.svg?type=shield&issueType=security)](https://app.fossa.com/projects/custom%2B162%2Fgithub.com%2Fopen-telemetry%2Fopentelemetry-demo?ref=badge_shield&issueType=security)
[![OpenSSF Scorecard for opentelemetry-demo](https://api.scorecard.dev/projects/github.com/open-telemetry/opentelemetry-demo/badge)](https://scorecard.dev/viewer/?uri=github.com/open-telemetry/opentelemetry-demo)
[![OpenSSF Best Practices](https://www.bestpractices.dev/projects/9247/badge)](https://www.bestpractices.dev/en/projects/9247)

## Welcome to the OpenTelemetry Astronomy Shop Demo

This repository contains the OpenTelemetry Astronomy Shop, a microservice-based
distributed system intended to illustrate the implementation of OpenTelemetry in
a near real-world environment.

Our goals are threefold:

- Provide a realistic example of a distributed system that can be used to
  demonstrate OpenTelemetry instrumentation and observability.
- Build a base for vendors, tooling authors, and others to extend and
  demonstrate their OpenTelemetry integrations.
- Create a living example for OpenTelemetry contributors to use for testing new
  versions of the API, SDK, and other components or enhancements.

We've already made [huge
progress](https://github.com/open-telemetry/opentelemetry-demo/blob/main/CHANGELOG.md),
and development is ongoing. We hope to represent the full feature set of
OpenTelemetry across its languages in the future.

If you'd like to help (**which we would love**), check out our [contributing
guidance](./CONTRIBUTING.md).

If you'd like to extend this demo or maintain a fork of it, read our
[fork guidance](https://opentelemetry.io/docs/demo/forking/).

## Quick start

You can be up and running with the demo in a few minutes. Check out the docs for
your preferred deployment method:

- [Docker](https://opentelemetry.io/docs/demo/docker_deployment/)
- [Kubernetes](https://opentelemetry.io/docs/demo/kubernetes_deployment/)

## Documentation

For detailed documentation, see [Demo Documentation][docs]. If you're curious
about a specific feature, the [docs landing page][docs] can point you in the
right direction.

## Demos featuring the Astronomy Shop

We welcome any vendor to fork the project to demonstrate their services and
adding a link below. The community is committed to maintaining the project and
keeping it up to date for you.

|                           |                |                                  |
|---------------------------|----------------|----------------------------------|
| [AlibabaCloud LogService] | [Google Cloud] | [Rootprint]                      |
| [Amazon Web Services]     | [Grafana Labs] | [Sentry]                         |
| [Apache Doris]            | [Guance]       | [ServiceNow Cloud Observability] |
| [AppDynamics]             | [Honeycomb.io] | [SigNoz]                         |
| [Aspecto]                 | [Instana]      | [SolarWinds Observability]       |
| [Axiom]                   | [Kloudfuse]    | [Splunk]                         |
| [Axoflow]                 | [Kopai]        | [Sumo Logic]                     |
| [Azure Data Explorer]     | [Last9]        | [Tayga]                          |
| [Bronto]                  | [Liatrio]      | [TelemetryHub]                   |
| [Causely]                 | [Logz.io]      | [Teletrace]                      |
| [ClickStack]              | [New Relic]    | [Tinybird]                       |
| [Coralogix]               | [Oodle]        | [Trace0]                         |
| [Dash0]                   | [OpenObserve]  | [Tracetest]                      |
| [Datadog]                 | [OpenSearch]   | [Tsuga]                          |
| [Dynatrace]               | [Oracle]       | [Uptrace]                        |
| [Elastic]                 | [Parseable]    | [VictoriaMetrics]                |

## Contributing

To get involved with the project see our [CONTRIBUTING](CONTRIBUTING.md)
documentation. Our [SIG Calls](CONTRIBUTING.md#join-a-sig-call) are every other
Wednesday at 8:30 AM PST and anyone is welcome.

New to CNCF Slack? [Create an account](https://slack.cncf.io/) first, then join
us in
[`#otel-community-demo`](https://cloud-native.slack.com/archives/C03B4CWV4DA).

### Maintainers

- [Cyrille Le Clerc](https://github.com/cyrille-leclerc), Datadog
- [Juliano Costa](https://github.com/julianocosta89), Datadog
- [Shenoy Pratik](https://github.com/ps48), AWS OpenSearch

For more information about the maintainer role, see the [community repository](https://github.com/open-telemetry/community/blob/main/guides/contributor/membership.md#maintainer).

### Approvers

- [D&#xF3;nal O'Sullivan](https://github.com/osullivandonal), Elastic
- [Piotr Kie&#x142;kowicz](https://github.com/Kielek), Splunk

For more information about the approver role, see the [community repository](https://github.com/open-telemetry/community/blob/main/guides/contributor/membership.md#approver).

### Emeritus

- [Austin Parker](https://github.com/austinlparker), Maintainer
- [Carter Socha](https://github.com/cartersocha), Maintainer
- [Cedric Ziel](https://github.com/cedricziel), Approver
- [Michael Maxwell](https://github.com/mic-max), Approver
- [Mikko Viitanen](https://github.com/mviitane), Maintainer
- [Morgan McLean](https://github.com/mtwo), Approver
- [Penghan Wang](https://github.com/wph95), Approver
- [Pierre Tessier](https://github.com/puckpuck), Maintainer
- [Reiley Yang](https://github.com/reyang), Approver
- [Roger Coll](https://github.com/rogercoll), Maintainer
- [Ziqi Zhao](https://github.com/fatsheep9146), Approver

For more information about the emeritus role, see the [community repository](https://github.com/open-telemetry/community/blob/main/guides/contributor/membership.md#emeritus-maintainerapprovertriager).

### Thanks to all the people who have contributed

[![contributors](https://contributors-img.web.app/image?repo=open-telemetry/opentelemetry-demo)](https://github.com/open-telemetry/opentelemetry-demo/graphs/contributors)

[docs]: https://opentelemetry.io/docs/demo/

