# Laundry Project

Umbrella repository for the laundry pickup & delivery platform. It holds no
application code itself — each app lives in its own repository and is pinned
here as a [git submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules).

| Path             | Repository                                                              | What it is                                                                 |
| ---------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `laundry_app/`   | [helnabawy/laundry_app](https://github.com/helnabawy/laundry_app)       | Flutter mobile app for **Customers** and **Drivers** (role-based routing). |
| `laundry_admin/` | [helnabawy/laundry_admin](https://github.com/helnabawy/laundry_admin)   | Web **Admin Portal** for staff and facility operators.                     |

See [PRODUCT.md](PRODUCT.md) for users, platforms, and product context.

## Getting started

Clone with submodules in one step:

```bash
git clone --recurse-submodules https://github.com/helnabawy/laundry_project.git
```

Already cloned without them? Fetch them now:

```bash
git submodule update --init --recursive
```

## How submodules work here

This repo does not track the apps' files. It records **one exact commit** of each
app. After cloning, each submodule is checked out at that commit in a detached
HEAD state, so switch to a branch before making changes:

```bash
cd laundry_app
git switch main
```

Check which commit each submodule is pinned to:

```bash
git submodule status
```

## Day-to-day workflow

### Working inside an app

Each submodule is a full git repository. Commit and push from inside it as usual:

```bash
cd laundry_admin
git switch main
# ...make changes...
git commit -am "feat: something"
git push
```

### Recording the new commit in this repo

Pushing inside a submodule does **not** update this repo. The project keeps
pointing at the old commit until you record the new one:

```bash
cd ..                        # back to laundry_project
git add laundry_admin
git commit -m "chore: bump laundry_admin"
git push
```

Push the submodule **before** pushing this repo. Otherwise this repo will point
at a commit that doesn't exist on GitHub, and other people's clones will fail.

### Pulling the latest everything

```bash
git pull
git submodule update --init --recursive       # check out the commits this repo pins
# or, to move every submodule to the tip of its remote branch instead:
git submodule update --remote --merge
```

### Adding another submodule

Always use `git submodule add` rather than `git clone`. A plain clone never
records the commit, so the folder won't appear on GitHub:

```bash
git submodule add https://github.com/helnabawy/<repo>.git <path>
git commit -m "chore: add <path> submodule"
git push
```
