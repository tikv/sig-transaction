# Summary

- [Intro](README.md)

# Documentation

- [Transactions in TiKV](doc/tikv/README.md)
- [Transactions handling newbiew perspective](doc/tikv/transaction-handling-newbie-perspective/transaction-handling-newbie-perspective.md)

# Design docs

- [Proposal for refactoring how transactional commands are scheduled and implemented](design/transaction-layer-refactoring.md)
- [Async commit](design/async-commit/README.md)
    - [Project summary](design/async-commit/project-summary.md)
    - [Initial design](design/async-commit/initial-design.md)
    - [Non-unique timestamps](design/async-commit/globally-non-unique-timestamps.md)
    - [Replica read](design/async-commit/replica-read.md)
    - [Known issues](design/async-commit/parallel-commit-known-issues-and-solutions.md)
