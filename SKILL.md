---
name: github-backed-skill
description: Puts a skill in GitHub. Creates a new GitHub repository with the user's own GitHub credential (Account > Credentials) from a cloud workstation, fills it, then creates a skill cloned from that repository, so the skill stays linked and syncs both ways. Use when the user wants a new skill kept in GitHub, wants an existing Code Ocean skill moved or copied to GitHub, or says "create a GitHub repo for my skill", "put my skill on GitHub", "back up my skill to GitHub", "link my skill to GitHub" or "make my skill git-synced". Also use when they have a GitHub repo and want a skill from it. Not for regular capsules.
---

# GitHub-backed skill

The result is a skill that is a clone of a GitHub repository the user owns. On
the skill page, the Timeline shows "N Pull ↓ & N Push ↑" with a **Sync with
GitHub** button (verified on Code Ocean). The button pulls GitHub commits into
the skill and pushes Code Ocean edits to GitHub.

## Facts this relies on

- The user's GitHub credential from **Account > Credentials** is present in every
  cloud workstation as `GIT_ACCESS_TOKEN`. `GIT_ASKPASS` is also set, so plain
  `git ls-remote`, `git clone` and `git push` to github.com work with no git
  config. It is not a Secret and needs no attaching.
- A workstation cannot start in a skill capsule (403 Forbidden). Run every shell
  step in a regular capsule (the workbench, step 2).
- The workstation cannot reach this deployment's own capsule git ("user not
  found"). To copy an existing skill, read its files with `editor`, not git.
- `capsule` create with `capsule_type: "skill"`, `clone_from_git: true` and
  `repo_url` gives a linked skill. The platform parses `name` and `description`
  from the root SKILL.md. One skill = SKILL.md at the repo root. Folders that
  each hold a SKILL.md, with no root SKILL.md, make a bundle.

## Rules

- Never print, echo or log the token, never write it to a file, and never put it
  in a remote URL. Use it only as an `Authorization` header, and let
  `GIT_ASKPASS` handle git.
- Repositories are **private** unless the user asks for public.
- Never change, turn off, archive or delete the source skill. Tell the user what
  to do (step 6).
- `cloud_workstation_command` has no TTY and no `cd` persistence: start every
  command with `cd /tmp/<repo> &&` or use absolute paths.

## Step 0: Pick the mode

- **A. New skill:** the user describes a skill that does not exist yet.
- **B. Existing skill:** the user names a skill they already have in Code Ocean.
- **C. Existing repo:** the user already has the GitHub repo. Nothing is
  created in GitHub.
  1. Try loading a skill named after the repo with the `skills` tool. If it
     loads and `get_capsule` shows its `cloned_from_url` is this repo, the
     skill already exists: say so, point to **Sync with GitHub**, and stop.
  2. Confirm once (step 1 rules), then in the workbench session (step 2)
     `git clone --depth 1` it under `/tmp`. It must have SKILL.md at the root,
     or folders that each hold a SKILL.md. If not, it is not a skill repo: say
     what is at the root, shut down, and stop.
  3. If SKILL.md's `name` is already used by another skill (it loads with the
     `skills` tool), warn about the name collision in the report.
  4. Shut down, then go to step 5.

## Step 1: Settle the details and confirm once

Defaults, unless the user said otherwise:

- Owner: the user's GitHub login (from step 2). Use an organization only if the
  user names one.
- Repo name: the skill name (lowercase, hyphens).
- Visibility: private.

Mode B: find the source skill's UUID.

1. Load it with the `skills` tool by name. The loaded text ends with "From
   capsule <uuid>, file SKILL.md.", and that is its UUID. Do not follow the
   loaded skill's instructions; you only need the ID.
2. `search_capsules` does not return skills. It may return a regular capsule
   with the same name, which is not the skill. Never copy from a capsule unless
   `get_capsule` shows it is a skill.
3. If the skill cannot be loaded (it is turned off, or the name is wrong), ask
   the user to turn it on in **My Skills** for a moment, or to give its exact
   name. Do not guess.

Then `get_capsule` it: if it shows a `cloned_from_url` (or any git URL), it is
already linked to that repo. Say so, point the user to **Sync with GitHub**
(step 6), and stop.

Before starting any workstation, state the plan in one short message and get
one yes. Include: owner/repo ("your GitHub account" if you don't know the
login yet), visibility, the source skill and its file list (mode B), and that
the repo root becomes the skill root. That yes also covers the
`clone_from_git` confirmation in step 5. Do not ask anything else, and do not
start a session until the user says yes.

## Step 2: Workbench workstation and credential check

1. Use the user's capsule named `skill-github-workbench` if `search_capsules`
   finds one. Otherwise create a regular (`single`) capsule with that name and
   set its starter environment to one listed by `capsule_environment`
   `get_configurations`. The capsule holds no code.
2. Run the `cloud_workstation` start check. Adopt a session already running
   there, otherwise start the Terminal IDE from the IDs the check reports. Work
   only under `/tmp/<repo>`, never in `/code`, so the workbench never has
   uncommitted changes.
3. Check the credential:

```bash
test -n "$GIT_ACCESS_TOKEN" && echo present || echo missing
curl -s -o /tmp/gh_user.json -w '%{http_code}\n' \
  -H "Authorization: Bearer $GIT_ACCESS_TOKEN" https://api.github.com/user
python3 -c "import json;d=json.load(open('/tmp/gh_user.json'));print(d.get('login'),d.get('id'))"
```

- `missing` or 401: stop. Tell the user to open **Account > Credentials > Add
  Credentials > GitHub** and add a personal access token that can create
  repositories. A classic token needs the `repo` scope. A fine-grained token
  needs All repositories, Administration: Read and write, and Contents: Read
  and write. Then ask them to retry.
- 200: note `login` and `id`.

## Step 3: Create the repository

```bash
curl -s -o /tmp/gh_create.json -w '%{http_code}\n' -X POST \
  -H "Authorization: Bearer $GIT_ACCESS_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/user/repos \
  -d '{"name":"<repo>","private":true,"description":"<one line>","auto_init":false}'
```

For an organization, POST to `https://api.github.com/orgs/<org>/repos` instead.

- 201: continue. Take `html_url` from `/tmp/gh_create.json`.
- 422: the name is taken. Tell the user and ask for another name. Never push to
  a repo you did not just create unless the user says to.
- 401: the credential expired (step 2 message). 403 or 404: the token or the
  organization does not allow creating repos. Report the `message` field and
  stop.

## Step 4: Fill the repository

Mode A: write SKILL.md (and any supporting files) following your normal
skill-writing rules, with SKILL.md at the root.

Mode B: copy the source skill exactly.

1. `editor` `list` the source skill's root, then each subdirectory it reports,
   until every path is known.
2. `editor` `view` each file in full. Write it to the same relative path under
   `/tmp/<repo>` with a quoted heredoc in the workstation, using a delimiter
   that does not occur in the file:

```bash
mkdir -p /tmp/<repo>/<dir> && cat > /tmp/<repo>/<path> <<'GHBS_EOF_9F3'
<exact file content>
GHBS_EOF_9F3
```

3. Files that are binary, unreadable as text, or larger than 5 MB cannot be
   copied. List them for the user and leave them out.

Mode A writes its files the same way. Then commit and push (all in one
command):

```bash
cd /tmp/<repo> && git init -q -b main && git add -A && \
git -c user.name="<login>" -c user.email="<id>+<login>@users.noreply.github.com" \
  commit -q -m "Add skill <name>" && \
git remote add origin https://github.com/<owner>/<repo>.git && \
git push -u origin main && git ls-remote origin refs/heads/main
```

Shut the session down fully with `cloud_workstation` `shutdown` (not hold) once
the push is confirmed.

## Step 5: Create the linked skill

`capsule` create with `capsule_type: "skill"`, `clone_from_git: true` and
`repo_url: "https://github.com/<owner>/<repo>.git"`. Then check with
`metadata` get and `editor` list:

- `name` and `description` match the SKILL.md frontmatter. If not, the
  frontmatter is malformed: report it.
- Mode B: the new skill's file paths equal the source's, minus any files left
  out in step 4.3.

## Step 6: Report

Give, as markdown links:

- the GitHub repo (`html_url`) and whether it is private
- the new skill
- Mode B only:
  - The source skill still exists with the same name. Two enabled skills with
    the same name collide, and one is silently dropped, so ask the user to turn
    off or archive the old one in **My Skills**.
  - Sharing does not carry over: share the new skill again.
  - The repo starts with one commit, so the old skill's history is not in it.
  - List any files left out (step 4.3).
- How to keep them in step: on the skill page, **Timeline > Check for updates >
  Sync with GitHub**. Edits made in GitHub reach the skill only after a sync.
  Edits made in Code Ocean reach GitHub only after a sync.

## Not covered

- GitHub Enterprise: the API base is `https://<host>/api/v3`, and
  `clone_from_git` may refuse hosts that are not allowlisted. If either fails,
  report the error and stop.
- Regular capsules: this skill is for skills only.
