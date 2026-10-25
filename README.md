# Using Emacs Ediff as Git Merge/Difftool

## Table of contents

1. [Overview](#overview)
2. [Objective](#objective)
3. [Implementation](#implementation)
   1. [Ediff Functions and Arguments](#ediff-functions-and-arguments)
   2. [Ediff Script](#ediff-script)
      1. [Script Modifications](#script-modifications)
   3. [Git Configuration](#git-configuration)
   4. [Caveats](#caveats)
4. [Footnotes](#footnotes)

## Overview

*Ediff* is «a comprehensive visual interface to Unix diff and patch utilities»<sup>[1](#myfootnote1)</sup> that comes with [Emacs](https://www.gnu.org/software/emacs/emacs.html). This document describes how to use Ediff with [Git](https://git-scm.com/) to resolve merge conflicts and inspect differences between file versions.

The solution uses a wrapper shell script called by `git mergetool` and `git difftool`. Git passes the appropriate arguments to the script through the relevant Git configuration.

The impatient may download the [Ediff wrapper script](https://github.com/paulotome/emacstool/raw/master/ediff.sh), adapt it as described in [Script Modifications](#script-modifications), and add the appropriate [Git configuration values](#git-configuration).

## Objective

The objective is to integrate Ediff smoothly into the Git workflow. The following requirements should be met:

1. Launch Ediff whenever `git mergetool` is invoked.
2. Launch Ediff whenever `git difftool` is invoked.
3. Prefer `emacsclient` over `emacs`.
4. Create a new Emacs frame when running under X.
5. When merging, include the ancestor (`BASE`) if available.
6. When merging, check the resulting file for conflict markers.
7. Return a non-zero exit status if conflict markers remain or the conflict-marker check fails.

## Implementation

As described in the [Git mergetool](https://git-scm.com/docs/git-mergetool) and [Git difftool](https://git-scm.com/docs/git-difftool) documentation, it is possible to define custom commands for merge and diff tools.

Although it would be feasible to define an `emacsclient` command directly in the Git configuration, I prefer to use a [shell script](#ediff-script). This makes it easier to select the appropriate Ediff function, pass the required arguments, check the merged file for conflict markers, and return an appropriate exit status.

The script is called with the arguments supplied by `git mergetool` or `git difftool`, as defined in the [Git configuration](#git-configuration).

### Ediff Functions and Arguments

There are three main cases that the script must handle, each requiring a specific Ediff function and its corresponding arguments.

1. **Git diff**

   A simple diff, triggered by `git difftool`, executes `ediff` with the pre-image and post-image files as arguments. This opens an Ediff session for comparing the two files.

2. **Git merge without ancestor**

   A merge without an ancestor can occur, for example, when two branches create the same path or file independently, without a common base version.

   In this case, `ediff-merge-files` is called with four arguments: the local file, the remote file, `nil` for the optional startup hooks, and the name of the file to associate with the merge buffer.

3. **Git merge with ancestor**

   Often, the conflicting versions of a file share an earlier version, known as the *ancestor* or *base*. When an ancestor is available, `ediff-merge-files-with-ancestor` is called with five arguments: the local file, the remote file, the ancestor file, `nil` for the optional startup hooks, and the name of the file to associate with the merge buffer.

   The final argument is the `MERGE-BUFFER-FILE` argument in the Ediff function interface. It identifies the file associated with the merge buffer; it should not be described simply as a guarantee that the merged result will be written to that file.

For the exact function signatures and argument descriptions, see the [Ediff manual](https://www.gnu.org/software/emacs/manual/html_node/ediff/Major-Entry-Points.html).

### Ediff Script

The following is the [Ediff wrapper script](https://github.com/paulotome/emacstool/blob/master/ediff.sh). The script should be executable. The [Git configuration](#git-configuration) below assumes that it is named `ediff.sh`.

```bash
#!/bin/bash

# test args
if [ "$#" -lt 2 ]; then
    echo 1>&2 "Usage: ${0} LOCAL REMOTE [MERGED BASE]"
    echo 1>&2 "       (LOCAL, REMOTE, MERGED, BASE can be provided by \`git mergetool'.)"
    exit 1
fi

# tools
_EMACSCLIENT=/usr/bin/emacsclient
_BASENAME=/usr/bin/basename
_CP=/bin/cp
_MKTEMP=/bin/mktemp
_GREP=/bin/grep

# args
_LOCAL=${1}
_REMOTE=${2}
if [ "${3}" ] ; then
    _MERGED=${3}
else
    _MERGED=${_REMOTE}
fi
if [ "${4}" ] && [ -r "${4}" ] ; then
    _BASE=${4}
    _EDIFF=ediff-merge-files-with-ancestor
    _EVAL="${_EDIFF} \"${_LOCAL}\" \"${_REMOTE}\" \"${_BASE}\" nil \"${_MERGED}\""
elif [ "${_REMOTE}" = "${_MERGED}" ] ; then
    _EDIFF=ediff
    _EVAL="${_EDIFF} \"${_LOCAL}\" \"${_REMOTE}\""
else
    _EDIFF=ediff-merge-files
    _EVAL="${_EDIFF} \"${_LOCAL}\" \"${_REMOTE}\" nil \"${_MERGED}\""
fi

# console vs. X
if [ "${TERM}" = "linux" ]; then
    unset DISPLAY
    _EMACSCLIENTOPTS="-t"
else
    _EMACSCLIENTOPTS="-c"
fi

# run emacsclient
${_EMACSCLIENT} ${_EMACSCLIENTOPTS} -a "" -e "(${_EVAL})" 2>&1

# check modified file
if ${_GREP} -Eq '^(<<<<<<<|=======|>>>>>>>|####### Ancestor)' "${_MERGED}"; then
    _MERGEDSAVE=$(${_MKTEMP} --tmpdir "$(${_BASENAME} "${_MERGED}")".XXXXXXXXXX)
    ${_CP} "${_MERGED}" "${_MERGEDSAVE}"
    echo 1>&2 "Oops! Conflict markers detected in ${_MERGED}."
    echo 1>&2 "Saved your changes to ${_MERGEDSAVE}"
    echo 1>&2 "Exiting with code 1."
    exit 1
else
    _GREP_STATUS=$?
    if [ "${_GREP_STATUS}" -gt 1 ]; then
        echo 1>&2 "Error checking conflict markers in ${_MERGED}."
        exit "${_GREP_STATUS}"
    fi
fi

exit 0
```

### Script Modifications

To use the script successfully, you may need to adapt it to your environment.

1. **Tool paths:** Check whether the required tools are available and adjust their paths if necessary: `_EMACSCLIENT`, `_BASENAME`, `_CP`, `_MKTEMP`, and `_GREP`.

2. **Console versus graphical display:** Adapt the display check to your needs. The script checks whether `TERM` is set to `linux` to select terminal mode; otherwise, it requests a new Emacs frame.

3. **Conflict-marker detection:** The script uses `grep -E` to match the configured conflict-marker patterns. The `-q` option suppresses normal output and allows `grep` to stop searching as soon as a match is found.

4. **Conflict markers found:** If the check finds a conflict marker, the script saves a copy of the edited file and exits with status `1`.

5. **No conflict markers found:** If the check completes successfully without finding a marker, the script exits with status `0`.

6. **Check fails:** If `grep` encounters an error, for example because the merged file cannot be read, it returns a status greater than `1`. The script propagates that status rather than treating the failed check as a successful merge.

### Git Configuration

To add Ediff as the default diff and merge tool, add the following to one of your Git configuration files, preferably `~/.gitconfig`:

```gitconfig
[diff]
    tool = ediff
    guitool = ediff

[difftool "ediff"]
    cmd = /PATH/TO/ediff.sh $LOCAL $REMOTE $MERGED

[merge]
    tool = ediff

[mergetool "ediff"]
    cmd = /PATH/TO/ediff.sh $LOCAL $REMOTE $MERGED $BASE
    trustExitCode = true
```

Replace `/PATH/TO/` with the actual path to the script. Leave the arguments (`$LOCAL`, `$REMOTE`, `$MERGED`, and `$BASE`) as they are. Git supplies these variables to the configured command, as appropriate.

Setting `trustExitCode = true` tells Git to use the merge tool's exit status to determine whether the merge succeeded. This makes the script's exit status relevant to Git's handling of the merge result.

### Caveats

1. **The script must wait for Emacs to finish.**

   To check for conflict markers reliably, the script needs to wait for `emacsclient` to return. It therefore does not use the `--no-wait` option.

   If you decide to use `--no-wait`, you should remove the conflict-marker check and set `mergetool.ediff.trustExitCode` to `false`. Otherwise, the script could check the file before Emacs has finished editing it.

2. **Unresolved conflict markers.**

   If conflict markers are found after `emacsclient` returns, the script exits with status `1`. With `mergetool.ediff.trustExitCode` set to `true`, Git uses the tool's exit status to determine whether the merge was successful. A non-zero exit status indicates that the merge was not successful, and Git may leave the file marked as unresolved.

   To prevent the edited version from being lost, the script saves a copy of the file before exiting. You can retrieve your edits from this backup if necessary.

3. **Conflict-marker checks are not a complete merge validation.**

   The script checks for lines beginning with the configured marker patterns. It does not determine whether the resulting merge is semantically correct. Matching text can also trigger the check even when it is not an actual unresolved conflict.

   In addition, the script's successful exit status indicates that no configured conflict markers were found and the check did not report an error. It should not be interpreted as proof that every aspect of the merge succeeded.

## Footnotes

<a name="myfootnote1">1</a>: [GNU Emacs Ediff Manual](https://www.gnu.org/software/emacs/manual/html_node/ediff/)

Adapted from: `http://ulf.zeitform.de/en/documents/git-ediff.html`

Author: Ulf Stegemann (ulf@zeitform.de)
