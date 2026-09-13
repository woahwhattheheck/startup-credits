# Bounty #16471 audit — docstring held-sweep starvation

Target: `Scottcjn/rustchain-bounties`

Audited upstream commit: `c2d3f22c33cfaf8b2ff9de3412e1460e64fe70a4`

Bounty: https://github.com/Scottcjn/rustchain-bounties/issues/16471

## Finding

The scheduled docstring sweep can permanently starve held claims beyond its first search page while reporting a fully green run with 60 successful adjudications.

Two current behaviors compose into the failure.

First, `.github/workflows/docstring-gate.yml` discovers `awaiting-merge` claims with one unpaginated Search API request:

```sh
held=$(gh api -X GET search/issues \
        -f q="repo:${GH_REPO} is:issue is:open label:awaiting-merge" \
        -f per_page=60 --jq '.items[].number' 2>/dev/null || true)
```

The same workflow stops the combined held/fresh loop after 60 attempts:

```sh
[ "$attempted" -ge 60 ] && echo "::notice::stopped at 60 claims this run; the rest process next sweep" && break
```

Second, when an `awaiting-merge` claim later becomes payable, `scripts/docstring_gate.py` adds `bounty-eligible` and `docstring-verified` but never removes `awaiting-merge`. On every subsequent invocation the gate's initial terminal-label check prints `already adjudicated; skipping` and returns `0`. The workflow treats every `rc == 0` as a successful adjudication and increments `adjudicated`.

Relevant current source:

- Workflow: https://github.com/Scottcjn/rustchain-bounties/blob/c2d3f22c33cfaf8b2ff9de3412e1460e64fe70a4/.github/workflows/docstring-gate.yml
- Gate: https://github.com/Scottcjn/rustchain-bounties/blob/c2d3f22c33cfaf8b2ff9de3412e1460e64fe70a4/scripts/docstring_gate.py

## Concrete silent-success path

1. There are 61 open claims carrying `awaiting-merge`.
2. Claims 1–60 later merge and are successfully verified. They retain `awaiting-merge` because the gate only adds the terminal labels.
3. Claim 61 also becomes merge-ready, but is outside the single `per_page=60` Search API response.
4. On a later scheduled sweep, the same 60 terminal claims can occupy the only fetched page. Each gate invocation immediately returns `0` as `already adjudicated`.
5. The shell increments `adjudicated` for all 60, reaches the 60-attempt break, prints `attempted 60 claim(s): adjudicated 60, failed 0`, and exits green.
6. Claim 61 was never fetched or adjudicated. Because the first-page claims still satisfy `label:awaiting-merge`, the same green no-op can repeat while later held claims remain invisible beyond page 1.

This is distinct from the previously reported inventory-failure path where a GitHub error becomes an empty queue, and from the closed-PR `awaiting-merge` loop. Here every GitHub read may succeed and return 60 rows. The wrong result comes from stale queue membership plus an unpaginated bounded inventory plus counting an already-terminal skip as a completed adjudication.

## Suggested remediation

- Remove `awaiting-merge` whenever a claim reaches a terminal state such as `docstring-verified`/`bounty-eligible`, `needs-human`, or a rejected/closed outcome; alternatively exclude terminal labels from the held search.
- Paginate the held and fresh Search API queries to exhaustion instead of treating one `per_page=60` response as the queue.
- Distinguish gate outcomes such as `processed`, `deferred`, and `already-terminal`; do not increment `adjudicated` solely because the process returned zero for `already adjudicated; skipping`.

## Submission status

A direct GitHub issue-comment submission to #16471 returned `403 Resource not accessible by integration`. The sponsor's `docs/HOW_TO_SUBMIT_A_BOUNTY.md` explicitly documents publication of a report in a claimant-controlled repository as an accepted fallback for this connector limitation.

Payout destination: GitHub handle `woahwhattheheck`.
