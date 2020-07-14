## Description

`git_xy` helps to synchronize sub directories between git repositories
semi-automatically, and may generate pull requests on `github` if
changes are detected on the destination repository.

`git_xy` reads a list of source/destination specifications from a
configuration file, and for each of them, `git_xy` fetches changes
from the source repository and synchronizes them to destination path
(thanks to `rsync`). It finally generates commit and creates new PR
(pull request) if necessary.
See more details in [How it works](#how-it-works).

## TOC

* [Description](#description)
* [Usage](#usage)
  * [Installation](#installation)
  * [Configuration](#configuration)
  * [Invocation](#invocation)
  * [Sample Prs on Github](#sample-prs-on-github)
  * [Environment variables](#environment-variables)
  * [Hooks](#hooks)
  * [How it works](#how-it-works)
* [TODO](#todo)
* [Why](#why)
* [Author. License](#author-license)

## Usage

### Installation

`git_xy` is a Bash4 script. It requires some additional tools on system:

* GNU tools: `awk`, `rsync`, `bash`, `git`, `grep`, `sed`
* Github command tool for PR creation: https://github.com/cli/cli/releases

The main program `git_xy` can be installed anywhere on your search path

```
$ sudo wget -O /usr/local/bin/git_xy \
    https://github.com/icy/git_xy/raw/ng/git_xy.sh

$ sudo chmod 755 /usr/local/bin/git_xy
```

### Configuration

Configuration consists of source/destination specification in the following
format:

```
src_repo src_branch src_path   dst_repo dst_branch dest_path [pr_base]
```
which reads in the order

* `src_repo`, `src_branch`, `src_path`: The source repository, branch and path
* `dst_repo`, `dst_branch`, `dst_path`: The destination repository, branch and path.
  By default, `rsync` without `--delete` option is used, which allows the
  downstream (destination) path to contain their own additional files.
  If you want destination to be exact the upstream, use the colon (`:`)
  as prefix of the destination path, i.e., `:dst_path`.

The last part `pr_base` is optional and is used to specify where
you want the PR arrives. By default, it's the upstream repository.

See examples in [git_xy.config-sample.txt](git_xy.config-sample.txt).

```
git@github.com:icy/pacapt ng lib/ git@github.com:icy/pacapt master lib/
```

### Invocation

Now execute the script

```
GIT_XY_CONFIG="git_xy.config-sample.txt" ./git_xy.sh
```

the script will fetch changes in `lib` directory from branch `ng`
in the `pacapt` repository,
and update the same repository on another branch `master`.
If changes are detected, a new branch will be created and/or
some pull request will be generated.

### Sample Prs on Github

These PRs are generated by using [sample configuration file](git_xy.config-sample.txt).

Generated by the latest version of the script:

* https://github.com/icyfork/pacapt/pull/18 (File synchronization)
* https://github.com/icyfork/pacapt/pull/17
* https://github.com/icyfork/pacapt/pull/14
* https://github.com/icyfork/pacapt/pull/15 (same configuration,
  but when `GIT_XY_REVERSE=yes`)

Generated by some older versions of the script:

* https://github.com/icyfork/pacapt/pull/10
* https://github.com/icyfork/pacapt/pull/1
* https://github.com/icy/pacapt/pull/140
* https://github.com/icy/pacapt/pull/139

### Environment variables

* `GIT_XY_CONFIG`: Path to the configuration file. When it is empty and/or
  not specified, the value `git_xy.config` is used.
* `GIT_XY_PUSH_OPTIONS`: Options used by `git push` command when
  new commit is generated by the script. Default to empty string.
* `GIT_XY_SET_OPTIONS`: Options used by `set` command. See `man set`
  for details. For example, if you want to turn on debug mode,
  use `GIT_XY_SET_OPTIONS=-x`. Please use this variable with care;
  and please note that the option `-u` and `+e` will be always enforced.
* `GIT_XY_HOOKS`: Specify a list of post-commit hooks. By default,
  it is `gh`. See [Hooks](#hooks) for details
* `GIT_XY_REVERSE`: When the value is `yes`, the synchronization direction
  is reversed (dst becomes src, src become dst). This is useful when
  you want to fetch something from the downstream.
  The `pr_base` option is not changed when the option is `yes`.
* `D_GIT_SYNC`: Where the script fetches remote repositories.
  It's a hard-coded string `$HOME/.local/share/git_xy/`
  here `$HOME` is your home folder.

### Hooks

A hook is a predefined method which would be executed when new commit
is generated. A hook is expected to return zero code.

* `gh`: Generate a Github pull request when new commit is created.

### How it works

Nothing magic, it's a wrapper of `git clone, rsync and git commit`:)
Let's say we have configuration file

```
src_repo src_branch src_path dst_repo dst_branch dst_path
```

the script will do as below

* Create a clone of the `src_repo` in `~/.local/share/git_xy/src_repo`
  (The actual folder name is a bit different to avoid some special characters
  in the user input.)
* Check out the existing branch `src_branch`
* Create a clone of the `dest_repo` in `~/.local/share/git_xy/dst_repo`
* Check out the existing branch `dst_branch`
* Create new branch from `dst_branch` (if neccessary).
  This branch is specially used for PR creation.
* Use `rsync` to synchronize the contents of the `src_path` and `dst_path`.
  On the local machine where the script runs, it's a variant of the command
  `rsync -ra --delete SRC/ DST/` here
  `SRC` is `~/.local/share/git_xy/src_repo/src_path/` and
  `DST` is `~/.local/share/git_xy/dst_repo/dst_path/`
* Generate new commit
* If specified, execute the hook to generate new pull request on Github

Well, it's so easy right? It's an automation support of your handy commands.

## TODO

- [ ] Add tests and automation support for the project
- [ ] Provide a link to the original source
- [ ] More `POSIX` ;)
- [ ] Sometimes we only need to create a PR without generate any commit
- [ ] Re-use existing branch to generate new PR

Done

- [x] `hook/gh`: Return zero if a PR already exists
- [x] Support file synchronization...
- [x] Add option to reverse the synchronization (dst becomes src and vice versa)
- [x] Better error reporting
- [x] Handle `--delete` option
- [x] Create a hook script to create pull requests
- [x] Gather multiple sub-folder in a single PR
- [x] Re-use existing `git_xy` branch
- [x] Better hook to handle where PRs will be created
- [x] Add some information from the last commit of the source repository
- [x] Make sure the top/root directory is not used (we allow that)
- [x] Allow a repository to update itself

## Why

There are many tools trying to solve the code-sharing problem:

### Native support

* `git submodule`:
  Create pointers to some commit hashes on the upstream repositories, and
  check out the upstream repository as sub-directory of the current
  repository upon request.
  It's you and your team who watch changes on upstream and fetch them
  manually. More submodules to watch, more issues to handle.
* `git subtree`:
  Quite similar to `git submodule`, but it doesn't provide fragile pointer.
  Instead, it fetches all upstream commits and creates some merge points
  in the current repository.
  This way it's more stable than `git submodule`,
  when you always have a copy of the upstream code in your repository.
  You get what you don't really mean: duplication of commits,
  bigger size, confused/noisy commit messages,
  and you have to learn how to merge (really?)
  Good reading: https://www.atlassian.com/git/tutorials/git-subtree.

  Git-subtree original experimental project is found
  here https://github.com/apenwarr/git-subtree.

Looked like `git submodule` requires you to understand C programming language,
while `git subtree` is kind of Python which hides pointers from your laptop:D

### Meta-repository

* https://github.com/ingydotnet/git-subrepo:
  Another `git-slave`-liked project, which helps to manage multiple
  small repositories. It uses `git work-tree`, and it helps to generate
  pull/push/merge command in sub repositories by using the same command.
  Forget your `git command`, as you have to learn to use this
  new wrapper for all sub commands
  (commit, pull, merge...) Written completely in Bash.
* https://github.com/twosigma/git-meta:
  Another `git-slave` which uses `git-submodule` to create
  the meta repository. Another NodeJs tool
* https://github.com/mateodelnorte/meta:
  Similar to `git-slave`, which creates a meta repository
  that includes multiple small repositories. You adapt both mono/micro
  repository idea. Written in NodeJs...
* https://sourceforge.net/projects/gitslave/:
  The project is still on `sourceforge`, looked like it's not maintained.
  The idea is to have meta project which handles multiple repository.
  The basic tutorial is here:
  http://gitslave.sourceforge.net/tutorial-basic.html.

### Very constraint tools

* https://github.com/teambit/bit (npm only):
  Use this if your current Linux kernel is written in NodeJs.
* https://github.com/lerna/lerna (javascript only):
  Use this if your current Linux kernel is written in Javascript.
* https://gerrit.googlesource.com/git-repo/ (Android only?):
  Use this if your server is running on Android.
* https://github.com/microsoft/VFSForGit (sic, Windows only):
  Use this if you are running Linux inside a VM inside a Windows host.

### Back-and-forth tools

* https://github.com/splitsh/lite:
  Split a repository to read-only standalone repositories
* https://github.com/unravelin/tomono:
  You hate micro-`***` enough and you just want a big repository
  that includes all your small repositories. This tool helps you.

Well, there are too many tools...
What I really need is a simple way to pull changes from some repository
to another repository, generates some pull request for reviewing,
and the downstream maintainer will decide what they would do next.

Morever, this process should be done automatically when the upstream
repository is updated. Human intervention is not the right way when
there are just 100 or 500 repositories because of the raise of the
micro-repository `design` (if any) :D

## Author. License

The script is written by Ky-Anh Huynh.
The work is released under a MIT license.
