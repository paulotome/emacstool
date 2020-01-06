# Using Emacs Ediff as Git Merge/Difftool

## Table of contents
1. [Overview](#overview)
2. [Objective](#objective)
3. [Implementation](#Implementation)
   1. [Ediff Functions and Arguments](#ediff-functions-and-arguments)
   2. [Ediff Script](#ediff-script)
      1. [Script Modifications](#script-modifications)
   3. [Git Configurations](#git-confifurations)
   4. [Caveats](#caveats)
[](#)

## Overview

*Ediff* is «a comprehensive visual interface to Unix diff and patch utilities»<sup>[1](#myfootnote1)</sup> that comes with your [Emacs](https://www.gnu.org/software/emacs/emacs.html). This document shows how to use Ediff with [Git](https://git-scm.com/) for resolving merge conflicts and inspecting differences between file versions.

The solution described here uses a wrapper-shell-script that is called by *git mergetool* and *git difftool*. Arguments for the script are provided by Git via the appropriate Git configuration.

The impatient may just download the [wrapper-script](https://github.com/paulotome/emacstool/raw/master/ediff.sh) for Ediff, possibly **[adapt](#script-modifications)** it (see also: Ediff Script) and add the appropriate **[Git configuration values](#git-configuration)**.


## Objective

The objective is to integrate Ediff smoothly into the Git workflow. The following should be achieved:

1. Launch Ediff whenever git mergetool is invoked.
2. Launch Ediff whenever git difftool is invoked.
3. Prefer *emacsclient* over *emacs*.
4. Create a new Emacs frame when on X.
5. When merging, include ancestor (BASE) if available.
6. When merging, check resulting merged file for conflict markers.
7. When merging, use exit code to decide if merge was successful.


## Implementation

As the [Git mergetool](https://git-scm.com/docs/git-mergetool) and *difftool* help describes, it is possible to define custom merge and diff tool commands. Although it would be feasible to define an *emacsclient* command directly in the Git configuration, I prefer to use a **[shell script](#ediff-script)**. This makes it easier to decide which Ediff function to execute and which exit code to return. The script is called with the appropriate arguments from *git mergetool* or *git difftool* as defined in the **[Git configuration](#git-configuration)**.


### Ediff Functions and Arguments

There are basically three different cases that the script must handle by executing a specific Ediff function with the proper arguments:

1. **Git diff**<br>A simple diff, triggered by *git difftool* should execute `ediff` with the diff pre- and post-image as arguments.

2. **Git merge without ancestor**<br>A merge without ancestors occurs for instance if you merge two branches and those two branches have created the same path/file independently, i.e. without a common base version. In this case `ediff-merge-files` should be executed with three arguments: the version of the file in the current branch, the version of the file in the branch to be merged from and the target file to write the merged version to.

3. **Git merge with ancestor**<br>Often the conflicting versions of a file have a common earlier version, their *ancestor* or *base*. When an ancestor is available `ediff-merge-files-with-ancestor` should be executed with four arguments: the version of the file in the current branch, the version of the file in the branch to be merged from, the target file to write the merged version to and the base version of the file.

### Ediff Script


Following below is the [wrapper-script](https://github.com/paulotome/emacstool/blob/master/ediff.sh) for Ediff which should be
executable. The <a href="#sec-3-3">Git configuration</a> below assumes that the script is named
<code>ediff.sh</code>.


<div class="org-src-container">

<pre class="src src-shell-script"><span class="linenr"> 1: </span><span style="color: #8b0000;">#</span><span style="color: #8b0000;">!/bin/</span><span style="color: #ee0000;">bash</span>
<span class="linenr"> 2: </span>
<span class="linenr"> 3: </span><span style="color: #8b0000;"># </span><span style="color: #8b0000;">test args</span>
<span class="linenr"> 4: </span><span style="color: #ee0000;">if</span> [ ! ${<span style="color: #a0522d;">#</span>} -ge 3 ]; <span style="color: #ee0000;">then</span>
<span class="linenr"> 5: </span>    <span style="color: #483d8b;">echo</span> 1&gt;&amp;2 <span style="color: #8b2252;">"Usage: ${0} LOCAL REMOTE MERGED BASE"</span>
<span class="linenr"> 6: </span>    <span style="color: #483d8b;">echo</span> 1&gt;&amp;2 <span style="color: #8b2252;">"       (LOCAL, REMOTE, MERGED, BASE can be provided by \`git mergetool'.)"</span>
<span class="linenr"> 7: </span>    <span style="color: #ee0000;">exit</span> 1
<span class="linenr"> 8: </span><span style="color: #ee0000;">fi</span>
<span class="linenr"> 9: </span>
<span class="linenr">10: </span><span style="color: #8b0000;"># </span><span style="color: #8b0000;">tools</span>
<span class="linenr">11: </span><span style="color: #a0522d;">_EMACSCLIENT</span>=/usr/bin/emacsclient
<span class="linenr">12: </span><span style="color: #a0522d;">_BASENAME</span>=/bin/basename
<span class="linenr">13: </span><span style="color: #a0522d;">_CP</span>=/bin/cp
<span class="linenr">14: </span><span style="color: #a0522d;">_EGREP</span>=/bin/egrep
<span class="linenr">15: </span><span style="color: #a0522d;">_MKTEMP</span>=/bin/mktemp
<span class="linenr">16: </span>
<span class="linenr">17: </span><span style="color: #8b0000;"># </span><span style="color: #8b0000;">args</span>
<span class="linenr">18: </span><span style="color: #a0522d;">_LOCAL</span>=${<span style="color: #a0522d;">1</span>}
<span class="linenr">19: </span><span style="color: #a0522d;">_REMOTE</span>=${<span style="color: #a0522d;">2</span>}
<span class="linenr">20: </span><span style="color: #a0522d;">_MERGED</span>=${<span style="color: #a0522d;">3</span>}
<span class="linenr">21: </span><span style="color: #ee0000;">if</span> [ ${<span style="color: #a0522d;">4</span>} -a -r ${<span style="color: #a0522d;">4</span>} ] ; <span style="color: #ee0000;">then</span>
<span class="linenr">22: </span>    <span style="color: #a0522d;">_BASE</span>=${<span style="color: #a0522d;">4</span>}
<span class="linenr">23: </span>    <span style="color: #a0522d;">_EDIFF</span>=ediff-merge-files-with-ancestor
<span class="linenr">24: </span>    <span style="color: #a0522d;">_EVAL</span>=<span style="color: #8b2252;">"${_EDIFF} \"${_LOCAL}\" \"${_REMOTE}\" \"${_BASE}\" nil \"${_MERGED}\""</span>
<span class="linenr">25: </span><span style="color: #ee0000;">elif</span> [ ${<span style="color: #a0522d;">_REMOTE</span>} = ${<span style="color: #a0522d;">_MERGED</span>} ] ; <span style="color: #ee0000;">then</span>
<span class="linenr">26: </span>    <span style="color: #a0522d;">_EDIFF</span>=ediff
<span class="linenr">27: </span>    <span style="color: #a0522d;">_EVAL</span>=<span style="color: #8b2252;">"${_EDIFF} \"${_LOCAL}\" \"${_REMOTE}\""</span>
<span class="linenr">28: </span><span style="color: #ee0000;">else</span>
<span class="linenr">29: </span>    <span style="color: #a0522d;">_EDIFF</span>=ediff-merge-files
<span class="linenr">30: </span>    <span style="color: #a0522d;">_EVAL</span>=<span style="color: #8b2252;">"${_EDIFF} \"${_LOCAL}\" \"${_REMOTE}\" nil \"${_MERGED}\""</span>
<span class="linenr">31: </span><span style="color: #ee0000;">fi</span>
<span class="linenr">32: </span>
<span class="linenr">33: </span><span style="color: #8b0000;"># </span><span style="color: #8b0000;">console vs. X</span>
<span class="linenr">34: </span><span style="color: #ee0000;">if</span> [ <span style="color: #8b2252;">"${TERM}"</span> = <span style="color: #8b2252;">"linux"</span> ]; <span style="color: #ee0000;">then</span>
<span class="linenr">35: </span>    <span style="color: #483d8b;">unset</span> DISPLAY
<span class="linenr">36: </span>    <span style="color: #a0522d;">_EMACSCLIENTOPTS</span>=<span style="color: #8b2252;">"-t"</span>
<span class="linenr">37: </span><span style="color: #ee0000;">else</span>
<span class="linenr">38: </span>    <span style="color: #a0522d;">_EMACSCLIENTOPTS</span>=<span style="color: #8b2252;">"-c"</span>
<span class="linenr">39: </span><span style="color: #ee0000;">fi</span>
<span class="linenr">40: </span>
<span class="linenr">41: </span><span style="color: #8b0000;"># </span><span style="color: #8b0000;">run emacsclient</span>
<span class="linenr">42: </span>${<span style="color: #a0522d;">_EMACSCLIENT</span>} ${<span style="color: #a0522d;">_EMACSCLIENTOPTS</span>} -a <span style="color: #8b2252;">""</span> -e <span style="color: #8b2252;">"(${_EVAL})"</span> 2&gt;&amp;1
<span class="linenr">43: </span>
<span class="linenr">44: </span><span style="color: #8b0000;"># </span><span style="color: #8b0000;">check modified file</span>
<span class="linenr">45: </span><span style="color: #ee0000;">if</span> [ ! $(egrep -c <span style="color: #8b2252;">'^(&lt;&lt;&lt;&lt;&lt;&lt;&lt;|=======|&gt;&gt;&gt;&gt;&gt;&gt;&gt;|####### Ancestor)'</span> ${<span style="color: #a0522d;">_MERGED</span>}) = 0 ]; <span style="color: #ee0000;">then</span>
<span class="linenr">46: </span>    <span style="color: #a0522d;">_MERGEDSAVE</span>=$(${<span style="color: #a0522d;">_MKTEMP</span>} --tmpdir <span style="color: #ff00ff;">`${_BASENAME} ${_MERGED}`</span>.XXXXXXXXXX)
<span class="linenr">47: </span>    ${<span style="color: #a0522d;">_CP</span>} ${<span style="color: #a0522d;">_MERGED</span>} ${<span style="color: #a0522d;">_MERGEDSAVE</span>}
<span class="linenr">48: </span>    <span style="color: #483d8b;">echo</span> 1&gt;&amp;2 <span style="color: #8b2252;">"Oops! Conflict markers detected in $_MERGED."</span>
<span class="linenr">49: </span>    <span style="color: #483d8b;">echo</span> 1&gt;&amp;2 <span style="color: #8b2252;">"Saved your changes to ${_MERGEDSAVE}"</span>
<span class="linenr">50: </span>    <span style="color: #483d8b;">echo</span> 1&gt;&amp;2 <span style="color: #8b2252;">"Exiting with code 1."</span>
<span class="linenr">51: </span>    <span style="color: #ee0000;">exit</span> 1
<span class="linenr">52: </span><span style="color: #ee0000;">fi</span>
<span class="linenr">53: </span>
<span class="linenr">54: </span><span style="color: #ee0000;">exit</span> 0
</pre>
</div>

#### Script Modifications

To use the script successfully you may have to adapt it.

1. Check if the required tools are available and adapt their path if necessary (line 11-15).
2. Adapt the poor man's check for console (vs. graphical display) to your needs (line 34-39).


### Git Configuration

To add the Ediff as default diff and merge tool, you need the following in one of your Git configurations (preferably in `~/.gitconfig`):

```
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

Just replace the `/PATH/TO/` parts in the above snippet and leave the arguments ($LOCAL etc.) as they are. These three or four variables are provided by git difftool and git mergetool respectively.

### Caveats
1. To perform the check for conflict markers (line 45ff) the script needs to wait for *emacsclient* to finish. This means that you cannot use *emacsclient* with the option `–no-wait`.<br><br>If you desperately need that option and want to add it anyway (in line 38) then you should remove the check for conflict markers (line 44-52) as it would always be successful. Additionally, you must set *mergetool.ediff.trustExitCode* to `false`.

2. If conflict markers are found after *emacsclient* has returned, the script exits with code 1. With *mergetool.ediff.trustExitCode* set to `true` Git will then restore the original (conflict) version of the file, thus throwing away everything you've done in Emacs. That's okay if you wanted to start over anyway. However, losing your edits may not be desired in some cases (e.g. you just forgot to remove a conflict marker). Therefore, the script saves your edited version away before exiting so that you can always retrieve what you've done (see line 46f).

## Footnotes

<a name="myfootnote1">1</a>: http://www.gnu.org/software/emacs/manual/html_node/ediff/


Adapted from: `http://ulf.zeitform.de/en/documents/git-ediff.html'

author: Ulf Stegemann (ulf@zeitform.de)