---
id: k8s-next-helm-incubator-values
title: Incubator configurations
---

## Introduction

It turns out to be a very common case, that we can not get certain features done within just one release. Hence implementation spans multiple releases and some of the new implementation will be available in the release that we are shipping before the new feature was finalized.

We need to ensure a way to enable development to use these new features in development deployments and for testing and at the same time minimize efforts during endgame to make these unfinished features "invisible" for a release and we need to prevent documentation efforts for features that might be somewhat visible to customers, but which should actually not be used yet.

### Example case "central logging configuration"

We have been working on our central logging configuration for quite some time. At the time of writing, we are right before the endgame sprint of CF199 and we are in the situation that we now have to remove logging configuration options just in the release branch, so that customers don't see the future capabilities and we also don't have to document anything we would not even want to be documented yet.

## Solution proposal

To circumvent these kinds of situations, the proposal is to establish a so called "incubator" section inside our helm values.yaml file.
We'll add appropriate documentation so that it is obvious to our customers that configuration options within this section are solely for internal use yet.
As soon as we then finalize a feature, its configuration will move out of the incubator section, which will sort of represent the release of the feature, which would also then include appropriate end user documentation, too.

A nice side effect of the approach is that, we finally would have a good place to hold feature flag configuration e.g. for some beta features that we would only want to allow to use for certain customers.

This example `values.yaml` extract provides a proposal on how the configuration extension would look like:

```yaml
# Image related configuration
images:
...

# Resource allocation settings, definition per pod
# Use number + unit, e.g. 1500m for CPU or 1500M for Memory
resources:
...

# This is the incubator that contains internal and/or not yet published configuration pieces.
# Please only touch any of the incubator configurations in case our support teams ask you to.
# Feel free to have a look at new configuration items that might be released in future and make sure to ask any question you might have!
incubator:
  # Logging configuration
  logging:
    # Notice: log level settings are currently under development
    # Core specific logging configuration
    core:
...
```
