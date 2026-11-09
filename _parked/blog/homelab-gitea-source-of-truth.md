---
title: "Gitea as the source of truth, GitHub as a mirror"
date: 2026-10-23
description: "Why my clusters read from a self-hosted Git server, how the GitHub mirror and the CI runners fit together, and the one mistake that erased a merge."
image: /img/og-gitea-gitops.png
categories: ["Homelab", "DevOps"]
tags: ["gitea", "gitops", "argocd", "ci-cd", "runners"]
---

My lab clusters are not exposed to the internet, so ArgoCD cannot pull from GitHub. I run **Gitea** as the source of truth and keep GitHub as a **push mirror**, which gives me an off-site copy without opening any inbound path. It is the same shape I meet at clients: an internal Git server that the clusters trust, and an external copy that nothing reads.

## Prerequisites

| What | Needed | Why |
|---|---|---|
| Git server | One VM or container host, 2 vCPU, 4 GiB RAM, 50 GB disk, Gitea 1.22+ with Actions enabled | Holds repositories, pull requests and the CI queue. |
| Runner hosts | Two Linux hosts, 2 vCPU and 4 GiB RAM free for jobs, rootless Podman | One job at a time per runner, capped by systemd. |
| Network | Clusters and runners reach Gitea over HTTPS with a certificate they trust | ArgoCD and the runners pull from it. |
| Outbound internet | Only from Gitea to `github.com` (push mirror) and from runners to the collection and image registries | Clusters need no inbound or outbound internet. |
| Accounts | A GitHub token that can create repositories and push, a Gitea token with `write:repository` | Used by the mirror playbook. |
| Runner registration | A runner token per scope (user or organisation) | Registers `act_runner` against Gitea. |
| Control node | Ansible 2.16+ with `ansible.posix` | Runs the mirror and runner playbooks. |

## How it fits together

![Engineers open pull requests in Gitea; two rootless runners run CI and CD; ArgoCD deploys a pinned commit to clusters that are not on the internet; GitHub receives a push mirror](/img/gitea-gitops.svg)

<p class="figsrc">Editable source: <a href="/img/gitea-gitops.excalidraw" download>gitea-gitops.excalidraw</a>, opens in <a href="https://excalidraw.com">excalidraw.com</a>.</p>

- Repositories live in Gitea. A playbook configures the mirrors to GitHub.
- ArgoCD reads Gitea. CI and CD run on two small Gitea runners (rootless Podman, capped CPU and memory per job).
- CI is incremental: a script picks the changed roles and playbooks and lints only those and the things that use them. CD deploys on merge.

## Code structure and variables

```text
ansible-automation-platform/
├── .gitea/workflows/
│   ├── ci.yml                 # lint only what changed
│   └── gitops.yml             # deploy on merge
├── playbook_gitea_mirrors.yml # mirrors, Actions secrets, Actions on or off per repo
├── playbook_linux_gitea_runner.yml
└── roles/role_linux_gitea_runner/
    ├── defaults/main.yml
    ├── handlers/main.yml
    ├── meta/argument_specs.yml    # the variables are validated
    ├── tasks/main.yml             # user, binary, registration, systemd unit
    └── templates/act-runner.service.j2
```

```yaml
# playbook_gitea_mirrors.yml (vars, shortened)
gitea_api: https://gitea.example.com/api/v1
gitea_owner: myuser
github_owner: myuser
gitea_mirrors:
  - {name: platform-automation, private: true}
  - {name: reusable-workflows,  private: false}  # called by other repositories
gitea_actions_disabled: [legacy-github-only-repo]
```

| Variable | Default | Meaning |
|---|---|---|
| `gitea_runner_version` | `0.6.1` | Pinned `act_runner` release (checksum verified). |
| `gitea_runner_instance` | none | URL of the Gitea server. |
| `gitea_runner_scope` | none | User or organisation the runner serves. |
| `gitea_runner_labels` | `molecule:host` | Labels jobs ask for. |
| `gitea_runner_user` | `gitea-runner` | Unprivileged account. |
| `gitea_runner_capacity` | `1` | Jobs at the same time. |
| `gitea_runner_memory_max`, `gitea_runner_cpu_quota` | `4G`, `200%` | systemd limits per runner. |
| `gitea_mirrors` | list | Repositories to mirror and their visibility. |
| `gitea_actions_disabled` | list | Repositories where Actions stay off. |

## The CI in a file

Gitea Actions reads the GitHub Actions syntax. The repository's CI has two jobs, not more, because the two runners hold one job each and a third job would wait while holding a runner. The first job runs the pre-commit hooks and a secret scan, but only on what the change brings:

```yaml
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  schedule:
    - cron: "0 3 * * *"        # the full run happens at night
  workflow_dispatch:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

The range of commits depends on the event, and a new branch or a force push has no usable base, so it falls back to every file:

```bash
if [ "${EVENT}" = pull_request ]; then
  args=(--from-ref "$(git merge-base "${PR_BASE}" "${PR_HEAD}")" --to-ref "${PR_HEAD}")
elif [ "${EVENT}" = push ] && git cat-file -e "${PUSH_BEFORE}^{commit}" 2>/dev/null; then
  args=(--from-ref "${PUSH_BEFORE}" --to-ref "${SHA}")
else
  args=(--all-files)
fi
pre-commit run "${args[@]}" --show-diff-on-failure
```

The second job runs `ansible-lint` on the playbooks and roles the change touches, and on everything every night. A role used by a playbook that the change did not touch is therefore caught the same night. Third-party tools are pinned: a version and the SHA-256 from the release checksums, changed together.

## The rule, and why it exists

**Merge pull requests in Gitea, never on GitHub.** The mirror force-pushes `main`, so a merge done on the GitHub side is silently erased by the next sync. I learned this the hard way once. Now the rule is written down, branch protection is on in Gitea, and my tooling opens pull requests on Gitea only.

## Smaller lessons

- Gitea reads `.gitea/workflows` **instead of** `.github/workflows` when the first exists, so the two systems can have different pipelines in one repository.
- A job that is waiting still holds a runner. With only two runners, a stuck job blocks everything behind it, so every job has a `timeout-minutes`.
- ArgoCD is pinned to a commit that the pipeline moves after the checks pass, so a push to `main` does not deploy by itself.
- When cleaning branches, delete only those with no commits ahead of `main`, and archive unmerged ones as tags first. Gitea rejects the whole batch if it includes the default branch.

## Takeaway

For GitOps in an isolated environment, choose one place where changes are merged and treat every other copy as read-only. The same rule applies in client networks with an internal Git server and an external mirror.
