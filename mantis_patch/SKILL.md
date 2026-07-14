---
name: mantis_patch
description: >-
  Generates minimal security fixes using transactional isolation (VCS branches or file backups), applies patches, and verifies them.
  Use when security findings are successfully reproduced and need patches applied and verified.
  Don't use for initial vulnerability research or reproduction payload generation.
---

# Patcher (/mantis_patch)

## System Goal

Security Patching Expert. Generates minimal, correct code fixes, applies them to
source code files, and verifies them inside isolated sandboxes before appending
logs to long-term memory.

## Command Definition

-   **Command:** `/mantis_patch`
-   **Description:** Generates minimal security fixes using transactional
    isolation (VCS branches or file backups), applies patches, and verifies
    them.

## Instructions

Fix successfully reproduced security flaws without breaking standard code
behavior.

Execute the patching and verification stage as follows:

1.  **Load Reproduction Results:** Read the JSON files in the
    `workspace/findings/` directory. Filter for entries where `repro_status` is
    `"reproduced"`. If none exist, notify the user.

2.  **Generate and Apply Minimal Patches:** For each reproduced security flaw:

    -   **Target Agnosticism (Binaries vs Source):** If the target is source
        code, proceed with generating and applying a code patch as described
        below. If the target is a compiled binary or firmware blob without
        source code available, **do not attempt to modify the binary or write
        binary patching scripts**. Instead, skip the branch
        isolation/modification/diff steps and generate a general, high-level
        recommendation for how this issue could be mitigated in a production
        environment without requiring deep technical depth. Output this
        mitigation string in place of the `patch_diff` field.

    -   **Exploit Chains:** If the finding is an exploit chain (identified by
        `"Exploit Chain:"` in the title or history details indicating it was
        constructed by chaining), do **not** generate a code patch or diff.
        Instead, monitor the patch status of its constituent findings (listed in
        its history). Once all constituent findings have been patched and
        verified (status `"VERIFIED_SECURE"`), mark the chain finding as
        `"VERIFIED_SECURE"`. If any constituent patch fails, mark the chain as
        `"VERIFICATION_FAILED"`. Skip branch isolation, testing, and re-attack
        steps for the chain finding itself.

    -   *Optional Parallel Trajectory Search:* If your framework supports
        subagents, you may spawn multiple concurrent subagents to design diverse
        patch implementations. Test all generated patches that successfully
        secure the code without breaking standard functionality, and select the
        *best* patch (e.g., the most minimal, readable, and idiomatic fix)
        rather than just the first one that works.

    -   Read the original flawed file to grasp function dependencies and
        structures.

    -   Design a minimal, correct patch to mitigate the security flaw (e.g.
        adding bound checks, validating sizes, inserting NUL-terminators)
        without breaking other features.

    -   **Transactional Isolation:** Before modifying any source code, evaluate
        the workspace and repository environment (e.g., Git, Mercurial, other
        VCS, or unversioned).

        -   **Warning on Shared Workspaces:** If multiple parallel workers share
            the same repository clone or working directory, using global
            commands like `git stash`, `git checkout`, or global branch
            switching will conflict. In such environments, avoid those commands.
            Instead, prefer working in completely isolated workspace
            clones/directories if supported by the host, or isolate edits using
            file-level backups.
        -   **VCS-Based Isolation (e.g., Git, Mercurial):** If in a dedicated,
            isolated Git repository clone, verify the working tree is clean.
            Stash or discard local changes only if safe to do so. Create and
            check out a temporary development branch (e.g., `git checkout -B
            mantis/tx-[finding_id]` from the base branch). Perform all edits,
            compilation, and testing exclusively on this branch.
        -   **Non-VCS/Unversioned Isolation:** If the repository is unversioned
            or shared without branch isolation, create backup copies of the
            original files (e.g., `cp target.c target.c.orig-[finding_id]`)
            before editing.
        -   Apply the generated patched code directly to the target file. If
            using a dedicated VCS branch, commit the changes to the branch
            (e.g., `git add -A && git commit -m "Apply patch for
            [finding_id]"`). If the commit command fails due to missing user
            configuration, configure local credentials first (e.g., `git config
            --local user.email "mantis@agent.local" && git config --local
            user.name "Mantis Patcher"`). This isolates your edits and prevents
            working tree pollution.

3.  **Post-Patch Verification Run:** *(Skip this step for binary-only targets
    where no code patch was applied)*. To confirm the patch works, re-run the
    reproducer script inside your isolated execution environment. Use the exact
    `"repro_file_path"` and `"run_command"` from the reproduction entry to
    verify the patch.

    -   **VERIFIED SECURE:** If the post-patch sandbox run fails to reproduce
        the bug, the initial patch holds. However, you must now perform a
        **Re-attack**: assume the patch is flawed and explicitly attempt to
        write a new reproducer variant that bypasses your patch to reach the
        same root cause. Only if the re-attack also fails to bypass the fix
        should you mark the patch as fully successful! To ensure true
        independence, launch a fresh `@mantis_reproduce` subagent against the
        patched code to perform this re-attack. You must map the fresh agent's
        output into the `reattack_*` schema fields to prevent overwriting the
        initial `repro_*` evidence.
    -   **VERIFICATION FAILED:** If the sandbox execution still triggers the
        bug, or if your re-attack successfully bypasses your patch, the patch is
        insufficient. Re-evaluate and adapt your fix.

4.  **Extract Patch and Rollback Transaction:** *(Skip this step for binary-only
    targets)*. Do not leave the codebase in an altered state. Once you have a
    final outcome (either `VERIFIED_SECURE` or you have exhausted your retries):

    -   If successful, generate a unified diff representing your exact changes
        and save it to the `"patch_diff"` field. Use a generic diff command
        comparing the modified code to the unmodified base/development version
        (e.g., for Git, compare to the base branch, or use `git diff` against
        the base commit; for unversioned code, use `diff -u original_file
        patched_file`). Do not assume the base branch is named `main` or that
        Git is always used.
    -   **Transactional Clean Up / Rollback**: Restore the codebase to its
        original state so that it is not left in a modified/broken state.
        -   If using Git branches (dedicated-clone mode only): Prior to checking
            out the base branch, ensure the working tree is clean. If there are
            uncommitted changes (e.g., temporary debug modifications or edits
            from verification trials), either commit them on the temporary
            branch first (if you want to preserve them) or discard them entirely
            (by running `git reset --hard` and `git clean -fd`; **do NOT run
            these destructive commands if sharing a clone with other parallel
            workers** as they will nuke concurrent edits) to prevent checkout
            failures or base branch contamination. Once the working tree is
            clean, checkout the base branch. If the patch was successful, you
            may delete the temporary branch (e.g., `git branch -D
            mantis/tx-[finding_id]`). If the patch failed, do not delete it;
            instead, rename it to include a unique Unix epoch suffix (e.g., `git
            branch -m mantis/tx-[finding_id]
            mantis/tx-[finding_id]-failed-$(date +%s)`) to preserve the failed
            changes for inspection without causing branch name collisions on
            retry.
        -   If using file backups or copies: Restore the original files from the
            backups, or delete the temporary workspace directories.

5.  **Append to Long-Term Memory (Continuous Reviewing Link):** For each
    security flaw processed, append a single structured JSON line to a workspace
    database file named `learnings.jsonl` (using append mode). This allows the
    strategist (`/mantis_plan`) to read these historical records in subsequent
    passes and avoid proposing fixes for already patched files.

    -   **Memory Entry Format:** `{"title": "[security_flaw_title]",
        "code_paths": ["[path1:line1]"], "status": "[VERIFIED_SECURE /
        VERIFICATION_FAILED / ERROR]"}`

6.  **Token-Optimized File Updates:** To minimize LLM output tokens, **do not
    re-emit or manually rewrite the entire JSON object in your output.**
    Instead, write a reusable helper script (e.g.,
    `workspace/helpers/append_patch.py`) during your first finding update. For
    all subsequent findings, do not regenerate the script; simply execute the
    existing helper script with the new parameters to append the required
    fields.

    You must append the following to the existing object:

    -   A `"patch_status"` field (e.g., `"VERIFIED_SECURE"`,
        `"VERIFICATION_FAILED"`, or `"ERROR"`).
    -   If a patch was successful, a `"patch_diff"` field containing the unified
        diff.
    -   If a re-attack was performed, the `"reattack_status"`,
        `"reattack_file_path"`, `"reattack_run_command"`, and
        `"reattack_output"` fields.
    -   An entry to the `"history"` array:

    ```json
    {
      "stage": "patch",
      "action": "patched",
      "details": "Patch status evaluated as [VERIFIED_SECURE/VERIFICATION_FAILED/ERROR]"
    }
    ```

When complete, notify the user.
