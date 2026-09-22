# Tracker provider: GitLab

This file is the GitLab implementation of the tracker operations contract (see `TEMPLATE.md` for the contract itself). It is a **stand-alone** provider: GitLab owns the issues, merge requests, reviews, CI pipelines, and labels, so no companion descriptor is needed. Every operation runs against the GitLab REST API v4 through the `glab` CLI (`glab api`), on gitlab.com or a self-managed instance.

At runtime: `om-setup-agent-pipeline` copies this file into the repository at `.ai/trackers/gitlab.md` and sets `"tracker": "gitlab"`. When a skill says "tracker operation **get-pr**", execute the command documented under that operation heading in the repo's copy. The repo's copy is authoritative: teams extend or override any operation by editing it, and every skill picks the change up on its next run.

Skills say "PR" throughout; on GitLab a PR is a **merge request** (MR), and `{prNumber}` is the MR's project-scoped `iid`.

## Prerequisites

- [`glab`](https://gitlab.com/gitlab-org/cli#installation) installed and authenticated (`glab auth login`, or `glab auth login --hostname <host>` for a self-managed instance; CI can export `GITLAB_TOKEN`), plus `jq`. Verify with **auth-check** before a batch run. Never print the token — do not run `glab auth status --show-token` in a skill.
- Every operation goes through `glab api` with REST v4 paths, not through convenience verbs (`glab mr update`, `glab issue edit`). The REST surface is stable across GitLab and `glab` versions, and each mutation touches only the fields it changes (`TEMPLATE.md`: mutate through the narrowest API surface). **auth-check** probes the `glab api` flags this file relies on (`--paginate`, `--input`, `--header`) instead of pinning a version.
- `glab` resolves the host and project from the checkout's git remote. All operations accept an optional `{repo}`: set `REPO` to the full project path (`group/subgroup/project`) to address another project on the same host; set `GITLAB_HOST` as well when that project lives on a different instance.
- Tier notes: multiple assignees, approval rules, and external status checks are GitLab Premium features. Everything here works on the Free tier and degrades as documented under **assign-issue / unassign-issue**, **review-pr**, and **get-required-checks**.

## Conventions

- Issues are written `#12`, merge requests `!12`; the two are **separate number spaces**. Human-facing text (PR bodies, comments) uses `!12` for an MR. The chaining reference lines skills emit keep their fixed machine shape — `PR: #<iid> (link: <MR URL>)` — so parsers are unchanged; the link disambiguates.
- An MR declares the issue it resolves with `Closes #12` (or `Fixes #12`, `Resolves #12`) in its description. GitLab closes the issue when the MR merges **into the project's default branch**; when the base branch is anything else, run **close-issue** explicitly after the merge.
- A **draft** MR is one whose title starts with `Draft:` (`[Draft]` and `(Draft)` are also recognized). **create-pr** adds the prefix, **mark-pr-ready** strips it, and **update-pr** preserves it while the MR is still a draft.
- Comments are GitLab **notes**. System notes (label changes, pushes) are always filtered out. Because issues and MRs have separate number spaces, a comment id is a **parent-qualified handle** — `issues/<iid>/<noteId>` or `merge_requests/<iid>/<noteId>` — emitted by **list-issue-comments**, **comment-issue**, and **comment-pr**, and accepted by **get-issue-comment**, **get-pr-comment**, **get-review-comment**, and **update-comment**. A note URL maps onto a handle directly: `…/-/issues/12#note_345` → `issues/12/345`, `…/-/merge_requests/12#note_345` → `merge_requests/12/345`.
- Operations that take "an issue or PR number" (**list-issue-comments**) need the kind: use `merge_requests` when the caller is working a PR, `issues` otherwise.
- Claim/lock signals on an issue or MR are: assignee = the automation user, the `in-progress` label, and a `🤖`-prefixed timestamped claim note. All three are readable back through **get-issue** / **get-pr**. The `ci-monitoring` label is **not** a claim signal — it marks finished, reported work whose CI-result follow-up is still owed, and never makes another skill back off.
- Review verdicts: GitLab's approval is native (`/approve`, `/unapprove`). GitLab has no REST verdict for "request changes" on every tier, so **review-pr** records each verdict as a note whose first line is a hidden marker — `<!-- review: APPROVED -->` or `<!-- review: CHANGES_REQUESTED -->` — and **get-pr** rebuilds `reviews` / `latestReviews` from those notes plus native approvals and reviewer states. `reviewDecision` is `CHANGES_REQUESTED` while a reviewer is in GitLab's "requested changes" state or the MR carries the `changes-requested` pipeline label, `APPROVED` when at least one approval satisfies the project's approval rules, and `REVIEW_REQUIRED` otherwise.
- CI truth for an MR comes from its **head pipeline** (**get-pr-checks**). A job with `allow_failure: true` that fails is reported as `NEUTRAL` — GitLab itself does not block the merge on it.
- Multi-line bodies are always sent from a file through `jq --rawfile` into `glab api --input -`, so formatting survives and nothing large touches a command line.
- Validate every externally-sourced value before interpolation: iids are numeric (`gl_iid`), `REPO` is a path (`gl_project`), comment handles match the handle shape (`gl_handle`).

## Shared helpers

Source this block once before running any operation below; the label guards and operations call these functions.

```bash
# Target project for API paths: $REPO (group/subgroup/project) URL-encoded when a skill
# addresses another project, otherwise glab's `:id` placeholder for the checkout's project.
gl_project() {
  if [ -n "${REPO:-}" ]; then
    case "$REPO" in
      /*|*/|*//*|*[!A-Za-z0-9._/-]*) echo "Invalid GitLab project path: $REPO" >&2; return 1 ;;
    esac
    printf '%s' "$REPO" | jq -sRr @uri
  else
    printf ':id'
  fi
}

gl_iid() {
  case "$1" in ''|*[!0-9]*) echo "Invalid GitLab iid: $1" >&2; return 1 ;; esac
}

# Comment handle: issues/<iid>/<noteId> or merge_requests/<iid>/<noteId>.
gl_handle() {
  case "$1" in
    *[!a-z_0-9/]*) ;;
    issues/[0-9]*/[0-9]*|merge_requests/[0-9]*/[0-9]*)
      rest=${1#*/}; case "${rest%/*}" in ''|*[!0-9]*) ;; *) return 0 ;; esac ;;
  esac
  echo "Invalid GitLab comment handle: $1" >&2; return 1
}

# JSON write. $1 = HTTP method, $2 = API path; the JSON body is read from stdin.
gl_write() {
  glab api -X "$1" "$2" -H 'Content-Type: application/json' --input -
}

# Every page of a list endpoint, merged into one JSON array.
gl_list() {
  case "$1" in *\?*) sep='&' ;; *) sep='?' ;; esac
  glab api --paginate "$1${sep}per_page=100" | jq -s 'add // []'
}

# jq definitions shared by the PR, check, and CI-run mappings below.
GL_JQ_DEFS='
def gl_state: if . == "opened" then "OPEN" elif . == "merged" then "MERGED" else "CLOSED" end;
def gl_draft_re: "^\\s*((\\[draft\\]|\\(draft\\)|draft:)\\s*)+";
def gl_review_state: (.body // "" | capture("^<!-- review: (?<s>APPROVED|CHANGES_REQUESTED) -->") | .s) // null;
def gl_mergeable:
  if .has_conflicts == true or .detailed_merge_status == "conflict" then "CONFLICTING"
  elif (.detailed_merge_status // "unchecked") | IN("unchecked", "checking", "preparing", "approvals_syncing") then "UNKNOWN"
  else "MERGEABLE" end;
def gl_merge_state:
  (.detailed_merge_status // "unchecked") as $s
  | if $s == "mergeable" then "CLEAN"
    elif $s == "conflict" or .has_conflicts == true then "DIRTY"
    elif $s == "need_rebase" then "BEHIND"
    elif $s == "draft_status" then "DRAFT"
    elif $s | IN("unchecked", "checking", "preparing", "approvals_syncing") then "UNKNOWN"
    else "BLOCKED" end;
def gl_check:
  (.status // "") as $s
  | (if $s == "success" then ["SUCCESS", "pass"]
     elif $s == "failed" then (if .allow_failure == true then ["NEUTRAL", "pass"] else ["FAILURE", "fail"] end)
     elif $s == "running" then ["IN_PROGRESS", "pending"]
     elif $s == "manual" then (if .allow_failure == false then ["PENDING", "pending"] else ["SKIPPED", "skipping"] end)
     elif $s == "skipped" then ["SKIPPED", "skipping"]
     elif $s | IN("canceled", "canceling") then ["CANCELLED", "cancel"]
     else ["PENDING", "pending"] end) as $m
  | {name, state: $m[0], bucket: $m[1], link: .web_url, workflow: .stage};
def gl_run_status:
  (.status // "") as $s
  | if $s | IN("created", "waiting_for_resource", "preparing", "pending", "scheduled", "waiting_for_callback") then {status: "queued", conclusion: ""}
    elif $s | IN("running", "canceling") then {status: "in_progress", conclusion: ""}
    elif $s == "success" then {status: "completed", conclusion: "success"}
    elif $s == "failed" then {status: "completed", conclusion: "failure"}
    elif $s == "canceled" then {status: "completed", conclusion: "cancelled"}
    elif $s == "skipped" then {status: "completed", conclusion: "skipped"}
    elif $s == "manual" then {status: "completed", conclusion: "action_required"}
    else {status: "queued", conclusion: ""} end;
def gl_run: {databaseId: .id, workflowName: (.source // "pipeline"), name: "pipeline #\(.id)", headSha: .sha, url: .web_url, createdAt: .created_at} + gl_run_status;
'

# Merge request → the GitHub-shaped PR object skills consume (field set: see get-pr).
# $1 = MR iid. GL_PR_LIGHT=1 skips notes, commits, and diffs (list-prs uses it).
gl_pr_json() {
  gl_iid "$1" || return 1
  p=$(gl_project) || return 1
  t=$(mktemp -d) || return 1
  if ! glab api "projects/$p/merge_requests/$1" > "$t/mr"; then rm -rf "$t"; return 1; fi
  glab api "projects/$p/merge_requests/$1/approvals" > "$t/approvals" 2>/dev/null || echo '{}' > "$t/approvals"
  glab api "projects/$p/merge_requests/$1/reviewers" > "$t/reviewers" 2>/dev/null || echo '[]' > "$t/reviewers"
  gl_list "projects/$p/merge_requests/$1/closes_issues" > "$t/closes" 2>/dev/null || echo '[]' > "$t/closes"
  light=false
  echo null > "$t/source"
  if [ "${GL_PR_LIGHT:-0}" = 1 ]; then
    light=true
    for f in notes commits diffs; do echo '[]' > "$t/$f"; done
  else
    gl_list "projects/$p/merge_requests/$1/notes?sort=asc" > "$t/notes"
    gl_list "projects/$p/merge_requests/$1/commits" > "$t/commits"
    gl_list "projects/$p/merge_requests/$1/diffs" > "$t/diffs"
    src=$(jq -r 'if .source_project_id != .target_project_id then .source_project_id else empty end' "$t/mr")
    if [ -n "$src" ]; then
      glab api "projects/$src" > "$t/source" 2>/dev/null || echo null > "$t/source"
    fi
  fi
  jq -n --argjson light "$light" \
    --slurpfile mr "$t/mr" --slurpfile approvals "$t/approvals" --slurpfile reviewers "$t/reviewers" \
    --slurpfile closes "$t/closes" --slurpfile notes "$t/notes" --slurpfile commits "$t/commits" \
    --slurpfile diffs "$t/diffs" --slurpfile source "$t/source" "$GL_JQ_DEFS"'
    def count(p): [(.diff // "") | split("\n")[] | select(startswith(p))] | length;
    $mr[0] as $m | ($approvals[0] // {}) as $a | ($reviewers[0] // []) as $rv
    | "merge_requests/\($m.iid)" as $k
    | ($m.references.full // "" | sub("!\\d+$"; "")) as $target
    | (if $m.source_project_id != $m.target_project_id then ($source[0].path_with_namespace // null) else $target end) as $head
    | [$diffs[0][] | {path: .new_path, additions: count("+"), deletions: count("-")}] as $files
    | [$notes[0][] | select(.system | not) | select(gl_review_state != null)
        | {id: "\($k)/\(.id)", author: {login: .author.username}, state: gl_review_state, body, submittedAt: .created_at}] as $marked
    | ($marked
       + [$rv[] | select(.state == "requested_changes") | .user.username as $u
          | select([$marked[] | select(.author.login == $u and .state == "CHANGES_REQUESTED")] | length == 0)
          | {id: null, author: {login: $u}, state: "CHANGES_REQUESTED", body: "", submittedAt: null}]
       + [($a.approved_by // [])[] | .user.username as $u
          | select([$marked[] | select(.author.login == $u and .state == "APPROVED")] | length == 0)
          | {id: null, author: {login: $u}, state: "APPROVED", body: "", submittedAt: null}]
      ) as $reviews
    | {
        number: $m.iid, title: $m.title, url: $m.web_url, body: ($m.description // ""),
        state: ($m.state | gl_state), author: {login: $m.author.username},
        isDraft: ($m.draft // $m.work_in_progress // false),
        baseRefName: $m.target_branch, baseRefOid: ($m.diff_refs.base_sha // null),
        headRefName: $m.source_branch, headRefOid: $m.sha,
        headRepository: {nameWithOwner: $head},
        headRepositoryOwner: {login: (if $head then ($head | sub("/[^/]+$"; "")) else null end)},
        isCrossRepository: ($m.source_project_id != $m.target_project_id),
        maintainerCanModify: ($m.allow_collaboration // false),
        mergeable: ($m | gl_mergeable), mergeStateStatus: ($m | gl_merge_state),
        reviewDecision: (
          if any($rv[]; .state == "requested_changes") or any($m.labels[]; . == "changes-requested") then "CHANGES_REQUESTED"
          elif ($a.approved // false) and (($a.approved_by // []) | length > 0) then "APPROVED"
          else "REVIEW_REQUIRED" end),
        labels: [$m.labels[] | {name: .}],
        assignees: [($m.assignees // [])[] | {login: .username}],
        reviews: ($reviews | sort_by(.submittedAt // "")),
        latestReviews: ($reviews | group_by(.author.login) | map(sort_by(.submittedAt // "") | last)),
        commits: [$commits[0][] | {oid: .id, messageHeadline: .title, authoredDate: .authored_date}],
        files: $files,
        comments: [$notes[0][] | select((.system | not) and .type != "DiffNote" and gl_review_state == null)
          | {id: "\($k)/\(.id)", author: {login: .author.username}, body, createdAt: .created_at, url: "\($m.web_url)#note_\(.id)"}],
        closingIssuesReferences: [$closes[0][] | {number: .iid, url: .web_url}],
        createdAt: $m.created_at, updatedAt: $m.updated_at, mergedAt: $m.merged_at, closedAt: $m.closed_at,
        mergeCommit: (($m.merge_commit_sha // $m.squash_commit_sha) as $s | if $s then {oid: $s} else null end),
        additions: (if $light then null else ([$files[].additions] | add // 0) end),
        changedFiles: (if $light then ($m.changes_count // null | tostring | tonumber? // null) else ($files | length) end)
      }'
  status=$?
  rm -rf "$t"
  return "$status"
}
```

## Label guards

Every label mutation goes through an existence guard so a missing label degrades to a logged skip instead of a failure, and `labels.enabled: false` in the config skips label operations entirely. The guards mutate with `add_labels` / `remove_labels` on the MR or issue itself, which changes only those labels. The project label list includes labels inherited from parent groups, so group-level labels satisfy the guard. GitLab separates label names with commas in these fields, so a label containing a comma is skipped.

```bash
label_exists() {
  gl_list "projects/$(gl_project)/labels" | jq -e --arg l "$1" 'any(.[]; .name == $l)' >/dev/null
}

# $1 = add|remove, $2 = label, $3 = issues|merge_requests, $4 = iid.
gl_label() {
  gl_iid "$4" || return 1
  jq -n --arg k "$1_labels" --arg v "$2" '{($k): $v}' \
    | gl_write PUT "projects/$(gl_project)/$3/$4" >/dev/null
}

# PR (merge request) labels. $1 = label, $2 = MR iid.
apply_label() {
  if [ "$LABELS_ENABLED" != "true" ]; then return 0; fi
  case "$1" in *,*) echo "Skipping label '$1' (GitLab label fields are comma-separated)."; return 0 ;; esac
  if label_exists "$1"; then
    gl_label add "$1" merge_requests "$2"
  else
    echo "Skipping label '$1' (not defined in this project or its groups). Create it with the create-label operation."
  fi
}

# Issue labels. $1 = label, $2 = issue iid.
apply_issue_label() {
  if [ "$LABELS_ENABLED" != "true" ]; then return 0; fi
  case "$1" in *,*) echo "Skipping label '$1' (GitLab label fields are comma-separated)."; return 0 ;; esac
  if label_exists "$1"; then
    gl_label add "$1" issues "$2"
  else
    echo "Skipping label '$1' (not defined in this project or its groups). Create it with the create-label operation."
  fi
}

# Removal. Removing a label that is not applied is a no-op, not a failure.
remove_label() {
  if [ "$LABELS_ENABLED" != "true" ]; then return 0; fi
  gl_label remove "$1" merge_requests "$2" 2>/dev/null || true
}
remove_issue_label() {
  if [ "$LABELS_ENABLED" != "true" ]; then return 0; fi
  gl_label remove "$1" issues "$2" 2>/dev/null || true
}

# Pipeline labels are mutually exclusive: setting one removes the others first.
# Note the argument order, same as every descriptor: $1 = MR iid, $2 = label.
set_pipeline_label() {
  if [ "$LABELS_ENABLED" != "true" ]; then return 0; fi
  for label in $PIPELINE_LABELS; do
    [ "$label" = "$2" ] && continue
    remove_label "$label" "$1"
  done
  apply_label "$2" "$1"
}
```

Cross-project targets need no extra flags: every guard resolves the project through `gl_project`, so the existence check and the mutation always address the same project.

Read the labels back (**get-pr** / **get-issue**, field `labels`) whenever the label state gates a later decision — a skipped mutation is a normal outcome.

## Operations

### Identity and repository

#### auth-check
Verify the CLI, its API surface, and the credentials. → non-zero exit on any gap.
```bash
gitlab_tracker_auth_check() {
  command -v glab >/dev/null || { echo "glab CLI is not installed: https://gitlab.com/gitlab-org/cli#installation" >&2; return 1; }
  command -v jq >/dev/null || { echo "jq is required by the GitLab tracker descriptor" >&2; return 1; }
  glab --version
  glab auth status || return 1
  glab api --help | grep -Fq -- '--paginate' &&
    glab api --help | grep -Fq -- '--input' &&
    glab api --help | grep -Fq -- '--header' || {
    echo "Installed glab lacks the required 'glab api' surface (--paginate, --input, --header); upgrade glab." >&2
    return 1
  }
  glab api user >/dev/null || {
    echo "glab cannot reach the GitLab API for this checkout; check the git remote, GITLAB_HOST, and glab auth." >&2
    return 1
  }
}
gitlab_tracker_auth_check
```

#### current-user
→ the automation user's username.
```bash
CURRENT_USER=$(glab api user | jq -r '.username')
[ -n "$CURRENT_USER" ] && [ "$CURRENT_USER" != null ] || { echo "Could not resolve the GitLab automation user" >&2; exit 1; }
```

#### repo-info
→ full project path (`group/subgroup/project`), default branch, and web URL.
```bash
glab api "projects/$(gl_project)" | jq '{nameWithOwner: .path_with_namespace, defaultBranch: .default_branch, url: .web_url, visibility}'
REPO=$(glab api "projects/$(gl_project)" | jq -r '.path_with_namespace')
```

#### default-branch
→ the project's default branch (used when the config's `baseBranch` is `"auto"`).
```bash
BASE_BRANCH=$(glab api "projects/$(gl_project)" 2>/dev/null | jq -r '.default_branch // empty')
[ -z "$BASE_BRANCH" ] && BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
[ -z "$BASE_BRANCH" ] && BASE_BRANCH="main"
```

### Issues

#### get-issue
`{issueId}`, field list → issue data in the same shape as `github.md` (`state` is `OPEN`/`CLOSED`; comments carry note handles).
```bash
gl_issue_json() {
  gl_iid "$1" || return 1
  p=$(gl_project) || return 1
  t=$(mktemp -d) || return 1
  glab api "projects/$p/issues/$1" > "$t/issue" || { rm -rf "$t"; return 1; }
  gl_list "projects/$p/issues/$1/notes?sort=asc" > "$t/notes"
  jq -n --slurpfile i "$t/issue" --slurpfile n "$t/notes" '
    $i[0] as $x | {
      number: $x.iid, title: $x.title, body: ($x.description // ""),
      state: (if $x.state == "opened" then "OPEN" else "CLOSED" end),
      author: {login: $x.author.username}, url: $x.web_url,
      labels: [$x.labels[] | {name: .}], assignees: [($x.assignees // [])[] | {login: .username}],
      createdAt: $x.created_at, closedAt: $x.closed_at,
      comments: [$n[0][] | select(.system | not)
        | {id: "issues/\($x.iid)/\(.id)", author: {login: .author.username}, body, createdAt: .created_at, url: "\($x.web_url)#note_\(.id)"}]
    }'
  status=$?; rm -rf "$t"; return "$status"
}
gl_issue_json {issueId}
```

#### search-issues
Query and state (`opened`, `closed`, or omit for all) → matching issues. Searches title and description.
```bash
Q=$(printf '%s' "<query>" | jq -sRr @uri)
glab api "projects/$(gl_project)/issues?state=opened&in=title,description&search=${Q}&per_page=100" \
  | jq '[.[] | {number: .iid, title, url: .web_url, state: (if .state == "opened" then "OPEN" else "CLOSED" end)}]'
```

#### create-issue
Title, body file, assignee, labels → created issue URL. Labels are applied **after** creation through the guard: GitLab silently creates any unknown label passed at creation time, which would bypass the taxonomy.
```bash
ASSIGNEE_ID=$(glab api "users?username=$(printf '%s' "<username>" | jq -sRr @uri)" | jq -r '.[0].id // empty')
ISSUE=$(jq -n --arg t "<title>" --rawfile d <body-file> --arg a "$ASSIGNEE_ID" \
  '{title: $t, description: $d} + (if $a == "" then {} else {assignee_ids: [($a | tonumber)]} end)' \
  | gl_write POST "projects/$(gl_project)/issues")
ISSUE_ID=$(printf '%s' "$ISSUE" | jq -r '.iid')
ISSUE_URL=$(printf '%s' "$ISSUE" | jq -r '.web_url')
for label in <labels>; do apply_issue_label "$label" "$ISSUE_ID"; done
```

#### close-issue
`{issueId}`, reason, closing comment. GitLab has no close reason; state it in the comment (`completed`, `not planned`, `duplicate of #N`).
```bash
gl_iid {issueId}
jq -n --rawfile b <comment-file> '{body: $b}' | gl_write POST "projects/$(gl_project)/issues/{issueId}/notes" >/dev/null
jq -n '{state_event: "close"}' | gl_write PUT "projects/$(gl_project)/issues/{issueId}" | jq -e '.state == "closed"' >/dev/null
```

#### comment-issue
`{issueId}`, body file → the new note's handle.
```bash
gl_iid {issueId}
jq -n --rawfile b <body-file> '{body: $b}' \
  | gl_write POST "projects/$(gl_project)/issues/{issueId}/notes" | jq -r '"issues/{issueId}/\(.id)"'
```

#### update-issue
`{issueId}`, new title and/or body. Edits only the issue's own fields; pass only what changed.
```bash
gl_iid {issueId}
jq -n --arg t "<title>" '{title: $t}' | gl_write PUT "projects/$(gl_project)/issues/{issueId}" >/dev/null
jq -n --rawfile d <body-file> '{description: $d}' | gl_write PUT "projects/$(gl_project)/issues/{issueId}" >/dev/null
```

#### assign-issue / unassign-issue
`{issueId}`, username. GitLab replaces the whole assignee list, so read it, add or remove the one user, and write it back. On the Free tier an issue holds one assignee: when another user already holds the slot, the add does not take effect — the read-back reports it, and the claim then rests on the `in-progress` label and the claim note.
```bash
# $1 = issues|merge_requests, $2 = iid, $3 = add|remove, $4 = username.
gl_assign() {
  gl_iid "$2" || return 1
  p=$(gl_project) || return 1
  uid=$(glab api "users?username=$(printf '%s' "$4" | jq -sRr @uri)" | jq -r '.[0].id // empty')
  [ -n "$uid" ] || { echo "Unknown GitLab user: $4" >&2; return 1; }
  ids=$(glab api "projects/$p/$1/$2" | jq -c --argjson u "$uid" --arg op "$3" \
    '[(.assignees // [])[].id] | if $op == "add" then (. + [$u] | unique) else map(select(. != $u)) end')
  jq -n --argjson ids "$ids" '{assignee_ids: $ids}' | gl_write PUT "projects/$p/$1/$2" >/dev/null
  if glab api "projects/$p/$1/$2" | jq -e --arg u "$4" 'any((.assignees // [])[]; .username == $u)' >/dev/null; then
    [ "$3" = add ] || { echo "GitLab still lists $4 as an assignee of $1/$2" >&2; return 1; }
  else
    [ "$3" = remove ] || echo "GitLab did not add $4 as an assignee of $1/$2 (the single Free-tier assignee slot is taken); relying on the label and claim note."
  fi
}
gl_assign issues {issueId} add <username>
gl_assign issues {issueId} remove <username>
```

#### label-issue / unlabel-issue
Always through the guards: `apply_issue_label "<label>" {issueId}` / `remove_issue_label "<label>" {issueId}`.

#### get-issue-comment
Comment handle → body, author, URL.
```bash
gl_get_comment() {
  gl_handle "$1" || return 1
  p=$(gl_project) || return 1
  web=$(glab api "projects/$p/${1%/*}" | jq -r '.web_url')
  glab api "projects/$p/${1%/*}/notes/${1##*/}" | jq --arg u "$web#note_${1##*/}" '{body, user: .author.username, url: $u}'
}
gl_get_comment {commentHandle}
```

#### list-issue-comments
Kind (`issues`, or `merge_requests` when the caller is working a PR) and iid → conversation comments, oldest first, with their handles. System notes and inline diff notes are excluded (inline notes come from **list-review-comments**).
```bash
gl_list_comments() {
  gl_iid "$2" || return 1
  gl_list "projects/$(gl_project)/$1/$2/notes?sort=asc" \
    | jq --arg k "$1/$2" '[.[] | select((.system | not) and .type != "DiffNote") | {id: "\($k)/\(.id)", user: .author.username, body, createdAt: .created_at}]'
}
gl_list_comments issues {issueId}
gl_list_comments merge_requests {prNumber}
```

#### update-comment
Comment handle, new body file → the note rewritten in place (issue and MR notes alike). This is how marker-idempotent comments are updated on re-runs: find the `🤖 …` marker via **list-issue-comments**, then update that handle.
```bash
COMMENT_HANDLE={commentHandle}
gl_handle "$COMMENT_HANDLE"
jq -n --rawfile b <body-file> '{body: $b}' \
  | gl_write PUT "projects/$(gl_project)/${COMMENT_HANDLE%/*}/notes/${COMMENT_HANDLE##*/}" >/dev/null
```

### Pull requests

#### get-pr
`{prNumber}` (MR iid) → PR data in the field set `github.md` documents, serialized the same way: `state` `OPEN`/`CLOSED`/`MERGED`, review states `APPROVED`/`CHANGES_REQUESTED`, ISO-8601 timestamps. Select the fields the calling skill names from the object.
```bash
gl_pr_json {prNumber} | jq '{number, title, url, state, isDraft, labels, reviewDecision}'   # select the requested fields
```
Mapping notes: `mergeable` / `mergeStateStatus` derive from GitLab's `detailed_merge_status` (`mergeable` → `CLEAN`, `conflict` → `DIRTY`, `need_rebase` → `BEHIND`, `draft_status` → `DRAFT`, still computing → `UNKNOWN`, every other blocker → `BLOCKED`). `files` carries per-file `additions`/`deletions` counted from the MR diffs; `additions` sums them and `changedFiles` counts them. GitLab collapses very large diffs, which then count as zero — when the size matters, fetch the branch with **checkout-pr** and use `git diff --shortstat`. `commits` carries `oid`, `messageHeadline`, and `authoredDate`. `closingIssuesReferences` comes from GitLab's own closes-issues list.

#### list-prs
State (`opened`, `merged`, `closed`, `all`), optional search text and date bound, limit (≤100) → PRs in the **get-pr** shape, without the heavy fields (`reviews`, `latestReviews`, `commits`, `files`, `comments`, and `additions` are empty or null — call **get-pr** for them). GitLab's `closed` state already means closed without merging.
```bash
# $1 = state, $2 = limit, $3 = optional updated-after ISO date, $4 = optional search text.
gl_list_prs() {
  q="projects/$(gl_project)/merge_requests?state=$1&order_by=updated_at&sort=desc&per_page=${2:-100}"
  [ -n "${3:-}" ] && q="$q&updated_after=$3"
  [ -n "${4:-}" ] && q="$q&search=$(printf '%s' "$4" | jq -sRr @uri)"
  glab api "$q" | jq -r '.[].iid' | while read -r iid; do GL_PR_LIGHT=1 gl_pr_json "$iid"; done | jq -s '.'
}
gl_list_prs opened 100
gl_list_prs merged {limit} "${SINCE_DATE}" | jq --arg d "${SINCE_DATE}" '[.[] | select(.mergedAt >= $d)]'
gl_list_prs closed {limit} "${SINCE_DATE}" | jq --arg d "${SINCE_DATE}" '[.[] | select(.closedAt >= $d)]'
```

#### search-prs
Free-text query and state (`opened`, `merged`, `closed`, `all`) → matching PRs. An issue reference (`#123`) uses GitLab's related-merge-requests list for that issue, which finds MRs that mention or close it; any other query (a plan path, a slug) searches MR titles and descriptions.
```bash
gl_search_prs() {
  p=$(gl_project) || return 1
  case "$1" in
    \#*)
      gl_iid "${1#\#}" || return 1
      gl_list "projects/$p/issues/${1#\#}/related_merge_requests" \
        | jq --arg s "$2" '[.[] | select($s == "all" or .state == $s)]' ;;
    *)
      glab api "projects/$p/merge_requests?state=$2&in=title,description&search=$(printf '%s' "$1" | jq -sRr @uri)&per_page=100" ;;
  esac | jq "$GL_JQ_DEFS"'[.[] | {number: .iid, title, url: .web_url, state: (.state | gl_state)}]'
}
gl_search_prs "#{issueId}" opened
```

#### create-pr
Base branch, draft flag, title, body file → PR. Push the branch first; the source branch is the current one. A draft gets the `Draft:` title prefix.
```bash
PR=$(jq -n --arg s "$(git rev-parse --abbrev-ref HEAD)" --arg b "$BASE_BRANCH" --arg t "<title>" \
  --argjson draft true --rawfile d <body-file> \
  '{source_branch: $s, target_branch: $b, title: (if $draft then "Draft: " + $t else $t end), description: $d, remove_source_branch: false}' \
  | gl_write POST "projects/$(gl_project)/merge_requests")
PR_URL=$(printf '%s' "$PR" | jq -r '.web_url')
PR_NUMBER=$(printf '%s' "$PR" | jq -r '.iid')
```

#### update-pr
`{prNumber}`, new title and/or body file → the MR's own title/description rewritten in place. Pass only what changed. A title update keeps the `Draft:` prefix while the MR is a draft, so it never promotes the MR as a side effect.
```bash
gl_iid {prNumber}
DRAFT=$(glab api "projects/$(gl_project)/merge_requests/{prNumber}" | jq '.draft // false')
jq -n --arg t "<title>" --argjson draft "$DRAFT" \
  "$GL_JQ_DEFS"'{title: (if $draft and ($t | test(gl_draft_re; "i") | not) then "Draft: " + $t else $t end)}' \
  | gl_write PUT "projects/$(gl_project)/merge_requests/{prNumber}" >/dev/null
jq -n --rawfile d <body-file> '{description: $d}' | gl_write PUT "projects/$(gl_project)/merge_requests/{prNumber}" >/dev/null
```

#### comment-pr
`{prNumber}`, body file → the new note's handle.
```bash
gl_iid {prNumber}
jq -n --rawfile b <body-file> '{body: $b}' \
  | gl_write POST "projects/$(gl_project)/merge_requests/{prNumber}/notes" | jq -r '"merge_requests/{prNumber}/\(.id)"'
```

#### attach-image-evidence
`{prNumber}`, a markdown body file (without the images), a `{slug}` (e.g. `pr-{prNumber}`), and local PNG paths → one note with the images embedded **inline**; returns the note URL.

Commit the images to a dedicated slash-free evidence branch (never the MR's own branch) through the Repository Files API, then reference their `/-/raw/` URLs. On a private or internal project those URLs render for viewers signed in to the instance.
```bash
gl_iid {prNumber}
P=$(gl_project)
WEB=$(glab api "projects/$P" | jq -r '.web_url')
DEFAULT_BRANCH=$(glab api "projects/$P" | jq -r '.default_branch')
EVIDENCE_BRANCH="qa-evidence-{slug}"
glab api "projects/$P/repository/branches/${EVIDENCE_BRANCH}" >/dev/null 2>&1 ||
  jq -n --arg b "$EVIDENCE_BRANCH" --arg r "$DEFAULT_BRANCH" '{branch: $b, ref: $r}' \
    | gl_write POST "projects/$P/repository/branches" >/dev/null

# Image bytes never touch a command line: base64 goes to a temp file that jq reads.
BODY_IMAGES=""
EV_TMP=$(mktemp)
for img in <image-paths>; do
  path="{slug}/$(basename "$img")"
  enc_path=$(printf '%s' "$path" | jq -sRr @uri)
  base64 < "$img" | tr -d '\n' > "$EV_TMP"
  jq -n --rawfile c "$EV_TMP" --arg b "$EVIDENCE_BRANCH" --arg m "qa evidence {slug}" \
    '{branch: $b, encoding: "base64", content: $c, commit_message: $m}' > "$EV_TMP.json"
  gl_write POST "projects/$P/repository/files/${enc_path}" < "$EV_TMP.json" >/dev/null 2>&1 ||
    gl_write PUT "projects/$P/repository/files/${enc_path}" < "$EV_TMP.json" >/dev/null
  BODY_IMAGES="${BODY_IMAGES}
![$(basename "$img")](${WEB}/-/raw/${EVIDENCE_BRANCH}/${path})"
done
rm -f "$EV_TMP" "$EV_TMP.json"

{ cat <body-file>; printf '%s\n' "$BODY_IMAGES"; } > "$EV_TMP.body"
NOTE_ID=$(jq -n --rawfile b "$EV_TMP.body" '{body: $b}' | gl_write POST "projects/$P/merge_requests/{prNumber}/notes" | jq -r '.id')
rm -f "$EV_TMP.body"
glab api "projects/$P/merge_requests/{prNumber}" | jq -r --arg n "$NOTE_ID" '"\(.web_url)#note_\($n)"'
```
Fallbacks: when the evidence branch cannot be created or written (no Developer access, a protected-branch rule matching `qa-evidence-*`), post the note with the local artifact paths instead and say inline rendering is unavailable. Never store evidence on the MR's own branch, and never force-push.

#### assign-pr / unassign-pr
The same read-modify-write as **assign-issue**, on the merge request: `gl_assign merge_requests {prNumber} add <username>` / `gl_assign merge_requests {prNumber} remove <username>`.

#### label-pr / unlabel-pr
Always through the guards: `apply_label "<label>" {prNumber}` / `set_pipeline_label {prNumber} "<label>"` for the mutually exclusive pipeline group; direct removal: `remove_label "<label>" {prNumber}`.

#### get-pr-diff
`{prNumber}` → the full unified diff, or the changed-file list.
```bash
gl_iid {prNumber}
gl_list "projects/$(gl_project)/merge_requests/{prNumber}/diffs" | jq -r '.[] |
  "diff --git a/\(.old_path) b/\(.new_path)\n--- \(if .new_file then "/dev/null" else "a/" + .old_path end)\n+++ \(if .deleted_file then "/dev/null" else "b/" + .new_path end)\n\(.diff)"'
gl_list "projects/$(gl_project)/merge_requests/{prNumber}/diffs" | jq -r '.[].new_path'   # name-only
```
GitLab truncates or collapses very large diffs. When a file's `diff` is empty but the file changed, check out the MR (**checkout-pr**) and use `git diff "origin/<baseRefName>...HEAD"`.

#### get-pr-files
`{prNumber}` → changed files with per-file status.
```bash
gl_iid {prNumber}
gl_list "projects/$(gl_project)/merge_requests/{prNumber}/diffs" | jq '[.[] | {path: .new_path,
  status: (if .new_file then "added" elif .deleted_file then "removed" elif .renamed_file then "renamed" else "modified" end)}]'
```

#### checkout-pr
`{prNumber}` → the MR head available locally. GitLab publishes every MR head, fork MRs included, as `refs/merge-requests/<iid>/head` on the target project.
```bash
gl_iid {prNumber}
git fetch origin "refs/merge-requests/{prNumber}/head:mr-{prNumber}"
git checkout "mr-{prNumber}"
```
To push fixes to a fork MR, push to the source project's repository URL and `headRefName` — possible only when `maintainerCanModify` is true. For a same-project MR, check out `headRefName` from `origin` instead.

#### review-pr
`{prNumber}`, verdict (approve / request changes), body file. Approving is GitLab's native approval; requesting changes revokes this user's approval, if any. Both then post the review body as a note that opens with the hidden verdict marker (see Conventions), which is how **get-pr** reads the verdict back.
```bash
gl_review() {
  gl_iid "$1" || return 1
  p=$(gl_project) || return 1
  case "$2" in
    approve)
      glab api -X POST "projects/$p/merge_requests/$1/approve" >/dev/null || {
        echo "GitLab refused the approval: authors may be barred from approving their own MR, or approval rules exclude this user." >&2
        return 1
      }
      marker='<!-- review: APPROVED -->' ;;
    request-changes)
      glab api -X POST "projects/$p/merge_requests/$1/unapprove" >/dev/null 2>&1 || true
      marker='<!-- review: CHANGES_REQUESTED -->' ;;
    *) echo "Unknown review verdict: $2" >&2; return 1 ;;
  esac
  { printf '%s\n\n' "$marker"; cat "$3"; } | jq -Rs '{body: .}' \
    | gl_write POST "projects/$p/merge_requests/$1/notes" | jq -r --arg k "merge_requests/$1" '"\($k)/\(.id)"'
}
gl_review {prNumber} approve <body-file>
gl_review {prNumber} request-changes <body-file>
```
Surface a refused self-approval instead of working around it, exactly as on GitHub.

#### merge-pr
`{prNumber}`; squash by default. Auto-merge (merge once the pipeline succeeds) only when the skill asks for it; delete the source branch only when asked.
```bash
gl_iid {prNumber}
jq -n '{squash: true}' | gl_write PUT "projects/$(gl_project)/merge_requests/{prNumber}/merge" | jq -e '.state == "merged"' >/dev/null
jq -n '{squash: true, merge_when_pipeline_succeeds: true}' | gl_write PUT "projects/$(gl_project)/merge_requests/{prNumber}/merge" >/dev/null   # auto-merge
```
A project whose squash setting is "Do not allow" rejects `squash: true`; surface that rather than merging differently.

#### mark-pr-ready
Promote a draft MR by stripping the draft prefix from its title, then read it back.
```bash
gl_iid {prNumber}
glab api "projects/$(gl_project)/merge_requests/{prNumber}" \
  | jq "$GL_JQ_DEFS"'{title: (.title | sub(gl_draft_re; ""; "i"))}' \
  | gl_write PUT "projects/$(gl_project)/merge_requests/{prNumber}" >/dev/null
glab api "projects/$(gl_project)/merge_requests/{prNumber}" | jq -e '(.draft // false) == false' >/dev/null
```

#### get-pr-checks
`{prNumber}` → the jobs and downstream-pipeline bridges of the MR's head pipeline, with `name`, `state`, `bucket` (`pass`/`fail`/`pending`/`skipping`/`cancel`, the same buckets `github.md` reports), `link`, and the stage as `workflow`. No head pipeline means no CI ran: an empty list.
```bash
gl_pr_checks() {
  gl_iid "$1" || return 1
  p=$(gl_project) || return 1
  pid=$(glab api "projects/$p/merge_requests/$1" | jq -r '.head_pipeline.id // empty')
  [ -n "$pid" ] || { echo '[]'; return 0; }
  { gl_list "projects/$p/pipelines/$pid/jobs"; gl_list "projects/$p/pipelines/$pid/bridges"; } \
    | jq -s "$GL_JQ_DEFS"'add | map(gl_check)'
}
gl_pr_checks {prNumber}
```
A failed job marked `allow_failure: true` reports `NEUTRAL` (bucket `pass`); a manual job reports `PENDING` only when it blocks the pipeline.

#### get-required-checks
GitLab has no per-branch list of required checks. The merge rule is the project's "Pipelines must succeed" setting, which a failing non-`allow_failure` job violates. This operation therefore prints no check names — the contract's "unreadable" case — and every check **get-pr-checks** reports counts as required. Allow-failure jobs are already `NEUTRAL` there, so they never block. Print the project setting for the record:
```bash
glab api "projects/$(gl_project)" | jq -r '"only_allow_merge_if_pipeline_succeeds=\(.only_allow_merge_if_pipeline_succeeds)"' >&2
```
External status checks (Ultimate) are not included; a team that uses them extends this operation in its copy.

#### get-pr-comment / get-review-comment
Comment handle → body, author, URL. Conversation notes and inline diff notes share one API on GitLab, so both use `gl_get_comment` from **get-issue-comment**: `gl_get_comment merge_requests/{prNumber}/{noteId}`. A pasted `…/-/merge_requests/<iid>#note_<id>` link maps onto that handle.

#### list-review-comments
`{prNumber}` → every inline diff note on the MR (file, line, author, body), with GitLab's thread `resolved` state — which GitHub's REST API does not expose. `reply_to` is the handle of the first note in the thread for replies, `null` for the note that opened it.
```bash
gl_iid {prNumber}
WEB=$(glab api "projects/$(gl_project)/merge_requests/{prNumber}" | jq -r '.web_url')
gl_list "projects/$(gl_project)/merge_requests/{prNumber}/discussions" \
  | jq --arg k "merge_requests/{prNumber}" --arg w "$WEB" '[.[] | .notes as $ns | $ns[] | select(.type == "DiffNote")
    | {id: "\($k)/\(.id)", user: .author.username, path: (.position.new_path // .position.old_path),
       line: (.position.new_line // .position.old_line), body, url: "\($w)#note_\(.id)",
       reply_to: (if .id == $ns[0].id then null else "\($k)/\($ns[0].id)" end), resolved: (.resolved // false)}]'
```

### CI runs

CI status for an *MR* comes from **get-pr-checks** above. These operations address **pipelines** directly — a GitLab pipeline is a "run" here, and its id is the `{runId}`.

#### list-runs
Branch (or head SHA) → recent pipelines with `databaseId`, `workflowName` (the pipeline source, e.g. `push`, `merge_request_event`), `status` (`queued`/`in_progress`/`completed`), and `conclusion` (`success`/`failure`/`cancelled`/`skipped`/`action_required`).
```bash
glab api "projects/$(gl_project)/pipelines?ref=$(printf '%s' "{branch}" | jq -sRr @uri)&order_by=id&sort=desc&per_page=20" \
  | jq "$GL_JQ_DEFS"'map(gl_run)'
glab api "projects/$(gl_project)/pipelines?sha={headSha}&order_by=id&sort=desc&per_page=20" | jq "$GL_JQ_DEFS"'map(gl_run)'
```

#### get-run
Pipeline id → status, conclusion, and per-job breakdown.
```bash
gl_iid {runId}
P=$(gl_project)
JOBS=$(mktemp)
gl_list "projects/$P/pipelines/{runId}/jobs" > "$JOBS"
glab api "projects/$P/pipelines/{runId}" | jq --slurpfile jobs "$JOBS" "$GL_JQ_DEFS"'gl_run + {jobs: [$jobs[0][]
  | {databaseId: .id, name, stage, url: .web_url, allowFailure: .allow_failure} + gl_run_status]}'
rm -f "$JOBS"
```

#### get-run-failed-logs
Pipeline id → the trace of each failed job (the last 400 lines of each; failures are at the end). This is the primary diagnosis input for CI failures.
```bash
gl_iid {runId}
P=$(gl_project)
gl_list "projects/$P/pipelines/{runId}/jobs?scope=failed" | jq -r '.[] | "\(.id)\t\(.name)\t\(.allow_failure)"' \
  | while IFS="$(printf '\t')" read -r job name allowed; do
      printf '=== job %s (#%s, allow_failure=%s) ===\n' "$name" "$job" "$allowed"
      glab api "projects/$P/jobs/$job/trace" | tail -n 400
    done
```

#### rerun-failed
Pipeline id → retry only its failed and canceled jobs. Use to disambiguate flaky failures before changing any code.
```bash
gl_iid {runId}
glab api -X POST "projects/$(gl_project)/pipelines/{runId}/retry" | jq "$GL_JQ_DEFS"'gl_run'
```

#### watch-run
Pipeline id → block until the pipeline finishes; exit non-zero unless it succeeded. `glab` has no blocking watch for an arbitrary pipeline, so this polls **get-run** within the caller's wait budget (`ci.maxWaitMinutes`, default 40).
```bash
gl_iid {runId}
DEADLINE=$(( $(date +%s) + ${CI_MAX_WAIT_MINUTES:-40} * 60 ))
while :; do
  RUN=$(glab api "projects/$(gl_project)/pipelines/{runId}" | jq -c "$GL_JQ_DEFS"'gl_run')
  [ "$(printf '%s' "$RUN" | jq -r .status)" = completed ] && break
  [ "$(date +%s)" -ge "$DEADLINE" ] && { echo "Pipeline {runId} still running after the wait budget" >&2; exit 2; }
  sleep 30
done
printf '%s' "$RUN" | jq -e '.conclusion == "success"' >/dev/null
```

### Labels

#### list-labels
→ every label name available to the project, including labels inherited from its groups.
```bash
gl_list "projects/$(gl_project)/labels" | jq -r '.[].name'
```

#### create-label
Name, color (`#rrggbb`), description. Creates a project label; teams that share a taxonomy across projects may create the same names as group labels instead. Never delete, rename, or recolor existing labels.
```bash
gl_create_label() {
  jq -n --arg n "$1" --arg c "$2" --arg d "$3" '{name: $n, color: $c, description: $d}' \
    | gl_write POST "projects/$(gl_project)/labels" >/dev/null
}
gl_create_label "<name>" "#<hex>" "<description>"
```

#### ensure-label-taxonomy
Create every label from the config's taxonomy that does not exist yet (used by `om-setup-agent-pipeline`; skips ones **list-labels** already returns):
```bash
EXISTING=$(gl_list "projects/$(gl_project)/labels" | jq -r '.[].name')
while IFS='|' read -r name color description; do
  [ -n "$name" ] || continue
  printf '%s\n' "$EXISTING" | grep -Fxq "$name" || gl_create_label "$name" "$color" "$description"
done <<'EOF'
review|#0366d6|Ready for code review
changes-requested|#b60205|Reviewer requested changes
qa|#fbca04|Manual QA in progress
qa-failed|#b60205|Manual QA failed
merge-queue|#0e8a16|Approved, ready to merge
blocked|#b60205|Blocked by a dependency
do-not-merge|#b60205|Hard merge block
bug|#d73a4a|Bug fix
feature|#a2eeef|New capability
refactor|#cfd3d7|No behavior change
security|#b60205|Security-relevant change
dependencies|#0366d6|Dependency update
documentation|#0075ca|Docs only
needs-qa|#fbca04|Requires manual QA before merge
skip-qa|#0e8a16|Low risk, QA not required
qa-approved|#0e8a16|Manual QA passed
qa-self-verified|#c5def5|Self-QA exception used
in-progress|#c5def5|An automated skill is working on this
ci-monitoring|#d4c5f9|Work complete and reported; agent is watching CI results
do-not-close|#c5def5|Humans only: never auto-close this issue
priority-low|#e4e669|Cosmetic or follow-up work
priority-medium|#fbca04|Ordinary bug or feature
priority-high|#d93f0b|Release-blocking
priority-extreme|#b60205|Outage or security incident
risk-low|#0e8a16|Isolated, low blast radius
risk-medium|#fbca04|Ordinary change with tests
risk-high|#b60205|Wide blast radius, review deeply
EOF
```
