# infraguide-org lab conventions

How lab repositories in this organization are structured, named and used. The rules apply to every lab (`lab-openstack`, `lab-scs`, and future ones).

This repository is a GitHub template. Start a new lab with "Use this template" and replace this README with the one of the lab.

## Principles

1. **One entry repository per lab.** It is the only repository a newcomer has to clone. It contains documentation, helper scripts and the list of satellite repositories, nothing else.
2. **One satellite repository per unit that a tool expects at its repository root.** Examples: the OSISM configuration repository, a Terraform root module, an Ansible collection, an image build project. Such a unit cannot live in a subfolder of another repository.
3. **Satellites are pinned.** The entry repository records the exact commit of every satellite, so a lab checkout is reproducible.
4. **Dependencies point one way.** The entry repository knows its satellites. A satellite never depends on the entry repository, so it can be reused by another lab.
5. **Secrets stay out of public repositories.** Repositories that contain secrets, even encrypted ones, are private.

## Repository types and naming

| Kind | Pattern | Examples |
|---|---|---|
| Entry repository | `lab-<topic>` | `lab-openstack`, `lab-scs` |
| Satellite of one lab | `lab-<topic>-<role>` | `lab-scs-config`, `lab-scs-artifacts` |
| Several configurations in one lab (one per tool) | `lab-<topic>-config-<component>` | `lab-scs-config-openstack`, `lab-scs-config-ceph` |
| Shared by several labs | `lab-common-<role>` | `lab-common-terraform`, `lab-common-scripts` |

Name a satellite by its **role**, not by the tool that fills the role today. Roles in use:

| Role | Content |
|---|---|
| `config` | configuration of the deployment tool (for example the OSISM configuration repository, Kolla-Ansible configuration) |
| `artifacts` | image build definitions, checksums, release manifests (no large binaries) |
| `modules` | Terraform or Ansible modules |
| `secrets` | secret material (private) |

## Layout of an entry repository

```
lab-<topic>/
├── .github/workflows/slides.yml  # builds the slides and the lab page
├── CLAUDE.md          # points to CONTRIBUTING.md
├── CONTRIBUTING.md    # contribution rules, commit message format
├── LICENSE            # MIT License
├── README.md          # what the lab is, repository layout, slides
├── conf/              # env.yaml, net-diagram.yaml, tmuxp.yaml, user-data.yaml
├── deploy/            # tool configuration, one folder per tool, or a submodule
├── modules/           # additional modules (automation, testing), or a submodule
├── scripts/           # create-env.sh and glue scripts across the satellites
├── slides/            # slide sources (src) and build scripts (scripts)
└── status/            # build.yml, result of the automatic lab build
```

Optional:
* `Makefile` with `init`, `update` and `status`, when the lab has satellites
* `artifacts/` as a submodule
* `tests/` with lab specific tests, `scripts/create-env.sh` copies them next to the common ones

Use the same directory name per role in every lab, so that scripts and documentation carry over from lab to lab. `deploy/` holds what gets deployed.

Current labs:

| Lab | `deploy/` |
|---|---|
| `lab-ceph` | plain folder, `cephadm/` |
| `lab-openstack` | plain folder, `kolla-ansible/` and `cephadm/` |
| `lab-scs` | submodule `lab-scs-config` (OSISM configuration repository; OSISM expects `environments/` and `inventory/` at the repository root) |

## Quick start for any lab

```bash
git clone --recurse-submodules https://github.com/infraguide-org/lab-<topic>.git
cd lab-<topic>
```

Without `--recurse-submodules` (or `make init` in a lab with a `Makefile`):

```bash
git submodule update --init --recursive
```

Minimal `Makefile` for a lab with satellites:

```make
init:    ; git submodule update --init --recursive
update:  ; git submodule update --remote --merge
status:  ; git submodule status; git status --short
```

## Working with submodules

Show the state of all submodules (a leading `+` means that the checked-out commit differs from the recorded one):

```bash
git submodule status
```

Change a satellite (two commits, because there are two repositories):

```bash
cd deploy
git checkout main                       # a submodule is on a detached commit by default
# edit files
git add -A && git commit -m "upd: <file> - <what changed>"
git push                                # 1. push inside the submodule first
cd ..
git add deploy                          # 2. record the new commit in the entry repository
git commit -m "upd: deploy - <what changed>"
git push
```

Always push the satellite before the entry repository. Otherwise others get `fatal: reference is not a tree`.

Take over the newest commit of a satellite:

```bash
git submodule update --remote deploy
git add deploy && git commit -m "upd: deploy - newest commit of the satellite"
```

## Adding a satellite to a lab

1. Create the repository in `infraguide-org` following the naming conventions above.
2. Add it as a submodule at a path named after its role. A relative URL follows the protocol that was used to clone the entry repository (HTTPS or SSH):

   ```bash
   git submodule add ../lab-<topic>-<role>.git <role>
   git commit -m "add: <role> - lab-<topic>-<role> as submodule"
   ```

3. Add it to the layout table of the lab's README.

Do not store large binaries (images, ISO files) in a satellite. Keep definitions, build scripts and checksums in Git and publish the files elsewhere (web server, release, registry).

## Several configurations in one lab

Use one configuration repository per tool that expects its own repository root. If a single tool deploys several components (for example OSISM for OpenStack and Ceph), one `lab-<topic>-config` is enough. If different tools are involved, use one repository per tool and mount them below `deploy/`:

```
deploy/
├── openstack/    # submodule -> lab-<topic>-config-openstack
└── ceph/         # submodule -> lab-<topic>-config-ceph
```

```bash
git submodule add ../lab-<topic>-config-openstack.git deploy/openstack
git submodule add ../lab-<topic>-config-ceph.git deploy/ceph
```

Split by component only when the tools differ. Parts that share inventory, host names or secrets are easier to keep consistent in one repository.

## Secrets and visibility

* Entry repositories are public. Satellites with secrets are private. Without access, the submodule folder stays empty for visitors.
* Encrypted secrets (for example Ansible Vault files) belong in a private repository. The vault password is never part of a repository. Store it in a password manager.
* Repositories that hold only configuration without secrets can be public.

## Creating a new lab

1. Create `lab-<topic>` from this template ("Use this template" on GitHub). Replace `lab-template` with `lab-<topic>` in all files, "Template Lab" in `slides/scripts/make-index.sh`, and fill `conf/` (the files are an example taken from `lab-ceph`). Replace this README with the one of the lab.
2. Create one satellite per unit that needs its own repository root (`lab-<topic>-<role>`).
3. Add the satellites as submodules at their role directories.
4. Test from scratch: clone with `--recurse-submodules` into an empty directory.

## Moving a configuration out of an entry repository

Use this when a tool needs its own repository root, for example when an existing `lab-<topic>` repository really is the tool configuration. Order matters, because GitHub redirects the old repository name only until a new repository takes that name.

1. Rename the existing repository:

   ```bash
   gh repo rename lab-<topic>-config --repo infraguide-org/lab-<topic>
   ```

2. Repoint everything that refers to the old name **before** creating a new repository: working copies (`git remote set-url origin ...`), servers that clone the repository (for example the OSISM manager and its configuration settings), CI files and documentation.
3. Create the new entry repository with the old name:

   ```bash
   gh repo create infraguide-org/lab-<topic> --public --clone
   ```

4. Add the configuration as a submodule and test from scratch:

   ```bash
   git submodule add ../lab-<topic>-config.git deploy
   git commit -m "add: deploy - configuration as submodule" && git push -u origin main
   git clone --recurse-submodules https://github.com/infraguide-org/lab-<topic>.git /tmp/lab-test
   ```
