# Concepts

## Brownout Strategies

At its core, Kubedim provides the ability to dim optional components in response
to load with minimal configuration, called **baseline dimming**. However,
Kubedim comes with two strategies which allow for greater control over how users
and components are dimmed, allowing Kubedim to meet business objectives such as
conversions and user experience. These strategies require additional
configuration.

### Baseline Dimming

This strategy dims all optional components uniformly.

### Component Weightings

This strategy allows you to dim components non-uniformly. By focusing dimming
on components which add the most to system load when they are made available,
optional components which contribute less to load should not need to be dimmed
as heavily, making your optional components more available to your users.

This strategy works by configuring each dimmable component to have a weighting
between 0 and 1. Components which contribute most load to a
system have a large value assigned. These weightings represent the probability
that an optional component will be dimmed, given that a dimming decision has
been made.

Where weightings are less obvious, perhaps due to interactions between
components as their availabilities change, we provide a training tool for use on
a non-production server with a realistic workload.

### Profiling

This strategy allows you to ensure that only users who are of lower priority to
your application are dimmed first. You declaratively specify rules on how an
application should be dimmed, then Kubedim's profiler will automatically assign
priorities to users based on these rules. Where users have not been profiled,
they are dimmed according to Kubedim's standard dimming strategy.
