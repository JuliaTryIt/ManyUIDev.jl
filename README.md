# ManyUIDev.jl

Development superproject for the ManyUI ecosystem. This repository contains no
Julia code of its own — it is a thin shell that pins every ManyUI package as a
git submodule so the whole stack can be checked out, branched, and tested
together.

## Packages

| Submodule       | Repository                                                              | Role                                          |
| --------------- | ----------------------------------------------------------------------- | --------------------------------------------- |
| `ManyUI`        | [s-celles/ManyUI.jl](https://github.com/s-celles/ManyUI.jl)               | Core widget tree and backend-agnostic API      |
| `ManyUICLI`     | [s-celles/ManyUICLI.jl](https://github.com/s-celles/ManyUICLI.jl)         | CLI projection of a `ManyUI.Widget` tree       |
| `ManyUITUI`     | [s-celles/ManyUITUI.jl](https://github.com/s-celles/ManyUITUI.jl)         | Text user interface backend                    |
| `ManyUIWeb`     | [s-celles/ManyUIWeb.jl](https://github.com/s-celles/ManyUIWeb.jl)         | Web bridge (web terminal and web native)       |
| `ManyUICImGui`  | [s-celles/ManyUICImGui.jl](https://github.com/s-celles/ManyUICImGui.jl)   | CImGui backend                                 |
| `ManyUIDemos`   | [s-celles/ManyUIDemos.jl](https://github.com/s-celles/ManyUIDemos.jl)     | Centralized demonstration hub                  |
| `ManyUIDoc`     | [s-celles/ManyUIDoc.jl](https://github.com/s-celles/ManyUIDoc.jl)         | Ecosystem documentation                        |

Every submodule tracks `main` and is referenced over SSH.

## Getting started

```sh
git clone --recurse-submodules git@github.com:s-celles/ManyUIDev.jl.git
cd ManyUIDev.jl
```

If you cloned without `--recurse-submodules`:

```sh
git submodule update --init --recursive
```

A plain `submodule update` leaves each submodule on a detached HEAD at the
pinned commit. To work on branches instead:

```sh
git submodule foreach 'git checkout main && git pull --ff-only'
```

## Everyday workflow

Make `git pull` and `git checkout` in the superproject carry the submodules
along:

```sh
git config submodule.recurse true
```

Advance every submodule to the tip of its tracked branch and record the new
pins:

```sh
git submodule update --remote --merge
git commit -am "chore: bump submodule pins"
```

Push a change in one package, then update the pin here:

```sh
cd ManyUI
git add -A && git commit -m "feat: ..." && git push
cd ..
git add ManyUI && git commit -m "chore: bump ManyUI pin"
```

Check whether any submodule has drifted from its recorded pin:

```sh
git submodule status
git submodule foreach 'git status --short'
```

## Notes

- The superproject records a **commit**, not a branch. Pushing inside a
  submodule is not enough — you must also commit the updated pin here, or
  collaborators will keep checking out the old revision.
- Push submodule commits *before* pushing the superproject, otherwise the pin
  points at a revision nobody else can fetch.
