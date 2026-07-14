---
name: mantis_report
description: >-
  Generates a human-readable security review packet compiled from reproduced findings.
  Use at the end of a review cycle to produce stakeholder-facing documentation.
  Don't use for auditing code or verifying patches directly.
---

# Reporter (/mantis_report)

## System Goal

Security Reporting Expert. Synthesizes complex, technical finding logs into a
high-quality, human-readable review packet for developers and stakeholders.

## Command Definition

-   **Command:** `/mantis_report`
-   **Description:** Generates a human-readable security review packet
    containing only reproduced findings.

## Instructions

Compile a professional Markdown report detailing only the verified and
reproduced vulnerabilities.

Execute the reporting stage as follows:

1.  **Load and Filter Findings:**

    -   Read all JSON files from the `workspace/findings/` directory.
    -   **Strict Filter:** Include only actionable findings. You MUST include:
        1.  Findings where `repro_status` is `"reproduced"`.
        2.  Findings where `repro_status` is `"statically_confirmed"` BUT
            contain valid external stack traces, sanitizer traces (e.g.
            ASan/UBSan), or crash logs proving empirical execution impact
            (treated as empirically reproduced by calibration).
        3.  Valid Exploit Chains (constructed by the chainer, identified by
            "Exploit Chain" in the title/history) even if they inherited a
            status of `"statically_confirmed"`. Do **not** include false
            positives, non-viable findings, findings that failed to reproduce,
            or ordinary statically confirmed findings that lack empirical
            execution traces.

2.  **Extract Key Artifacts:** For each reproduced finding, extract and format:

    -   **Header Metadata:** Title, ID (UUID), Inferred Exposure, Final Risk
        Score, and Qualitative Priority.
    -   **Vulnerability Description & Impact:** A clear explanation of the bug
        and the concrete impact on the system.
    -   **Reproduction Evidence:**
        -   The PoC script path (`repro_file_path`) and execution command
            (`run_command`).
        -   A clean snippet of the stdout/stderr showing the successful exploit
            trigger (`repro_output`).
    -   **Risk Rationale:** The independent validation reasoning (`reasoning`),
        production viability reasoning (`critic_reasoning`), and outrage factor
        analysis (`outrage_commentary`).
    -   **Remediation & Patch:**
        -   The recommended mitigation strategy.
        -   The verified patch diff (`patch_diff`) and re-attack status to prove
            the fix is resilient.

3.  **Generate Review Packet:**

    -   Write the compiled report to `workspace/report/review_packet.md`.
    -   Use clean, professional Markdown formatting with clear headers, tables
        for metadata, and syntax-highlighted code blocks for logs and diffs.
    -   Include a high-level **Executive Summary** table at the top listing all
        included findings, their priority, and their risk scores.

When complete, notify the user.
