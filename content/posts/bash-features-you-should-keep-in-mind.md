+++
title = "Bash stuff to keep in mind"
date = 2019-08-05T16:41:58+02:00
draft = true
+++

When working with VMs or containers nowadays scripting or command line handling in general becomes relevant again (even for the ordinary developer).<br/>
Hence I'd like to collect a few (sometimes even unrelated) shell/scripting snippets.

### Keyboard shortcuts

- `Ctrl-l` clear screen (preserves current line - unlike the `clear` command)
- `Ctrl-u` wipe current command line
- `Ctrl-a` go to beginning of the current line
- `Ctrl-e` go to end of the current line
- `Ctrl-c` interrupt running process
- `Ctrl-d` Exit

### Variable handling

- `${parameter:-default}`: substitutes `default` if `parameter` is unset or null (_very useful in `Dockerfile`s_)

    {{< highlight bash >}}
PROXY=${HTTP_PROXY:-proxy.me:8080} {{< /highlight >}}

- `${parameter:offset[:length]}`: substrings `parameter` - accepts negative values as well

    {{< highlight bash >}}
FOO=asdfghjkl
echo ${FOO:5}      # hjkl
echo ${FOO:2:2}    # df
echo ${FOO:2:-2}   # dfghj {{< /highlight >}}

- `${parameter#[#]pattern}`: delete (`#`/`##` shortest/longest matching) `pattern` at the beginning of `parameter` 
         
    {{< highlight bash >}}
FOO='The quick brown fox jumps over the lazy dog'
echo ${FOO#The quick}  # brown fox jumps over the lazy dog
echo ${FOO##*[The]}    # lazy dog {{< /highlight >}}

- `${parameter%[%]pattern}`: same as with `#` but applies to the end of `parameter`

- `${parameter/pattern/subst}`: applies the `pattern` and replaces the match with `subst`

    {{< highlight bash >}}
FOO='The quick brown fox jumps over the lazy dog'
echo ${FOO/fox/whale}    # The quick brown whale jumps over the lazy dog {{< /highlight >}}

If you want to go beyond the tip of the iceberg ... [see here][parameter-expansion].

### Use aliases 

`alias` is a _built-in command_ (not a script or program) which exists in almost every shell (in `bash`, in `sh` and `csh` and also in `zsh`) and which allows you to define shortcuts.

{{< highlight bash >}}
alias c='clear'
alias ..='cd ..'
alias ...='cd ../..'
alias ll='ls -lahtr'
alias dcl='docker-compose logs -f --tail 100'
{{< /highlight >}}

As built-in command you can type/define 'em right at your prompt - but then they're lost once you logout of our shell session. Put 'em in your `.bashrc`/`.zshrc` or even better in a dedicated `.aliases` (which gets sourced by the former rc-files) 'cause your list of alias-definitions can grow quickly.

### Use functions (_where aliases don't do it_)

`alias`es cannot take parameters - and generally are not made for complex things. For those use-cases you should define functions instead (again, put 'em into your `.bashrc`-file to preserve 'em).

{{< highlight bash >}}
# makes sure dir-structure exists before touching 
# file in that structure
# usage: touchin foo/bar/buzz.md
function touchin() { 
    for c in $@; do 
        mkdir -p $(dirname $c); 
        touch $c;
    done 
}

# encode given input as base64 
# usage: enc secret
function enc() { echo -n "$1" | base64 }
{{< /highlight >}}


[parameter-expansion]:https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html
