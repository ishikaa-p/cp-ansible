# A4: superseded by Phase B

The plan called for a rendered-artifact diff on 2-3 existing molecule scenarios (converge from
both `v7.6.12` and HEAD, diff rendered configs + directory listings). Not executed as originally
scoped:

- No Docker/molecule available in this local environment.
- Running it via this repo's Semaphore CI on-demand task would require pushing the local commits
  from this session to a branch Semaphore can see - which hits the same branch-protection
  constraint encountered earlier this session (direct push to `ANSIENG-5900-rootless-fixes-molecule`
  is blocked; the personal fork's push is also org-blocked). Getting HEAD onto CI would need going
  through a PR-into-`ANSIENG-5900` cycle, a heavier action.

Phase B (the live EC2 root-brownfield-upgrade test) produces the same category of evidence -
actual rendered config files and actual on-disk directory state, diffed between the pre-PR and
post-PR code - but on real VMs with a real running cluster and real data, which is a stronger
check than a molecule-container diff would have been. A4's goal is considered met by Phase B's
results; see `B_SUMMARY.md`.
