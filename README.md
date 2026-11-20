# reltix-moments

**Moments**: an AI service layer for reltix's centrix platform, built at an AI hackathon.

It detects the moment a resident needs a service (move-in, repair, move-out), pre-fills everything the partner needs from centrix, holds the conversation with the resident in one tap, hands the job to a partner, and writes the result back to the property file. Every partner plugs in through one manifest file, with no per-partner code.

## Prompts

- [RELTIX_BUILD_PROMPT.md](RELTIX_BUILD_PROMPT.md): the foundation. Build this first.
- [RELTIX_ADDONS_PROMPT.md](RELTIX_ADDONS_PROMPT.md): the extras, in priority order, added once the foundation works.

## Deliverables

- **A. Working demo:** event → resident → partner → result
- **B. Connector spec:** the next partner plugs in the same way (`spec/`)
