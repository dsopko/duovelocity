# DuoVelocity

Turns nightly Duolingo snapshots into velocity: lessons per day, units per week and per month, and Score over time. Duolingo keeps only ~7 days of lesson history and stamps no dates on the learning path, so DuoVelocity observes daily and remembers.

**Status:** design phase, no code yet.

- [Design doc](docs/duovelocity-design.md)

Planned stack: .NET 10 (Core library, CLI, Azure Functions nightly sync, ASP.NET Core Web API) on Azure free tiers.
