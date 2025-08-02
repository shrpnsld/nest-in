# nest-in

![tests](https://github.com/shrpnsld/nest-in/actions/workflows/tests.yml/badge.svg)

Setup your personal environment with a single command. Useful when *a shell-script becomes too complicated or not enough for this task*.

### Features

* Configuration file has well-defined structure and simple syntax.
* Allows setup process to be granular, selective and recoverable.
* Has no external dependencies, so can be used right away in a completely fresh environment.
* Written with Bash 3.2, so it works even on macOS with its stock Bash installation.

## Usage

### One-liners

```bash
curl -fsSL <twigs-file> | /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/shrpnsld/nest-in/master/nest-in)"
```

```bash
wget -qO- <twigt-file> | /bin/bash -c "$(wget -qO- https://raw.githubusercontent.com/shrpnsld/nest-in/master/nest-in)"
```

### Command-line interface

```bash
$ nest-in <twigs-file> [-sdnh] [-- targets ...]
```

### Options

* `<twigs-file>` – file with nesting configuration.
* `-d`, `--dependencies` – show dependencies for targets.
* `-n`, `--dry-run` – do not run scripts, only show them.
* `-i`, `--system-info` – print detected system information.
* `-- targets ...` – targets to work with.
* `-h` – show help message.

If no `<twigs-file>` was passed, then configuration is read from *stdin*.

## Guide

The entire nesting process is divided into steps. Each nesting step is defined as a target with zero or more requirements, dependencies, and artifacts, with or without a script. Requirements are preconditions for the environment that must be met for the target to be nested. Dependencies are other targets that are nested before the target itself. Artifacts are files, directories, or programs produced by the target. If these artifacts already exist, the target is considered to be already nested. And scripts are just Bash-scripts that nest target.

Shell scripts for each target run independently, but they share variable and function definitions, so a variable or a function that was defined in one target will be accessible in a another target.

If some script fails, nest-in will suggest to open shell to fix things, then, uppon exit, prompt to nest failed target again or continue nesting next target.

### Targets

Target declaration should start at the beginning of the line and *should not* be preceeded by any whitespace. Each script line *should* be preceeded by any whitespace. Target names can include letters, numbers, dashes, and underscore characters, but they must always start with a letter.

```bash
cmake
    brew install cmake
```

Dependencies should be listed after a `/`.

```bash
nvim / dotfiles stow
    brew install neovim
    cd ~/.dotfiles
    stow nvim
```

### Artifacts

Artifacts are also listed after the same `/` as dependencies. Each artifact should be enclosed in a pair of single quotes and can specify either a path to a file/directory or a program name:

```bash
dotfiles / git '~/.dotfiles/'
    git clone https://github.com/me/dotfiles ~/.dotfiles
```

### Requirements

Requirements are listed after the same `/` as dependencies and artifacts and each requirement should be enclosed in square brackets. Requirements can have specifiers, where each specifier must be preceded by a `:` character. Requirement and specifier names can include letters, numbers, dashes, and underscore characters, but requirement names must always start with a letter.

The same target can have multiple variants with different sets of requirements, each variant can have its own dependencies, artifacts, and script. The example below declares first variant to be chosen if the operating system is macOS, and the second variant will be chosen if the environment has the available program `apt-get`.

```bash
installer / [macos]
    INSTALL='brew install'

installer / [avail:apt-get]
    INSTALL='sudo apt install'
```

If multiple target variants have their requirements met, the first one declared will be chosen. So, with the example below, if the environment has both `curl` and `wget` installed, the first variant will be chosen; if only `wget` is present, then the second one will be selected.

```bash
downloader / [avail:curl]
    DOWNLOAD='curl -fsSL'

downloader / [avail:wget]
    DOWNLOAD='wget -qO-'
```

Targets might not always have a variant with met requirements for every environment. In such a case, if nothing depends on this target, it will be skipped without an error. In the example below, when nesting in Ubuntu, `brew` doesn't have any variant for Ubuntu, but since `pkgmgr / [ubuntu]` does not depend on it, there is no error.

```bash
brew / [macos]
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/.../install.sh)"

pkgmgr / [macos] brew
    INSTALL='brew install'

pkgmgr / [ubuntu]
    INSTALL='sudo apt install'
```

### Special targets

By default, all targets will be chosen for nesting. But some targets may not have been designed to be included in the default nesting. In the example below, the `cleanup` target removes configurations and uninstalls programs that were installed previously. To exclude it from the default nesting, it can be marked with `!` after its name, and it will be nested only if specified through a command line or if it is a dependency for another target.

```bash
cleanup!
    cd ~/.dotfiles
    stow --delete tmux ranger nvim git
    rm -rf ~/.dotfiles
    brew uninstall tmux ranger nvim git stow
    brew autoremove
```

When some work needs to be done before or after nesting, `.first` and `.last` targets can be used. These targets can have requirements but cannot have dependencies or artifacts. In the example below, `.first` defines a global variable with the dotfiles path.

```bash
.first
    DOTFILES_PATH="~/.dotfiles"
```

When predefined requirements are not enough, `.reqs` target can be used to define Bash functions as custom requirements. This target can't have requirements, dependencies or artifacts.

```bash
spyware / [name:jasonbourne]
    sudo apt install spyware

.reqs
    name() { [[ $USER == $1 ]]; }
```

### More on requirements

Requirements always constist of a name and may have a set of specifiers, where each specifier is preceeded by a `:`. Requirement names can have letters, numbers, dash and underscore characters, but always should start with a letter. For all predefined requirements, any number of last specifiers or version components can be dropped, to make requirement broader.

`[avail:<command>]` – specified command is available.

`[os:<name>:<version>]` – OS name or Linux distribution matches specified name and version.

`[macos:<version>]` – OS is macOS and matches specified version.

`[cygwin]` – OS is cygwin

`[linux:<family>:<dist>:<version>]` – OS is Linux and matches specified family, distribution and version.

`[arch|debian|redhat|suse:<distribution>:<version>]` – predefined shortcuts of `[linux:::]` requirement for some Linux families.

`[manjaro|ubuntu|elementary|kali|tails|fedora|opensuse:<version>]` – predefined shortcuts of `[linux:::]` requirement for some Linux distributions.

### More on artifacts

Artifact types are determined by the following examples:

* `~/artifact/is/a/file` – path *with no* trailing `/`
* `~/artifact/is/a/direcotry/` – path *with* trailing `/`
* `program` – name that can be found in one the paths specified by `$PATH` environment variable

### Additional notes

When recovering from a script error inside a shell, any variable and function definitions made inside that shell will not be visible in subsequent scripts.

Comments are not supported and any syntax-check pass for a shell-style comment is just a happy accident :)

## Examples

### Typical setup

```bash
# "nvim" depends on "pkgmgr" and "dotfiles" being nested first
nvim / pkgmgr dotfiles
    $INSTALL neovim
    cd "$DOTFILES_DIR"
    stow nvim

# "dotfiles" needs no dependencies but it produces two artifacts: directory "~/.dotfiles/"
# and program "stow". if both already exist, then target is considered nested.
dotfiles / '~/.dotfiles/' 'stow'
    $INSTALL stow
    git clone https://github.com/me/dotfiles "$DOTFILES_DIR"

# this variant of "pkgmgr" will be used only on macOS. its script uses Homebrew,
# so target declaration lists dependency on "brew" target.
pkgmgr / [macos] brew
    $INSTALL='brew install'
    $UNINSTALL='brew uninstall'

# this variant of "pkgmgr" will be used only if "apt-get" is available.
pkgmgr / [avail:apt-get]
    $INSTALL='sudo apt-get -y install'
    $UNINSTALL='sudo apt-get -y uninstall'

# target for installing Homebrew on macOS. it lists "brew" as an artifact,
# so if Homebrew is already installed, there's no need for installation
brew / [macos] 'brew'
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# dedicated target for cleanup, intended to be used only explicitly, like so:
# $ nest-in cleanup
clenaup!
    cd "$DOTFILES_DIR"
    stow --delete nvim
    rm -rf "$DOTFILES_DIR"
    $UNINSTALL nvim stow

# "setup" target that defines a variable that would be used throughout the nesting process
.first
    DOTFILES_DIR="$HOME/.dotfiles"
```

### Home/Work setup

```bash
# "home" and "work" targets specify what should be nested through their dependencies.
# they are made dedicated, because there's no sense for both of them to be included
# in the default nesting at the same time.
home! / git-personal pet-projects
work! / git-work caffeinate games

# your personal ".gitconfig" for your pet projects
git-personal! / stow
    cd "$DOTFILES_DIR"
    stow git-personal

# setting up ".gitconfig" at your work
git-work!
    read -p 'git name: ' name
    read -p 'git email: ' email
    git config --global user.name="$name"
    git config --global user.email="$email"

# will keep your status "online", while you're avoiding your responsibilities
caffeinate
    $INSTALL caffeinate

# keeps your sanity and helps to get through your work day
games
    $INSTALL nethack myman ninvaders

pet-projects / git-personal
    mkdir "$PROJECTS_DIR"
    cd "$PROJECTS_DIR"
    git clone https://github.com/me/pet
    git clone https://github.com/me/another-pet
    git clone https://github.com/me/and-another-one

# in case you're fired, hide the evidence
cleanup!
    $UNINSTALL games caffeinate

# and you've seen all this before
pkgmgr / [macos] brew
    $INSTALL='brew install'
    $UNINSTALL='brew uninstall'

pkgmgr / [avail:apt-get]
    $INSTALL='sudo apt-get -y install'
    $UNINSTALL='sudo apt-get -y uninstall'

brew / [macos] 'brew'
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

.first
    DOTFILES_DIR="$HOME/.dotfiles"
    PROJECTS_DIR="$HOME/Projects"
```

### My own twigs.txt

[https://gist.github.com/shrpnsld/933a367bcba6a3b42b82fd901b7736eb](https://gist.github.com/shrpnsld/933a367bcba6a3b42b82fd901b7736eb)
