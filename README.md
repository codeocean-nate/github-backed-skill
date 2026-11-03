# github-backed-skill

An Aqua skill for Code Ocean that puts skills in GitHub. Aqua creates a GitHub
repository with your own GitHub credential, fills it, and then creates a skill
cloned from that repository. The skill stays linked to the repo and syncs both
ways from the skill page (**Timeline > Check for updates > Sync with GitHub**).

## What you can ask Aqua

- **A new skill kept in GitHub:** "I want a new skill called my-skill, kept in
  GitHub. It should ..."
- **An existing skill moved to GitHub:** "Put my skill my-skill on GitHub."
- **A skill from a repo you already have:** "Make a skill from my GitHub repo
  github.com/me/my-skill."

Aqua shows the plan and asks once before it starts any work. Repositories are
private unless you ask for public.

## Install

In Code Ocean, create a new skill from this Git repository, or ask Aqua:

> Create a skill cloned from https://github.com/codeocean-nate/github-backed-skill

The repository root is the skill root (SKILL.md at the top level).

## Requirements

- A GitHub credential under **Account > Credentials > GitHub**, with a personal
  access token that can create repositories:
  - classic token: the `repo` scope
  - fine-grained token: All repositories, Administration (read and write) and
    Contents (read and write)
- The deployment can reach github.com and api.github.com.

## How it works

- Your GitHub credential is available inside cloud workstations, so Aqua runs
  the GitHub steps in a small helper capsule, `skill-github-workbench`, which it
  creates the first time. Aqua never prints or stores the token.
- For an existing skill, Aqua reads its files and writes them to the new repo.

## Limits

- **Moving an existing skill:**
  - The old skill keeps its name, so turn it off after the move, or the two
    collide.
  - Sharing doesn't carry over: share the new skill again.
  - History doesn't carry over: the repo starts with one commit.
- **Files:** binary files and files over 5 MB can't be copied this way. Aqua
  lists any it leaves out.
- **Tested setup:** github.com only. GitHub Enterprise and organization-owned
  repositories have not been tested.
