# nest-in

Setup your personal environment with a single command. Useful when *a shell-script becomes too complicated or not enough for this task*.

### Features

* Configuration file has well-defined structure and simple syntax.
* Allows setup process to be granural, selective and adjustible.
* Has no external dependencies, so can be used right away in a completely fresh environment.
* Written with Bash 3.2, so it works even on macOS with its stock Bash installation.

## Usage

### One-liners

```bash
curl -fsSL PUT-URL-TO-YOUR-TWIGS-FILE-HERE.txt | /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/shrpnsld/nest-in/master/nest-in)" -- --
```

```bash
wget -qO- PUT-URL-TO-YOUR-TWIGS-FILE-HERE.txt | /bin/bash -c "$(wget -qO- https://raw.githubusercontent.com/shrpnsld/nest-in/master/nest-in)" -- --
```

### Command-line interface

```bash
$ nest-in [-sdnh] [-- [<twigs.txt>]]
```

### Options

* `<target>...` – specify targets to nest.
* `-d` – show all target dependencies.
* `-n` – do not run, only show assembled script.
* `-- <twigs.txt>` – specify file with nesting configuration (should be the last argument).
* `--` – read nesting configuration from *stdin* (should be the last argument).
* `-h` – show help message.

If no `--` was passed, then input file is searched under `~/.config/nest-in/twigs.txt`.

## Guide

The whole nesting process is divided into steps. Each nesting step is defined as a target with zero or more requirements, dependencies and artifacts; with or without a script. Requirements are preconditions for the enviromnent that should be fulfilled in order for the target to be nested. Dependencies are other targets that would be nested before the target itself. Artifacts are files/directories/programs that are produced by the target and if those already exist, target would be considered as nested. All shell-scripts for all the chosen targets would be combined in a single work-script and thus share the same scope.

### Targets

Target declaration should start at the beginning of the line and *should not* be preceeded by any whitespace. Each script line *should* be preceeded by any whitespace. Target names can have letters, numbers, dash and underscore characters, but always should start with a letter.

```bash
cmake
	brew install cmake
```

Dependencies should be listed after a `/`.

``` bash
nvim / dotfiles stow
	brew install neovim
	cd ~/.dotfiles
	stow nvim
```

### Artifacts

Artifacts are also listed after the same `/` that dependencies are. Each artifact should be enclosed in a pair of single quotes and can specify a path to a file/directory or a program name:

```bash
dotfiles / git '~/.dotfiles/'
	git clone https://github.com/me/dotfiles ~/.dotfiles
```

### Requirements

Requirements are listed after the same `/` that dependencies and artifacts are. Each requirement should be enclosed in square brackets. Same targets can have multiple variants with different set of requirements, and each variant can have its own dependencies, artifacts and script. The example bellow declares first variant to be chosen if operating system is macOS, and the second variant will be chosen if environment has available program `apt-get`.

```bash
installer / [macos]
	INSTALL='brew install'

installer / [avail:apt-get]
	INSTALL='sudo apt install'
```

If multiple target variants have their requirements fulfilled, then the one that is declared first will be chosen. So with the example bellow, if the environment has both `curl` and `wget` installed, then the first variant will be chosen; if there's only `wget`, then the second one.

```bash
downloader / [avail:curl]
	DOWNLOAD='curl -fsSL'
	
downloader / [avail:wget]
	DOWNLOAD='wget -qO-'
```

Targets might not always have a variant with fulfilled requirements for every environment. In such case, if no one depends on this target, it would be skipped without an error. In the examle below, when nesting in Ubuntu, `brew` doesn't have any variant for Ubuntu, but since `pkgmgr / [ubuntu]` does not depend on it, there would be no error.

```bash
brew / [macos]
		/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/.../install.sh)"

pkgmgr / [macos] brew
	INSTALL='brew install'

pkgmgr / [ubuntu]
	INSTALL='sudo apt install'
```

### Special targets

By default, all targets would be chosen for the nesting. But some targets may have not been designed to be included in the default nesting. In the example below `cleanup` target removes configurations and uninstalls programs that were installed previously. To exclude it from the default nesting it can be marked with `!` after its name, and it will be nested only if it was specified through a command line or if it is a dependency for another target.

```bash
cleanup!
	cd ~/.dotfiles
	stow --delete tmux ranger nvim git
	rm -rf ~/.dotfiles
	brew uninstall tmux ranger nvim git stow
	brew autoremove
```

When some work needs to be done before or after nesting, `.first` and `.last` targets can be used. These targets can have requirements, but they can't have dependencies or artifacts. In the example bellow, `.first` defines global variable with dotfiles path:

```bash
.first
	DOTFILES_PATH="~/.dotfiles"
```

When predefined requirements are not enough, `.reqs` target can be used to define Bash functions as custom requirements. This target can't have requirements, dependencies or artifacts.

```bash
spyware / [name:jasonbourne]
	sudo apt install spyware

.reqs
	name()	{ [[ $USER == $1 ]]; }
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

`[desktop:<name>]` – Linux desktop environment matches specified name.

`[gnome|kde|mate|xfce|cinnamon]` – predefined shortcuts of `[desktop:]` requirement for some Linux desktop environments.

### More on artifacts

Artifact types are determined by the following examples: 

* `~/artifact/is/a/file` – path *with no* trailing `/`
* `~/artifact/is/a/direcotry/` – path *with* trailing `/`
* `program` – no path, just a name

File-artifacts are checked with `[ -r <file> ]`; directory-artifacts with `[ -d <directory> ]`; program-artifacts with `command -v <program>`.

### Additional notes

Comments are not supported and any syntax-check pass for a shell-style comment is accidential :)

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
