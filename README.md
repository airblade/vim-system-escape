# Escaping Vim system() Calls

You can run non-interactive shell commands in Vim via the [`system(command, input)`][system] function.  The commands must of course be escaped correctly for the shell, and different shells have different escaping rules.

In constructing and running a command, two levels of escaping are done:

1. You have to use [`shellescape()`][shellescape] to escape special characters in the arguments, e.g. file paths, of the command you pass to `system()`.
2. Vim escapes the entire command appropriately for your shell.

This works pretty well in recent versions of Vim on non-Windows systems.  Windows is problematic because its shells' escaping rules are odd and they change from one shell to the next.

The way Vim escapes commands for your shell has evolved over time.  Consider these patches for example:

* [Vim 7.3.443][443]: MS-Windows: 'shcf' and 'sxq' defaults are not very good
* [Vim 7.3.445][445]: (after 7.3.443) can't properly escape commands for cmd.exe
* [Vim 7.3.446][446]: (after 7.3.445) external command with special char doesnt work
* [Vim 7.3.447][447]: (after 7.3.446) External commands with "start" do not work
* [Vim 7.3.448][448]: (after 7.3.447) Win32: Still a problem with "!start /b"
* [Vim 7.3.450][450]: (after 7.3.448) Win32: Still a problem with "!start /b"

The problem for plugin authors is using `system()` in a way that works whatever version of Vim their users have.

The only way to do it is to handle as much of the escaping in your own VimL code, effectively backporting Vim's escaping logic.  This is non-trivial and, as far as I can tell, there is no consensus on the best approach.

I would love to see a definitive shell-escaping plugin which everybody else can depend on once and for all.  It's silly for every plugin author to reinvent the wheel, especially when it's so trick to get right.  I hope that this repo can become that plugin.

Below we look at how 3 popular plugins handle escaping.  But first here's an overview of how Vim constructs the commands it passes to the shell.


## How Vim Constructs Shell Commands

The command executed in constructed using several options (source: [system() documentation][system]):

```
'shell' 'shellcmdflag' 'shellxquote' command 'shellredir' tmp 'shellxquote'
```

– where `command` is the string you passed to `system()` and `tmp` is an automatically generated file name.

Here are the relevant options:

(Shell constraints are shown in square brackets.)

Name | Description | Default Unix | Default Windows
-----|-------------|--------------|----------------
[shell][]        | Name of the shell to use | `$SHELL` or `sh` | `command.com` or `cmd.exe`
[shellcmdflag][] | Flag passed to the shell | `''` | [contains `sh`]: `-c`; otherwise `/c`
[shellpipe][]    | String to use to put output of `:make` in error file | [`csh`, `tcsh`]: `|& tee`; [`sh`, `ksh`, `zsh`, `bash`]: `2>&1| tee`; otherwise `| tee` | `>`
[shellquote][]   | Quoting character(s) surrounding command passed to shell excluding redirection | `''` | [contains `sh`]: `\"`; otherwise ``
[shellredir][]   | String to use to put output of a filter command in a temporary file | [`csh`, `tcsh`, `zsh`]: `>&`; [`sh`, `ksh`, `bash`]: `>%s 2>&1`; otherwise `>` | [`cmd`]: `>%s 2>&1`; same as unix






  [system]: http://vimdoc.sourceforge.net/htmldoc/eval.html#system()
  [shellescape]: http://vimdoc.sourceforge.net/htmldoc/eval.html#shellescape()
  [443]: http://code.google.com/p/vim/source/detail?r=de050fcc24cfb56a7dc07dd283cc1132d774e7b7
  [445]: http://code.google.com/p/vim/source/detail?r=397e7e49bb0b831f7260d3ad70f6b07175c44a0c
  [446]: http://code.google.com/p/vim/source/detail?r=20ca2e05ae20ece942490182691ed45746f64cb6
  [447]: http://code.google.com/p/vim/source/detail?r=6a03b0ea2e12d748c1e4199e3f428ee080760939
  [448]: http://code.google.com/p/vim/source/detail?r=756d712b3118b896b57ddb4f4c071135bc031607
  [450]: http://code.google.com/p/vim/source/detail?r=3479ac596f6c4b38849d2e5235ad590378605eb8
  [shell]: http://vimdoc.sourceforge.net/htmldoc/options.html#'shell'
  [shellcmdflag]: http://vimdoc.sourceforge.net/htmldoc/options.html#'shellcmdflag'
  [shellpipe]: http://vimdoc.sourceforge.net/htmldoc/options.html#'shellpipe'
  [shellquote]: http://vimdoc.sourceforge.net/htmldoc/options.html#'shellquote'
