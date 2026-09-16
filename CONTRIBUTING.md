# Contributing

Documentation for a solo-maintained platform (see [README.md](README.md)). Corrections and
clarifications are welcome via pull request.

Every change goes through a PR into `main` — branch protection requires GitHub-verified
signed commits (see `shop_platform/CONTRIBUTING.md` for how to set up commit signing) and a
passing `trufflehog` check. There's no `build` check here since this repo has no code to
build.

Keep documents in sync with the code they describe — if you change behavior in one of the
service repos that a doc here describes, update the doc in the same PR where practical, or
open a follow-up issue here.
