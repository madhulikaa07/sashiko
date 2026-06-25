# Design: Optional Author Suggestions

## Motivation

Sashiko currently focuses on review findings: possible bugs, regressions,
security issues, or other concerns that may require author or maintainer action.
That signal is intentionally narrow and should remain distinct from advisory
feedback.

There is a separate class of feedback that can be useful to patch authors but
should not be presented as a finding. Examples include design alternatives,
maintainability improvements, documentation suggestions, test suggestions,
patch organization feedback, and small code simplifications.

This document proposes an optional "author suggestions" feature for that
feedback. Suggestions are intended to help authors improve patches without
mixing optional advice into Sashiko's existing findings stream.

## Goals

- Keep suggestions separate from verified review findings.
- Make suggestions optional and configurable.
- Generate suggestions using the final verified review context.
- Avoid increasing mailing-list noise by using conservative defaults.
- Support both design-level and code-level suggestions.
- Allow API, CLI, web UI, and email consumers to display suggestions distinctly.
- Bound the number and confidence level of suggestions that are shown to users.

## Non-goals

- Do not turn Sashiko into an automatic patch author.
- Do not generate large rewrites by default.
- Do not treat suggestions as review failures.
- Do not mix optional suggestions into the existing findings list.
- Do not email suggestions by default unless policy explicitly enables it.
- Do not suggest pure style churn or subjective rewrites with no clear benefit.

## Terminology

A **finding** is a possible correctness, security, regression, or other review
issue that Sashiko believes may need author attention.

A **suggestion** is optional advice that may improve the patch but is not asserted
to be a bug. Suggestions should be presented as advisory, not as blocking review
comments.

Examples of suggestions include:

- documenting a new lifetime or locking rule;
- splitting a refactor from a behavior change;
- adding a focused test;
- extracting duplicated validation into a helper;
- simplifying an error path;
- considering an alternative API shape;
- adding a note to a commit message when the motivation is otherwise unclear.

## User-visible behavior

Suggestions should be displayed only after the normal review has completed and
the suggestion generation stage, if enabled, has finished.

Findings and suggestions must be displayed in separate sections. A user should
not have to infer from wording whether an item is a bug report or optional
advice.

Example text output:

```text
Findings:
  - Possible missing unlock on the error path in foo_probe().

Suggestions:
  - Consider documenting the new ownership rule for foo_register().
```

If there are no findings but suggestions exist, the UI and CLI may still display
suggestions. Email delivery should remain more conservative and is discussed
below.

## Review pipeline integration

Suggestions should be generated after the normal review stages have produced
verified findings. The suggestion stage should not alter review status and should
not convert a successful review into a failed review.

Conceptually:

```text
Patch and context
  -> normal multi-stage review
  -> verified findings
  -> optional suggestion generation
  -> findings and suggestions stored separately
```

The suggestion stage should receive:

- the commit message;
- the target patch diff;
- relevant source context loaded by Sashiko;
- final verified findings;
- dismissed concerns, when useful for avoiding false suggestions;
- series context, including later patches that may already address an idea;
- subsystem-specific review prompts and kernel rules.

The suggestion stage should not treat unverified intermediate concerns as facts.
If intermediate concerns are provided, they should be clearly labeled as
unverified hypotheses.

## Suggestion quality rules

The prompt and post-processing should enforce conservative behavior:

- Do not suggest pure style churn.
- Do not suggest changes that contradict Linux kernel style or subsystem norms.
- Do not suggest broad rewrites unless explicitly requested.
- Prefer small, actionable suggestions.
- Explain why the suggestion improves the patch.
- Avoid duplicating verified findings as suggestions.
- Avoid suggestions that are already addressed by later patches in the series.
- Omit low-confidence suggestions by default.
- Limit the total number of suggestions per patchset.

A suggestion should be omitted if the model cannot provide a concrete rationale.

## LLM output contract

The suggestions stage should return structured JSON. This keeps suggestions easy
to validate, store, filter, and render separately from findings.

Each suggestion should include:

- category;
- title;
- rationale;
- suggested change;
- optional suggested diff;
- confidence;
- whether it is design-level or code-level;
- optional file and line location.

Example output:

```json
{
  "suggestions": [
    {
      "category": "documentation",
      "title": "Document the new ownership rule",
      "rationale": "The patch changes when callers may release the object, but the kerneldoc still describes the old lifetime rule.",
      "suggested_change": "Update the kerneldoc for foo_register() to describe the new ownership requirement.",
      "suggested_diff": null,
      "confidence": "high",
      "is_design_level": true,
      "is_code_level": false,
      "applies_to_file": "include/linux/foo.h",
      "applies_to_line": 87
    }
  ]
}
```

Suggested categories include:

- `design`
- `api`
- `documentation`
- `testing`
- `maintainability`
- `performance`
- `error-handling`
- `locking`
- `lifetime`
- `code-simplification`
- `patch-organization`

Suggested confidence values are:

- `low`
- `medium`
- `high`

Implementations should default to hiding or dropping `low` confidence
suggestions.

## Suggested diffs

The feature may eventually allow suggestions to include small proposed diffs, but
suggested diffs should be treated carefully.

Suggested diffs should be:

- optional;
- disabled by default in email;
- limited in size;
- clearly labeled as generated suggestions;
- restricted to small, local changes with high confidence.

Large generated rewrites should not be emitted by default. If Sashiko cannot
produce a small, reviewable diff, it should provide prose guidance instead.

## Storage model

Suggestions should be stored separately from findings.

A future schema could add a `suggestions` table with fields such as:

- `id`;
- `review_id`;
- `patchset_id`;
- `patch_id`;
- `category`;
- `title`;
- `rationale`;
- `suggested_change`;
- `suggested_diff`;
- `confidence`;
- `is_design_level`;
- `is_code_level`;
- `applies_to_file`;
- `applies_to_line`;
- `created_at`.

Linking suggestions to `review_id` preserves the exact review run that produced
them. Linking to `patchset_id` and optionally `patch_id` makes UI and API queries
straightforward.

## API behavior

Suggestions should be available through review and patchset APIs.

Possible endpoints:

```text
GET /api/suggestions?review_id=<id>
GET /api/suggestions?patchset_id=<id>
```

Existing review endpoints may also include a `suggestions` field once storage
exists. API responses should keep findings and suggestions separate:

```json
{
  "findings": [],
  "suggestions": []
}
```

## CLI behavior

The CLI should be able to show suggestions separately from findings.

Possible interface:

```bash
sashiko-cli show latest --suggestions
sashiko-cli local HEAD --suggestions --force-local
```

The exact default output behavior can be decided during implementation. A
conservative default is to show suggestions in detailed output but not in compact
or summary output unless explicitly requested.

## Web UI behavior

The web UI should render suggestions in a section separate from findings.

Recommended UI behavior:

- label the section "Suggestions" or "Optional suggestions";
- show confidence and category;
- group suggestions by patch when possible;
- make suggested diffs collapsible;
- make clear that suggestions are advisory and not verified findings.

## Email behavior

Suggestions should not be included in outbound email by default.

A future email policy option could allow:

- no suggestions in email;
- high-confidence suggestions only;
- all suggestions above a configured confidence threshold.

This avoids sending optional advice to authors unless an operator explicitly
chooses that behavior. If suggestions are included in email, they should appear
in a separate section after findings and be clearly marked as optional.

Example:

```text
Optional suggestions:

- Consider documenting the new ownership rule in include/linux/foo.h.
  This is not necessarily a correctness issue, but it may help future callers
  avoid misuse of the new API.
```

## Configuration

Possible future review settings:

```toml
[review]
suggestions_enabled = false
suggestion_mode = "both" # "off", "design", "code", or "both"
max_suggestions_per_patchset = 5
include_suggestion_diffs = false
min_suggestion_confidence = "medium"
```

Email policy should remain separate from review generation. For example:

```toml
[email]
include_suggestions = false
suggestion_email_min_confidence = "high"
```

The exact location of email settings should follow the existing email policy
configuration conventions.

## Failure handling

Suggestion generation should be best-effort. Failure to generate suggestions
should not fail the review if findings were generated successfully.

If the suggestion stage fails:

- log the error;
- store review results normally;
- mark suggestions as unavailable or omitted;
- do not change the patchset status to failed solely because suggestions failed.

Operators may choose to expose suggestion-stage failures in review logs for
debugging.

## Implementation plan

A possible implementation sequence is:

1. Add schema and database methods for suggestions.
2. Add Rust types for suggestion records and LLM output parsing.
3. Add an optional suggestions stage to the review worker.
4. Expose suggestions through the API.
5. Display suggestions in the web UI.
6. Add CLI display support.
7. Add email-policy support, disabled by default.

The email support can be deferred until the rest of the feature is stable.

## Open questions

- Should suggestions be generated by default?
- Should suggestions be shown when no findings exist?
- Should suggested diffs be allowed initially, or deferred?
- Should suggestions be linked primarily to reviews, patchsets, or patches?
- What confidence threshold should be used by default?
- Should local review mode enable suggestions independently from daemon mode?
- Should suggestions be generated for every review, or only when requested by the
  submitter/operator?
