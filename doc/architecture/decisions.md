# Design Decisions

This documentation collects reasoning and documentation regard decisions made
regarding the design of the Helm charts in this repository.

## Attempt to catch problematic configurations

Due to the complexity of these charts and their level of flexibility, there are some
overlaps where it is possible to produce a configuration that would lead to an
unpredictable, or entirely non-functional deployment. In an effort
to prevent known problematic settings combinations, we have implemented template logic
designed to detect and warn the user that their configuration will not work.

This replicates the behavior of deprecations, but is specific to ensuring functional configuration.

Introduced in [!757 checkConfig: add methods to test for known errors](https://gitlab.com/gitlab-org/charts/gitlab/-/merge_requests/757)

## Breaking changes via deprecation

During the development of these charts, we occasionally make improvements that require
alterations to the properties of existing deployments. Two examples were the centralization
of configuring the use of MinIO, and the migration of external object storage configuration
from properties to secrets (in observance of our preference).

As a means of preventing a user from accidentally deploying an updated version of these
charts which includes a breaking change against a configuration that would not function, we
have chosen to implement [deprecation](../development/index.md#handling-configuration-deprecations) notifications. These are designed to detect
properties have have been relocated, altered, replaced, or removed entirely, then inform
the user of what changes need to be made to the configuration. This may include informing
the user to see documentation on how to replace a property with a secret. These notifications
will cause the Helm `install` or `upgrade` commands to stop with a parse error, and output a complete list of items that need to be addressed. We have taken care to ensure a user will not be placed into a loop of error, fix, repeat.

All deprecations must be addressed in order for a successful deployment to occur. We believe
the user would prefer to be informed of a breaking change over experiencing unexpected
behavior or complete failure that requires debugging.

Introduced in [!396 Deprecations: implement buffered list of deprecations](https://gitlab.com/gitlab-org/charts/gitlab/-/merge_requests/396)

## Preference of Secrets in initContainer over Environment

Much of the container ecosystem has, or expects, the capability to be configured
through environment variables. This [configuration practice](https://12factor.net/config)
stems from the concept of [The Twelve-Factor App](https://12factor.net). This
greatly simplifies configuration across multiple deployment environments, but there
remains a security concern with passing connection secrets such as passwords and
private keys via the container's environment.

Most container ecosystems provide a simple method to inspect the state of a running
container, which usually includes the environment. Using [Docker](https://www.docker.com/)
as an example, any process capable of communicating with the daemon can query the
state of all running containers. This means that if you have a privileged container
such as [dind][], that container can then inspect the environment of _any_ container
on a given node, and expose _all_ secrets contained within.
As a part of the [complete DevOps lifecycle][devops-post], [dind][] is regularly
used for building containers that will be pushed to a registry and subsequently
deployed.

This concern is why we've decided to prefer the population of sensitive information
via [initContainers][].

Related issues:

- [#90](https://gitlab.com/gitlab-org/charts/gitlab/issues/90)
- [#114](https://gitlab.com/gitlab-org/charts/gitlab/issues/114)

[dind]: https://hub.docker.com/r/gitlab/dind/
[devops-post]: https://about.gitlab.com/blog/2017/10/11/from-dev-to-devops/
[initContainers]: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

## Sub-charts are deployed from global chart

All sub-charts of this repository are designed to be deployed via the global chart.
Each component can still be deployed individually, but make use of a common set of
properties facilitated by the global chart.

This decision simplifies both the use and maintenance of the repository as a whole.

Related issue:

- [#352](https://gitlab.com/gitlab-org/charts/gitlab/issues/352)

## Template partials for `gitlab/*` should be global whenever possible

All template partials of the `gitlab/*` sub-charts should be a part of the global or
GitLab sub-chart `templates/_helpers.tpl` whenever possible. Templates from
[forked charts](#forked-charts) will remain a part of those charts. This reduces
the maintenance impact of these forks.

The benefits of this are straight-forward:

- Increased DRY behavior, leading to easier maintenance. There should be no reason
  to have duplicates of the same function across multiple sub-charts when a single
  entry will suffice.
- Reduction of template naming conflicts. All [partials throughout a chart are compiled together][helm-dev-doc],
  and thus we can treat them like the global behavior they are.

Related issue:

- [#352](https://gitlab.com/gitlab-org/charts/gitlab/issues/352)

[helm-dev-doc]: https://helm.sh/docs/chart_template_guide/named_templates/#declaring-and-using-templates-with-define-and-template

## Forked charts

The following charts have been forked or re-created in this repository following
our [guidelines for forking](../development/index.md#guidelines-for-forking)

### Redis

With the `3.0` release of the GitLab Helm chart, we no longer fork the [upstream Redis chart](https://github.com/helm/charts/tree/master/stable/redis),
and instead include it as a dependency.

### Redis HA

Redis-HA was a chart we included in our releases prior to `3.0`. It has now been removed,
and replaced with [upstream Redis chart](https://github.com/helm/charts/tree/master/stable/redis)
which has added optional HA support.

### MinIO

Our [MinIO chart](../charts/minio/index.md) was altered from the upstream [MinIO](https://github.com/helm/charts/tree/master/stable/minio).

- Make use of pre-existing Kubernetes secrets instead of creating new ones from properties.
- Remove providing the sensitive keys via Environment.
- Automate the creation of multiple buckets via `defaultBuckets` in place of
  `defaultBucket.*` properties.

### registry

Our [registry chart](../charts/registry/index.md) was altered from the upstream [docker-registry](https://github.com/helm/charts/tree/master/stable/docker-registry).

- Enable the use of in-chart MinIO services automatically.
- Automatically hook authentication to the GitLab services.

### NGINX Ingress

Our [NGINX Ingress chart](../charts/nginx/index.md) was altered from the upstream [NGINX Ingress](https://github.com/helm/charts/tree/master/stable/nginx-ingress).

- Add feature to allow for the tcp configmap to be external to the chart
- Add feature to allow Ingress class to be templated based on release name