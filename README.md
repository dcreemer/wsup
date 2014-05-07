wsup
====

```wsup``` (pronounced "wassup") is a lightweight package management systems for your shell scripts,
editor configurations, and general compute environment setup. Using wsup you can go from bare
OS to a completely personalized and consistent working environment with two or three commands.

## Targets ##

```wsup``` works on the concept of independent, layered, named "targets". Each target is simply a
fragment of your home directory, and is contained in a git repository. For example, you may
have a ```~/bin``` directory which is composed of various utility commands. Some of these
commands may be applicable to Mac OS X, some to Linux. Some may be for personal related tasks,
some for various work projects. In wsup, you would partition your work and personal
configurations into separate "slices" of your ```~/bin``` directory, and store each in a
separate git repo.

Example: your work computer contains the following ```~/bin```:

    ~/bin
       |
       +- work-command
       |
       +- personal-command

You've decided that the "personal-command" should be installed everywhere you have an account
(for example your laptop and VPS shell account). However the "work-command" should only be
installed on work-related accounts. To accomplish this with wsup, create two git repositories:

**"example.com/joe/work" git repository:**

    bin/work-command
    README.md


**"github.com/joe/personal" git repository:**

    bin/personal-command
    README.md

On your laptop, you can then install both configurations with wsup:

    $ wsup add personal
    $ wsup add https://example.com/joe/work

If your laptop username is "joe" and the wsup configuration is left as the defaults, wsup will
automatically look for a corresponding user on github.com for targets not specified in URL
format. Both the default repository site and user can be changed in your local
configuration. See "Configuration Files" below.

## Commands ##

Using "wsup add" as show above with clone the named repositories into the ```~/.wsup```
directory, and then create the ```~/bin``` directory (if needed) and make symbolic links for
the files in each target's bin directory. The resulting directory tree will look like:

    ~/bin
       |
       +- work-command -> ~/.wsup/work/bin/work-command
       |
       +- personal-command -> ~/.wsup/personal/bin/personal-command

```wsup link <target>``` and ```wsup unlink <target>``` will add (or remove) the links to the
target's local repository, and so enable or disable the target. For example:

    $ wsup unlink work
    [            work] unlink /Users/joe/bin/work-command
    $ wsup list
    TARGET           S REPOSITORY
    ---------------- - --------------------------------------------------------------------
    work             ! https://example.com/joe/work.git
    personal         * git@github.com:joe/personal.git

```wsup link``` and ```wsup unlink``` can be called with multiple targets, or with no targets,
in which case the command applies to all know targets.

**Important**
```wsup link``` and ```wsup unlink``` are idempotent -- repeatedly running the same link or
unlink command is just fine.

```wsup list``` as show above lists the currently known targets, their corresponding git
repositories, and the installation status (the "S" column). Here ```*``` indications that the
target is correctly installed. ```!``` indicates that the target is not, or only partially
installed, and ``` ``` (blank) means that the target is know, but does not have a local
repository (see ```add_target``` in the Configiguration Files section below).

Finally ```wsup help``` lists a summary of the available commands.

## Linking Rules ##

```wsup``` manages slices of your home directory by creating symbolic links. It uses a simple
rule for creating symlinks, with some basic exceptions. For any non-directory file found in a
target's top level directory, ```wsup``` makes a direct symlink to that file. If ```wsup```
finds a directory in your target, it first creates that directory in your home directory (if
needed), and then creates symlinks to everything in that directory.

Expanding on the example above, if the work and personal repositories look like:

You've decided that the "personal-command" should be installed everywhere you have an account
(for example your laptop and VPS shell account). However the "work-command" should only be
installed on work-related accounts. To accomplish this with wsup, create two git repositories:

**"example.com/joe/work" git repository:**

    bin/work-command
    .gitignore
    README.md


**"github.com/joe/personal" git repository:**

    bin/personal-command
    .bashrc
    README.md

The resulting home directory structure would be:

    +- .gitignore -> ~/.wsup/work/.gitignore
    |
    +- .bashrc -> ~/.wsup/personal/.bashrc
    |
    ~/bin
       |
       +- work-command -> ~/.wsup/work/bin/work-command
       |
       +- personal-command -> ~/.wsup/personal/bin/personal-command

## Configuration Files ##

Before executing a command, wsup reads the ```.wsup/config``` file, if available, in addition
to any target-specific configurations files (see below). This file is a bash file that is
executed in the context of the ```wsup``` script, and is used to customize the ```wsup```
environment.

Variable      | Default                       | Description
------------- | -----------                   | -----------
VERBOSE       | 1                             | set to enable debug output
WD            | $HOME/.wsup                   | wsup home directory
REPO_USER     | $USER                         | the name of the account on the git repo site 
REPO_PREFIX   | "git@github.com:${REPO_USER}" | prefix to use when cloning named targets
BASE_TARGET   | dotfiles                      | name of the target / repository to automatically add to the targets list as a "base" target
IGNORE_RE     | <see below>                   | regular expression of repo files to ignore

In the default configuration, ```wsup``` ignores (does not link into $HOME) any repository
files that match the ```IGNORE_RE``` regular expression. This is used to excluded files and
directories such as ````.git````, ```README.md```, etc. The default IGNORE_RE is:

```
(^\.git$)|(^\.gitignore$)|(^\.wsup$)|(^[Rr][Ee][Aa][Dd][Mm][Ee].*)|(^LICENSE.*)
```

### Target Configuration Files ###

In addition to the ```.wsup/config``` file, each target may also include a
```~/.wsup/<target>/.wsup/config``` file that adds to and overrides the global file. This is
also just a bash file that is executed just prior to executing ```wsup link``` (or ```wsup
add```, which calls link). In addition to setting configuration, you can also use the
```wsup``` built-in ```add_package``` command to install packages. For example:

    install_package "Linux" aspell
    install_package "FreeBSD" aspell
    install_package "Darwin" aspell

will install the "aspell" package on Linux, FreeBSD, and Darwin. ```install_package``` uses the
OS underlying package manager to install the named packages (apt-get, pkg, and brew on Linux,
FreeBSD, and Mac OS X respectively).

Finally, each target may have a ```~/.wsup/<target>/.wsup/postinstall``` script. This must also
be a bash script which is executed after a successful link of the named target. One example use
may be installing additional software:

```bash
OS=`uname -s`
if [[ "$OS" == "Darwin" ]]; then
    if [[ ! -d /Applications/Dropbox.app ]]; then
        echo "[INSTALL] Dropbox"
        curl -fsSL "https://www.dropbox.com/download?plat=mac" > ~/Downloads/dropbox.dmg
        hdiutil attach ~/Downloads/dropbox.dmg
        SAVEIFS=$IFS
        IFS=$(echo -en "\n\b")
        open "/Volumes/Dropbox Installer/Dropbox.app"
        IFS=$SAVEIFS
        rm -f ~/Downloads/dropbox.dmg
    fi
fi
```

### OS Specific Directories ###

If a ```~/.wsup/<target>/.wsup/<OS>``` directory exists, where ```<OS>``` is one of

- Darwin
- Linux
- FreeBSD

then files in that directory will be linked by ```wsup``` on the appropriate operating system
(as determined by ```uname -s```).

## Bootstrap ##

Assumes curl bash, git, and sudo.
