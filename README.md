# Escaping Vim system() Calls

You can run non-interactive shell commands in Vim via the [`system(command, input)`][system] function.  The commands must of course be escaped correctly for the shell, and different shells have different escaping rules.

In constructing and running a command, two levels of escaping are done:

1. You have to use [`shellescape()`][shellescape] to escape special characters in the arguments, e.g. file paths, of the command you pass to `system()`.
2. Vim escapes the entire command appropriately for your shell.

This works pretty well in recent versions of Vim on non-Windows systems.  Windows is problematic because its shells' escaping rules are odd.

The way Vim escapes commands for your shell has evolved over time.  Consider these patches for example:

Patch | Description
------|------------
[7.3.443][443] | MS-Windows: 'shcf' and 'shellxquote' defaults are not very good. Make a better guess when 'shell' is set to "cmd.exe".
[7.3.445][445] | Can't properly escape commands for cmd.exe. Default 'shellxquote' to '('.  Append ')' to make '(command)'. No need to use "/s" for 'shellcmdflag'.
[7.3.446][446] | Win32: External commands with special characters don't work. Add the 'shellxescape' option.
[7.3.447][447] | Win32: External commands with "start" do not work. Unescape part of the command.
[7.3.448][448] | Win32: Still a problem with "!start /b". Escape only '&#124;'.
[7.3.450][450] | Win32: Still a problem with "!start /b". Fix pointer use.

You can read some of the [discussion around the patches][vim-dev-discussion-0] if you're feeling brave, and a [follow-up][vim-dev-discussion-1].

The problem for plugin authors is using `system()` in a way that works whatever version of Vim their users have.

The only way to do it is to handle as much of the escaping in your own VimL code, effectively backporting Vim's escaping logic.  This is non-trivial and, as far as I can tell, there is no consensus on the best approach.

I would love to see a definitive shell-escaping plugin which everybody else can depend on once and for all.  It's silly for every plugin author to reinvent the wheel, especially when it's so tricky to get right.

Below we look at how several popular plugins handle escaping.  But first here's an overview of how Vim constructs the commands it passes to the shell.


## How Vim Constructs Shell Commands

The command executed in constructed using several options (source: [system() documentation][system]):

```
'shell' 'shellcmdflag' 'shellxquote' command 'shellredir' tmp 'shellxquote'
```

– where `command` is the string you passed to `system()` and `tmp` is an automatically generated file name.  On Unix braces are put around `command` to allow for concatenated commands.

For example, when I run `:echo system('ls')` using Vim 7.4.052 with Bash on OS X:

```
/bin/bash -c "(ls) >some_tmp_file 2>&1"
```

You can see everything after `shellcmdflag` by setting Vim's verbosity: `set verbose=4`.

Here are the relevant options, as of 7818ca6de3d0 (11 December 2013):

(Shell constraints are shown in square brackets.)

Option | Description | Default Unix | Default Windows
-------|-------------|--------------|----------------
[shell][]        | Name of the shell to use | `$SHELL` or `sh` | `command.com` or `cmd.exe`
[shellcmdflag][] | Flag passed to the shell | `-c` | [contains `sh`]: `-c`; otherwise `/c`
[shellpipe][]    | String to use to put output of `:make` in error file | [`csh`, `tcsh`]: <code>&#124;& tee</code>; [`sh`, `ksh`, `mksh`, `pdksh`, `zsh`, `bash`]: <code>2>&1&#124; tee</code>; otherwise <code>&#124; tee</code> | `>`
[shellquote][]   | Quoting character(s) surrounding command passed to shell excluding redirection | | [contains `sh`]: `"`
[shellredir][]   | String to use to put output of a filter command in a temporary file | [`csh`, `tcsh`, `zsh`]: `>&`; [`sh`, `ksh`, `bash`]: `>%s 2>&1`; otherwise `>` | [`cmd`]: `>%s 2>&1`; same as unix
[shellslash][]   | Only when a backslash can be used as a path separator: when set, a forward slash is used when expanding file names. Useful when a Unix-like shell is used on Windows. | off | off
[shelltemp][]    | When set, use temp files for shell command; otherwise use a pipe | on | on
[shelltype][]    | (Amiga only) | off | off
[shellxescape][] | When `shellxquote` is set to `(`, the characters listed in this option will be escaped with `^` | | <code>"&&#124;<>()@^</code>
[shellxquote][]  | Quoting character(s) surrounding command passed to shell including redirection | when using `system()`: `"` | [`cmd.exe`]: `(`; [contains `sh`]: `"`

You also need to know the [rules for including whitespace in a string option][backslash] value.

And here's the [`shellescape(str)`][shellescape] function, as of 350272cbf1fd (23 January 2014):

> Escape `str` for use as a shell command argument.  On Windows, when `shellslash` is not set, it encloses `str` in double quotes and doubles all double quotes within `str`.
> For other systems, it encloses `str` in single quotes and replaces all `'` with `'\''`.

Overall there are quite a few moving parts to account for.

## How Several Popular Plugins Handle Escaping

I think it instructive to examine how several popular plugins handle escaping.  Popular plugins are, by definition, widely used and therefore have had to learn to cope with the wide variety of Vim versions and shells in the wild.  As you will see, they take different approaches ;)

### vim-dispatch

[vim-dispatch][] provides a [function][vim-dispatch-escape] to escape a command invoked on Windows with `silent execute '!start cmd.exe ...'`:

```viml
function! s:escape(str)
  if &shellxquote ==# '"'
    return '"' . substitute(a:str, '"', '""', 'g') . '"'
  else
    let esc = exists('+shellxescape') ? &shellxescape : '"&|<>()@^'
    return &shellquote .
          \ substitute(a:str, '['.esc.']', '^&', 'g') .
          \ get({'(': ')', '"(': ')"'}, &shellquote, &shellquote)
  endif
endfunction
```

The `if` block doubles up any `"` characters when `shellxquote` is `"` – which sounds like `shellescape()` on Windows when `shellslash` is off.  The `else` block implements `shellxescape`'s escaping rules.

### vim-fugitive

[vim-fugitive][] implements its own [`shellescape(arg)`][vim-fugitive-shellescape]:

```viml
function! s:shellesc(arg) abort
  if a:arg =~ '^[A-Za-z0-9_/.-]\+$'
    return a:arg
  elseif &shell =~# 'cmd'
    return '"'.s:gsub(s:gsub(a:arg, '"', '""'), '\%', '"%"').'"'
  else
    return shellescape(a:arg)
  endif
endfunction
```

And its own [`s:fnameescape(file)`][vim-fugitive-fnameescape]:

```viml
function! s:fnameescape(file) abort
  if exists('*fnameescape')
    return fnameescape(a:file)
  else
    return escape(a:file," \t\n*?[{`$\\%#'\"|!<")
  endif
endfunction
```

Finally, here is an [excerpt][vim-fugitive-excerpt] from the `s:ReplaceCmd()` function:

```viml
if &shell =~# 'cmd'
  let cmd_escape_char = &shellxquote == '(' ?  '^' : '^^^'
  call system('cmd /c "' . prefix . s:gsub(a:cmd,'[<>]', cmd_escape_char.'&') . ' > ' . tmp . '"')
else
  call system(' (' . prefix . a:cmd . ' > ' . tmp . ') ')
endif
```

This looks a little like an implementation of `shellxescape` combined with `shell` and `shellcmdflag`.


### vundle

[vundle][] provides a Windows-aware `cd` [function][vundle-cd]:

```viml
func! g:shellesc(cmd) abort
  if ((has('win32') || has('win64')) && empty(matchstr(&shell, 'sh')))
    if &shellxquote != '('      " workaround for patch #445
      return '"'.a:cmd.'"'      " enclose in quotes so && joined cmds work
    endif
  endif
  return a:cmd
endf

func! g:shellesc_cd(cmd) abort
  if ((has('win32') || has('win64')) && empty(matchstr(&shell, 'sh')))
    let cmd = substitute(a:cmd, '^cd ','cd /d ','')  " add /d switch to change drives
    let cmd = g:shellesc(cmd)
    return cmd
  else
    return a:cmd
  endif
endf
```

The `g:shellesc()` function looks like a partial implementation of `shellescape()` when `shellslash` is off.

The `g:shellesc_cd()` function adds the `/d` switch to support Windows' drives and then calls `g:shellesc()`.

It is invoked in the `s:sync()` function as follows (omitting some irrelevancies):

```viml
if ...
  let cmd = 'cd '.shellescape(a:bundle.path()).' && git pull && git submodule update --init --recursive'
  let cmd = g:shellesc_cd(cmd)
  let get_current_sha = 'cd '.shellescape(a:bundle.path()).' && git rev-parse HEAD'
  let get_current_sha = g:shellesc_cd(get_current_sha)
  let initial_sha = s:system(get_current_sha)[0:15]
else
  let cmd = 'git clone --recursive '.shellescape(a:bundle.uri).' '.shellescape(a:bundle.path())
endif
let out = s:system(cmd)
```

### vim-signify

[vim-signify][] provides a [replacement][vim-signify-escape] for `shellescape()` for use with file paths:

```viml
function! sy#util#escape(path) abort
  if exists('+shellslash')
    let old_ssl = &shellslash
    if fnamemodify(&shell, ':t') == 'cmd.exe'
      set noshellslash
    else
      set shellslash
    endif
  endif

  let path = shellescape(a:path)

  if exists('old_ssl')
    let &shellslash = old_ssl
  endif

  return path
endfunction
```

An example invocation from `autoload/sy/repo.vim` is:

```viml
let root = finddir('.git', fnamemodify(b:sy.path, ':h') .';')
let root = fnamemodify(root, ':h')
let output = system('cd '. sy#util#escape(root) .' && git diff --numstat')
```

The justifications for setting `noshellslash` before invoking `shellescape()` were:

> But many people use [`shellslash`] even with cmd.exe because it lets you use forward slashes within Vim easier, and most (but not all) Windows programs in cmd.exe will work with both forward and backwards slashes.
>
> It would probably be best, if you also test the value of 'shell' to see whether it actually is still set to start with "command" or "cmd"; shellslash can remain set if a Unix-like shell is actually being used. The reason for resetting 'shellslash' if cmd.exe or similar is in use, is that shellescape() assumes a Unix-like shell if shellslash is set.

Source: [issue #15][vim-signify-15]

And:

> So, basically, when one starts Vim from Command Prompt (`cmd.exe`), then `&shell` points to `cmd.exe` indeed, and paths for `system()` call have to be escaped with `noshellslash`. However, when one starts Vim from Sh, Bash, Ksh, Csh, etc., then `&shell` points to one of them, and, in this case, paths for `system()` call have to be escaped with `shellslash`, otherwise everything breaks because of backward slashes `\` in paths.

Source: [issue #99][vim-signify-99]


## What Now?

I would like a VimL replacement for `shellescape(str)` and possibly one for escaping the entire command passed to `system()`.

These should work on Unix and Windows regardless of the user's Vim version.

Reading the [discussions][vim-dev-discussion-0] on the vim-dev mailing list, it feels like a perfect solution is unlikely.  And a lot depends on the complexity of the commands invoked with `system()`.

However I hope we can get 90%-95% of the way there.


## What Can You Do?

Please send me your suggestions!







  [vim-dev-discussion-0]: https://groups.google.com/d/topic/vim_dev/vVOynF5fKlA/discussion
  [vim-dev-discussion-1]: https://groups.google.com/d/topic/vim_dev/NkiQOCNErt8/discussion
  [system]: http://code.google.com/p/vim/source/browse/runtime/doc/eval.txt?r=350272cbf1fd8371aeb93a12fbf0d11708fb2776#5907
  [shellescape]: http://code.google.com/p/vim/source/browse/runtime/doc/eval.txt?r=350272cbf1fd8371aeb93a12fbf0d11708fb2776#5407
  [443]: http://code.google.com/p/vim/source/detail?r=de050fcc24cfb56a7dc07dd283cc1132d774e7b7
  [445]: http://code.google.com/p/vim/source/detail?r=397e7e49bb0b831f7260d3ad70f6b07175c44a0c
  [446]: http://code.google.com/p/vim/source/detail?r=20ca2e05ae20ece942490182691ed45746f64cb6
  [447]: http://code.google.com/p/vim/source/detail?r=6a03b0ea2e12d748c1e4199e3f428ee080760939
  [448]: http://code.google.com/p/vim/source/detail?r=756d712b3118b896b57ddb4f4c071135bc031607
  [450]: http://code.google.com/p/vim/source/detail?r=3479ac596f6c4b38849d2e5235ad590378605eb8

  [shell]:        http://code.google.com/p/vim/source/browse/runtime/doc/options.txt?r=7818ca6de3d07439571bc1fe788a3efff6979531#5994
  [shellcmdflag]: http://code.google.com/p/vim/source/browse/runtime/doc/options.txt?r=7818ca6de3d07439571bc1fe788a3efff6979531#6026
  [shellpipe]:    http://code.google.com/p/vim/source/browse/runtime/doc/options.txt?r=7818ca6de3d07439571bc1fe788a3efff6979531#6044
  [shellquote]:   http://code.google.com/p/vim/source/browse/runtime/doc/options.txt?r=7818ca6de3d07439571bc1fe788a3efff6979531#6079
  [shellredir]:   http://code.google.com/p/vim/source/browse/runtime/doc/options.txt?r=7818ca6de3d07439571bc1fe788a3efff6979531#6096
  [shellslash]:   http://code.google.com/p/vim/source/browse/runtime/doc/options.txt?r=7818ca6de3d07439571bc1fe788a3efff6979531#6122
  [shelltemp]:    http://code.google.com/p/vim/source/browse/runtime/doc/options.txt?r=7818ca6de3d07439571bc1fe788a3efff6979531#6137
  [shelltype]:    http://code.google.com/p/vim/source/browse/runtime/doc/options.txt?r=7818ca6de3d07439571bc1fe788a3efff6979531#6154
  [shellxescape]: http://code.google.com/p/vim/source/browse/runtime/doc/options.txt?r=7818ca6de3d07439571bc1fe788a3efff6979531#6168
  [shellxquote]:  http://code.google.com/p/vim/source/browse/runtime/doc/options.txt?r=7818ca6de3d07439571bc1fe788a3efff6979531#6177
  [backslash]:    http://code.google.com/p/vim/source/browse/runtime/doc/options.txt?r=7818ca6de3d07439571bc1fe788a3efff6979531#173

  [vim-dispatch]: https://github.com/tpope/vim-dispatch
  [vim-dispatch-escape]: https://github.com/tpope/vim-dispatch/blob/ffbd5eb50c9daf67657b87fd767d1801ac9a15a7/autoload/dispatch/windows.vim#L8-L17
  [vim-fugitive]: https://github.com/tpope/vim-fugitive
  [vim-fugitive-shellescape]: https://github.com/tpope/vim-fugitive/blob/8f0b8edfbd246c0026b7a2388e1d883d579ac7f6/plugin/fugitive.vim#L29-L37
  [vim-fugitive-fnameescape]: https://github.com/tpope/vim-fugitive/blob/8f0b8edfbd246c0026b7a2388e1d883d579ac7f6/plugin/fugitive.vim#L39-L45
  [vim-fugitive-excerpt]: https://github.com/tpope/vim-fugitive/blob/8f0b8edfbd246c0026b7a2388e1d883d579ac7f6/plugin/fugitive.vim#L2038-L2043
  [vundle]: https://github.com/gmarik/vundle
  [vundle-cd]: https://github.com/gmarik/vundle/blob/f31aa52552ceb40240e56e475e6df89cc756507e/autoload/vundle/installer.vim#L253-L270
  [vim-signify]: https://github.com/mhinz/vim-signify
  [vim-signify-escape]: https://github.com/mhinz/vim-signify/blob/1789155bf3d4c1380cdbcfd43f6789ae50f3169d/autoload/sy/util.vim#L6-L23
  [vim-signify-15]: https://github.com/mhinz/vim-signify/issues/15#issuecomment-15529339
  [vim-signify-99]: https://github.com/mhinz/vim-signify/issues/99#issue-23377283

