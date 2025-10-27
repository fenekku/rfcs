- Start Date: 2025-10-24
- RFC PR: [#109](https://github.com/inveniosoftware/rfcs/pull/109)
- Authors: Guillaume Viger

# Python support in Invenio/RDM ecosystem

## Summary

This RFC outlines the Python support policy for InvenioRDM and the related Invenio ecosystem. In particular, it outlines how the policy relates to concrete implementations/use-cases/actions. Some aspects are left flexible for agile decision making in future contexts.

The general idea is:

|                                 | Minimum Python version required               | Maximum Python version supported    |
| ------------------------------- | --------------------------------------------- | ----------------------------------- |
| Invenio framework level modules | Keep current minimum                          | Latest version officially supported |
| InvenioRDM specific modules     | Decided miminum for that InvenioRDM's version | Latest version officially supported |


## Motivation

As an Invenio(RDM) developer (contributor or maintainer), I want to know:

- what Python features I can use or not, so that I am effective at my work.
- what Python deprecations must be supported via an adaptive layer, so that Invenio(RDM) can be stable across Python versions.
- which deprecations in these adaptive layers can be dropped completely and when, in order to keep the maintenance burden low.
- what `requires-python =` (`pyproject.toml`) or `[requires] python_version =` (`Pipfile`) or `python_requires =` (`setup.cfg`) constraints to apply to which module/application, in order to enforce the policy.
- what versions of Python should be used for the continuous integration tests, in order to test well but be efficient with test runners.
- how to deal with yearly release cadence of the Python language, in order to stay relevant (secure/performant).

More than just describing what the policy is, this RFC will try to outline how it is manifested.


## Detailed design

After maintainer meetings and online discussions, the following approach was outlined. The problem space was divided at a high-level between:
InvenioRDM v14 onwards, InvenioRDM periods of overlap with deprecated Python versions, and the wider Invenio framework.

### InvenioRDM — v14 and onwards

Starting with InvenioRDM v14, the idea is to set the minimum required Python version to a specific version and keep it so until the InvenioRDM release of the year where that version is outdated. This version will be tested against directly to ensure backwards compatibility over time. The maximum officially supported Python version will be the latest released version of Python at time of release of InvenioRDM. This applies to RDM specific modules and RDM instances.

**Aside** InvenioRDM and Python are on different release cadences with new Python versions typically releasing annually early October while InvenioRDM releases annually late July.

**Example for InvenioRDM v14**

For InvenioRDM v14, the target minimum version is Python 3.14 (it's nice that both have 14 in the version number). Here is how Python support would look like pre and post release of that version:

| Timeline | Context                                                     | Min. Python version required | Max. Python version supported |
| -------- | ----------------------------------------------------------- | ---------------------------- | ----------------------------- |
| 2025-10  | post Python 3.14 release — pre RDMv14 release — development | 3.14                         | 3.14                          |
| 2026-08  | RDMv14 release — users + development                        | 3.14                         | 3.14                          |
| 2026-10  | post Python 3.15 release — pre RDMv15 release — development | 3.14                         | 3.15                          |
| 2026-10  | post Python 3.15 release — pre RDMv15 release — users       | 3.14                         | 3.14                          |
| 2027-08  | RDMv15 release — users + development                        | 3.14                         | 3.15                          |
| 2027-10  | post Python 3.16 release - pre RDMv16 release — development | 3.14                         | 3.16                          |
| ...      | ...                                                         | 3.14                         | ...                           |
| 2030-10  | post Python 3.19 release — pre RDMv19 release — development | Re-assess, 3.19?             | ...                           |

Although a version of InvenioRDM could un-officially support a newly released version of Python, the official support is only guaranteed
when a dedicated release is made that officially supports it.

**CI**
The idea would be for the CI to test the minimum version required and the maximum supported. Skip in-between versions as they would have been tested in the past and this keeps the CI efficient in aggregate *and* per individual PR.

**Reasoning**
- Jumping to 3.14 as minimum version for RDMv14 allows us to benefit from the longest support window
- Alignment of version numbers is nice
- Take advantage of performance gains as a baseline
- Don't have to change minimum version every year and constantly churn
- Be alerted and account for upcoming deprecations in development as soon as official next latest Python version is out

### InvenioRDM — periods of overlap with deprecated Python versions

The Python support policy outlined above leads to a scenario where a newly released InvenioRDM and the previous, but still supported for 6 months per [RDM support policy](https://inveniordm.docs.cern.ch/releases/maintenance-policy/), do not share the same mimimum required Python version for that overlap period. This could be a rare occurrence (e.g., every 5 years), but it's not ideal either.

As is, it would mean that during that period of time, any fix would have to be compatible with the previous minimum Python version — a different and unsupported (by Python community) one.

[Open speculative]
I see two potential approaches. In both cases though a Docker image of InvenioRDM with a still supported Python version should be available.

First option: continue for 6 months to make fixes and backports that are compatible with the previous minimum Python version. This means being more careful about fixes and it doesn't promote adoption of a secure/supported version of Python. It is simpler because it doesn't require anything else. Past the 6 months mark, all the new features of the new minimum Python version can be adopted (and all deprecation workarounds that can be dropped are).

Second option: switch to the new smallest supported Python version. It would mean changing the minimum bound of the CI, but also being able to adopt some new features faster. It would also mean releasing a new InvenioRDM version in the prior major series that requires this new minimum (patch or minor is debatable). It would encourage users to switch to a secure version if they want patches.


Third option: explicitly drop the 6 months overlap support for that prior InvenioRDM version. This would only be considered if we have been respectful of users' trust and alerted them of that shorter support window at the time of the release of the prior InvenioRDM version. This wouldn't be a possible solution for v13 -> v14 for instance.
[End speculative]

The current (at time of writing) scenario of RDMv13 is illustrative of this "periods of overlap with deprecated Python versions" section. Note however that Python 3.10 would have been the next smallest
version, but it was never officially supported by InvenioRDM, so 3.11 would probably be the next minimum supported version (per the second option).

### Invenio framework modules

Invenio-wide modules like invenio-app,-search... (as opposed to InvenioRDM-specific ones like invenio-records-resources,-rdm-records ...) are not driven by the InvenioRDM cadence and should be even more stable. They should keep supporting the minimum Python version they did and be independent of RDM.

[Mine]
It seems strange to keep them supporting versions of Python that are deprecated however. I could see us moving that minimum in tandem with officially supported version or with InvenioRDM cadence actually. Would simplify development concerns and encourage users to switch to newer Python versions to keep receiving patches/new features.

It also means that those modules are the ones that will "cost" us most in terms of maintenance:
- they cannot benefit from greater mimimum as RDM modules
- they require adaptive code to support their now deprecated parts
- If minimum changed every year, that is very frequent upkeep
[end mine]

### Tested ranges

As laid out, it becomes apparent that a single shared reusable GitHub Action test workflow will not be able to cover all modules. A framework module with a minimum Python version of 3.8 would either not be tested by such a workflow or the addition of 3.8 to the workflow's tested versions would be incompatible to an InvenioRDM module that requires at least Python 3.14. Add to the mix the above note with respect to CI bounds in the context of periods of overlap with deprecated Python versions and the need for different test ranges (potentially workflows) becomes apparent.

To be efficient with CI resources, the minimum and maximum versions should be explicitly tested with in-between versions skipped.

Options:

**Same workflow, per repo ranges**. The same test workflow is adopted by all repositories, but each repository defines their own Python version range. This has a lot of range repetition, but otherwise keeps things simple.

**One test workflow for Invenio modules and one for InvenioRDM modules (both versioned)**. Each of these test workflows defines its default range and therefore addresses repetition and enforces uniformity. To account for periods of overlap with deprecated Python versions, new versions of these workflows would exist with the appropriate range. Specific version of workflows would be used. This adds some complexity to the approach.


## How we teach this

We can document the gist of it in the docs in the [Maintenance Policy](https://inveniordm.docs.cern.ch/releases/maintenance-policy/) section.

By having the appropriate workflows, this will also be conveyed to developers.

## Drawbacks

This lays out long support times for some Python versions (up to 5 year?) which means accounting for deprecations for a long time and sometimes not using helpful newer features. But we get stability and working within our available development resources in exchange.

Other drawbacks were inlined.

## Alternatives

Alternaives were inlined.

## Unresolved questions

Unresolved questions were inlined.

## Resources/Timeline

This should be adopted as soon as possible since minimum Python version of InvenioRDM v13 is Python 3.9 which goes out of support at end of October.
