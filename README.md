# nest-in

Setup new environment, copy configs, install programs, etc., with a single command.

### Features

* Nesting process is defined using simple syntax.
* The whole setup is divided into nesting steps, called targets.
* Nesting for each target is specified as a shell-script.
* Targets can have dependencies on other targets.
* Targets can have different definitions depending on environment conditions.
* Written with Bash 3.2, so it works even on macOS with its stock Bash installation.

## One-liner

```bash
curl -fsSL PUT-URL-TO-YOUR-TWIGS-FILE-HERE.txt | /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/shrpnsld/nest-in/master/nest-in)" -- '--'
```

## Usage

```bash
$ nest-in [-sdnh] [-- [<twigs.txt>]]
```

### Options

* `<target>...` – specify targets to nest.
* `-d` – show all target dependencies.
* `-s` – show target script.
* `-n` – dry run.
* `-- <twigs.txt>` – specify file with nesting configuration (should be the last argument).
* `--` – read nesting configuration from *stdin* (should be the last argument).
* `-h` – show help message.

If no `--` was passed, then input file is searched under `~/.config/nest-in/twigs.txt`.

## Syntax

Target declaration should start at the beginning of the line and should not be preceeded by any whitespace. Target declaration always consists of a target name, which can be followed by `!`, and then can be followed by `/` with a list of dependencies and requirements which are separated by a whitespace.

Target names can have letters, numbers, dash and underscore characters, but always should start with a letter. `!` next to a target name makes that target dedicated.

Each requirement is enclosed in its own square brackets. It always constist of a name and may have list of values, each value is preceeded by `:`. Requirement names can have letters, numbers, dash and underscore characters, but always should start with a letter.

Targets can list shell-scripts that describe nesting process. Each command should be preceeded by some whitespace to distinguish it from target declaration.

```bash
target! / [linux] [has:apt-get] another and-another
    command1
    command2
    ...
    commandN
```

There are also setup and teardown targets named `+` and `-` respectively. These targets can have requirements, but they can't have dependencies.

Comments are not supported and any syntax-check pass for a shell-style comment is accidential :)

## Nesting Process

During the default nesting process (no targets are passed through command line), all targets are being processed. Each target has its dependencies procsssed recursively first, then the target itself is being processed.

Dedicated targets are not included in the default nesting, unless they are listed as a dependency for another target.

If some targets were specified through a command line, then only those targets and their dependencies are nested. Dedicated targets will also be nested in this way.

Before the nesting process, *nest-in* would check:

* if all dependencies are declared and their requirements are fulfilled
* if there's no duplicates, or more than one version of a target has its requirements fulfilled
* for circular dependencies between targets

Target `+` will be nested before any other target, and target `-` will be nested after all the other targets.

All shell-scripts for all targets share the same variable scope. This allows to define common variables, like paths or install commands, that would be used in other shell-scripts.


## Examples

### Typical setup

```bash
# 'nvim' depends on 'pckgmgr' and 'dotfiles' being nested first
nvim / pckgmgr dotfiles
	$install neovim
	cd "$DOTFILES_DIR"
	stow nvim

# 'dotfiles' needs no dependencies
dotfiles
	$install stow
	git clone https://github.com/me/dotfiles "$DOTFILES_DIR"

# this variant of 'pckgmgr' will be used only on macOS.
# it installs homebrew and sets a variable with a call to the install command.
pckgmgr / [macos]
	/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
	$install='brew install'
	$uninstall='brew uninstall'

# this variant of 'pckgmgr' will be used only if 'apt-get' can be called.
# it sets a variable with a call to the install command.
pckgmgr / [has:apt-get]
	$install='sudo apt-get -y install'
	$uninstall='sudo apt-get -y uninstall'

# dedicated target for cleanup, intended to be used only explicitly, like so:
# $ nest-in cleanup
clenaup!
	$uninstall nvim stow
	rm -rf "$DOTFILES_DIR"

# 'setup' target that defines a variable
# that would be used throughout the nesting process
+
	DOTFILES_DIR="$HOME/.dotfiles"

```

### Home/Work setup

```bash
# 'home' and 'work' targets specify what should be nested through their dependencies.
# they are made dedicated, because there's no sense for both of them to be included
# in the default nesting at the same time.
home! / git-personal pet-projects
work! / git-work caffeinate games

# your personal '.gitconfig' for your pet projects
git-personal! / stow
	cd "$DOTFILES_DIR"
	stow git-personal

# setting up '.gitconfig' at your work
git-work!
	read -p 'git name: ' name
	read -p 'git email: ' email
	git config --global user.name="$name"
	git config --global user.email="$email"

# will keep your status 'online', while you're avoiding your responsibilities
caffeinate
	$install caffeinate

# keeps your sanity and helps to get through your work day
games
	$install nethack myman ninvaders

pet-projects / git-personal
	mkdir "$PROJECTS_DIR"
	cd "$PROJECTS_DIR"
	git clone https://github.com/me/pet
	git clone https://github.com/me/another-pet
	git clone https://github.com/me/and-another-one

# in case you're fired, hide the evidence
cleanup!
	$uninstall games caffeinate

+
	DOTFILES_DIR="$HOME/.dotfiles"
	PROJECTS_DIR="$HOME/Projects"
```
