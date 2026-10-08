+++
date = '2026-10-08'
title = 'Allow-list Architecture'
tags = [  # pragma: alphabetize
  "python",
  "software architecture",
]
+++

Every significant software project I've worked on has,
at some point,
developed rules about its architecture.
These have usually been expressed in the form of restrictions:
this component cannot depend on that component,
a certain dependency cannot be used in some part of the project,
*etc*.
Linting tools work well in this paradigm:
if you can express what isn't allowed,
you can write a tool to identify cases of that
and prevent them from happening.
And this works,
up to a point.

This is architecture defined by rules about what isn't allowed.
By specifying all the things we don't want,
we're left to infer to correct architecture
from the negative space that's left.

When we first define these rules,
everything might be working fine.
We can carefully constrain things
so that the only remaining options are what you want.
But software is *soft*.
It changes
and with those changes, new negative space opens up.
Quite often, this means there are new and exciting ways
to build software with architecture we don't want.
So we must remain ever-vigilant,
noticing where there are opportunities for problems
and plugging the gaps before anything gets built there.

Quite frankly,
that's exhausting.

And not just for the maintainers that want to control the architecture.
For anyone joining the project,
getting a clear picture of what the architecture should be
is an exercise in imagination.
They have to read the rules
and visualise that negative space
all by themselves.

What I've described is "block-list" architecture.
Everything is allowed, except for those things that aren't.

## Allow-list Architecture

The alternative,
logically,
is "allow-list" architecture.
What would it look like to explicitly state what the architecture *should* be?

Many projects do this with documentation:
architecture diagrams,
decision records,
descriptions of each component and where it fits into the whole.
But this documentation only matters if it's followed;
the mechanism that enforces the documentation is often still a block-list.

In a few projects recently,
my colleagues and I have been experimenting with an allow-list.
We ban all dependencies between modules in our project
and then exempt only the relationships we want to keep.
The result is an explicit list of which dependencies we want.
If, as inevitably happens, we want to introduce a new connection,
we have to add it to the list and -- by our own convention -- explain it.

This means we have made a positive decision
about each dependency within our project.
It also means we can't sleepwalk into complexity by mistake.
Our tooling doesn't let us get away with accidents.
More than once,
we've made one component depend on another
in a way that hasn't already been allowed
and our tooling has called us out.
That's been a prompt to re-think our approach
and it's saved us from mistakes more than once.

## `import-linter` for Python

I mostly work on Python projects,
and our tool of choice is [import-linter],
built by my former colleague [David Seddon].

Import linter allows us to define *contracts*
that govern what Python modules can import.
For our allow-list,
we use the built-in "independence" contract type:
we want all of the modules within our application to be independent,
unless we have exempted them from the restriction.

An example configuration might look like:

```ini
[importlinter]
root_package = my_project

[importlintter:contract:architecture]
name = Allow-list Architecture
type = independence
modules =
    my_project.*
ignore_imports =
    # Interfaces know which commands they call.
    my_project.interfaces.** -> my_project.commands

    # Everything is allowed to use the domain.
    my_project.** -> my_project.domain

    # Everything is allowed to use common utilities.
    my_project.** -> my_project.logging
    my_project.** -> my_project.typing
```

This contract gets checked as part of our CI pipeline.
It's so fast that we even run it locally before every test run
as part of our TDD loop.

## Taking things further: external dependencies

Our allow-list architecture lets us describe the internal structure of our project.
But we have several external dependencies.
Usually, we want to constrain those to be accessed through a façade.
We started out by using import-linter's built-in "forbidden" contract type
to block-list certain 3rd-party dependencies,
except for where we explicitly allow them.
I'm sure you can see the problem here!

We recently added a new dependency
and we forgot to add it to the block-list!
Very quickly, we had depended on it directly
in places that we shouldn't have.
The block-list is failing us once again
by not being able to account for how our software changes.

So, we now take a different approach.
We allow-list dependencies on external packages.
Every import is banned
until we explicitly allow it.
This has brought to light multiple cases of dependency-leakage.

We started to enforce this using the "forbidden" contract type
and specifying `*` as our forbidden modules.
That sort of worked, but it also picked up standard library imports as forbidden.
Allow-listing those was frustrating
and distracted from what we were trying to document with the contract.

So I've written my own contract type:
[import-linter-forbid-external-packages].
This works like the built-in "forbidden" contract,
but only forbids modules that are not in the standard library.
This gives us a contract that looks something like:

```ini
[importlinter]
root_package = my_project
include_external_packages = true
contract_types =
    forbid_external_packages: importlinter_forbid_external_packages.Contract

[importlinter:contract:external-packages]
name = Allow-list External Packages
type = forbid_external_packages
source_modules =
    my_project.*
ignore_imports =
    # Everything is allowed to use `attrs` to construct objects.
    my_project.** -> attrs

    # Integrations wrap external dependencies.
    my_project.integrations.opentelemetry -> opentelemetry
    my_project.integrations.sentry -> sentry
    my_project.integrations.structlog -> structlog
```

We now have an explicit list of dependencies in our project.
When we add a new module
-- either inside our own code or by depending on a 3rd-party library --
we know *exactly* which parts of our code will couple to it.
That makes us mindful about where we couple to our dependencies
and ensures every architectural decision is deliberate.

[David Seddon]: https://seddonym.me
[import-linter-forbid-external-packages]: https://pypi.org/project/import-linter-forbid-external-packages/
[import-linter]: https://import-linter.readthedocs.io/en/stable/
