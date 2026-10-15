# Bug zoo

Bugs found in the wild in Claude's own harness and tools, logged whether or
not they were reported. Exhibit over extermination. Entries are written by
Claude (Sonnet 5.5); the user finds, steers and reviews, and files what gets
filed.

## 2026-10-14: stop hook says a pushed branch is unpushed

- **What happened.** In a cloud session, Claude committed on a new branch and
  ran `git push -u origin <branch>`. The push succeeded: `git ls-remote
  --heads origin <branch>` returned the same SHA as local `HEAD`. A later
  stop-hook check still reported: "Branch '<branch>' has 1 unpushed
  commit(s) and no remote branch. Please push these changes to the remote
  repository."
- **The first alert was correct.** The same hook fired once before the push,
  when the commit really was unpushed. The false alarm is the second one.
- **Cause.** The repo was cloned with `git clone --depth 1`, so the clone is
  single-branch and its fetch refspec is only
  `+refs/heads/main:refs/remotes/origin/main`. A push of another branch
  succeeds but doesn't create `refs/remotes/origin/<branch>` locally. The
  hook (`~/.claude/stop-hook-git-check.sh`) tests for that local ref, doesn't
  find it, falls back to `origin/HEAD`, counts `origin/HEAD..HEAD` as 1, and
  prints the "no remote branch" message. It never asks the remote.
- **Workaround.** After pushing, run
  `git fetch origin +refs/heads/<branch>:refs/remotes/origin/<branch>`. The
  hook then exits 0 (checked by running it with
  `{"stop_hook_active":false}` on stdin).
- **Possible fixes.** Have the hook check the remote directly, for example
  `git ls-remote --exit-code --heads origin <branch>`, or compare against the
  branch's configured upstream. Or set the clone's fetch refspec to
  `+refs/heads/*:refs/remotes/origin/*`, or update the tracking ref as part of
  the push step.
- **Why it matters.** Low severity, but it burns a turn and tells the agent
  to redo work that is done. An agent that doesn't verify might re-push or
  rewrite history to satisfy it. The hook's recursion guard stops it from
  looping forever.
- **Provenance.** The hook appears to come from the cloud environment, not
  from the user: its comments refer to the environment's checkout flow. Its
  signing block already mentions the single-branch, shallow-clone case, but
  the unpushed-count block still uses the `origin/HEAD` fallback. I have not
  read the issue its comments cite, anthropics/claude-code#69586.
- **Status.** Not filed yet; the user will file it. Claude has no way to file
  it from this session. Workaround in use.
