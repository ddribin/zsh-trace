# zsh-trace

Analyze Zsh startup performance from xtrace logs. Generates summaries and
[flamegraphs](https://www.brendangregg.com/flamegraphs.html).

## Requirements

- Python 3.6+
- [flamegraph.pl](https://github.com/brendangregg/FlameGraph) (for SVG output)

## Generating a trace

Add this near the top of your `~/.zshenv` (since that [runs before][zsh-startup-order] `~/.zshrc`):

[zsh-startup-order]: https://shreevatsa.wordpress.com/2008/03/30/zshbash-startup-files-loading-order-bashrc-zshrc-etc/

```zsh
# Profiling via:  
# https://kev.inburke.com/kevin/profiling-zsh-startup-time/
: "${PROFILE_STARTUP:=false}"
: "${PROFILE_ALL:=false}"
# Run this to get a profile trace and exit: time zsh -i -c true
# Or: time PROFILE_STARTUP=true /bin/zsh -i --login -c true
if [[ "$PROFILE_STARTUP" == true || "$PROFILE_ALL" == true ]]; then
    # http://zsh.sourceforge.net/Doc/Release/Prompt-Expansion.html
    PS4='%D{%M%S%6.} %N:%i> '
    #zmodload zsh/datetime
    #PS4='+$EPOCHREALTIME %N:%i> '
    local _profile_log=/tmp/zsh_profile.$$
    exec 3>&2 2>$_profile_log
    setopt xtrace prompt_subst
    print "Profiling to $_profile_log" >&3
fi
# "unsetopt xtrace" is at the end of ~/.zshrc
```

And at the end of your `~/.zshrc`:

```zsh
if [[ "$PROFILE_STARTUP" == true ]]; then
    unsetopt xtrace
    exec 2>&3 3>&-
fi
```

Start a login shell and analyze the trace file:

```
> PROFILE_STARTUP=true /bin/zsh -i --login -c true
Profiling to /tmp/zsh_profile.21786
```

**Important:** Use `%6.` (microseconds) rather than `%.` (milliseconds) in PS4
for useful time resolution.

## Usage

```
zsh-trace summary /tmp/zsh_profile.<pid>
zsh-trace flamegraph /tmp/zsh_profile.<pid> | flamegraph.pl > startup.svg
```

### summary

Show top time consumers:

```
$ zsh-trace summary /tmp/zsh_profile.652

Total: 73,255 μs (73.3 ms)

Top 15:

    14.4 ms  (19.7%)  ~/.zshenv > /etc/zprofile > (eval) > ~/.zprofile > /etc/zshrc > ~/.zshrc
    10.0 ms  (13.6%)  ~/.zshenv > /etc/zprofile > (eval) > ~/.zprofile > /etc/zshrc
     8.2 ms  (11.2%)  ~/.zshenv > ... > my_fast_compinit > compinit
     ...
```

Use `-n` to control how many entries to show:

```
zsh-trace summary -n 25 /tmp/zsh_profile.652
```

### flamegraph

Output folded stacks for `flamegraph.pl`:

```
zsh-trace flamegraph /tmp/zsh_profile.652 | flamegraph.pl --title "Zsh Startup" --countname "μs" > startup.svg
open -a Safari startup.svg
```

Add `--summary` to also print the summary to stderr while generating the
flamegraph:

```
zsh-trace flamegraph --summary /tmp/zsh_profile.652 | flamegraph.pl > startup.svg
```

## Tips

The biggest startup time wins come from avoiding fork/exec. Every external
command (`date`, `stat`, `scutil`, `brew shellenv`, `zoxide init zsh`, etc.)
costs 1–5 ms to fork a process. Strategies:

- **Use Zsh builtins** instead of external commands (e.g., `zmodload zsh/stat`
  and `zstat` instead of `stat`, `$EPOCHSECONDS` instead of `date +%s`)
- **Cache command output** to a file for commands like `zoxide init zsh` or
  `scutil --get ComputerName` that rarely change
- **`compinit -C`** skips security checks and is faster — use it when the
  dump file is fresh
