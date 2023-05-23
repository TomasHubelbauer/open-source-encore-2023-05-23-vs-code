# Open Source Encore: VS Code create path while renaming with `/`

VS Code has a feature where when creating a new file, if its name contains `/`s,
the path the directory the files is being created in concatenated with the path
given by the slashes gets created (if it doesn't already exist) and the new file
gets placed in this final directory.

For years I have been missing the ability to do this when renaming files, too.
Got a file in a directory and could use it being in a subdirectory and maybe
even named something else?
Not problem, just prepend the subdirectory name to the file name while renaming!
That was my dream for some long and today I set out to advocate for this feature
to be included in VS Code and I felt pretty good about my justification seeing
as the precedent was already there with the new file creation flow supporting
this behavior.

I opened this VS Code issue to track my plea:
<https://github.com/microsoft/vscode/issues/183203>

Wanting to be a good netizen, I set out to figure out how to contribute this
myself so that the VS Code team is more inclined to take the idea in if it comes
with a working implementation.

I've contributed bits to VS before, but it has been a long time and I wasn't
even sure how to build and run it anymore.
So, the first step was to find the contribution guide.
VS Code has <https://github.com/microsoft/vscode/blob/main/CONTRIBUTING.md> but
what is honestly rather odd is that there is so much prose and just a single
line detailing what to do when meaning to contribute code!
And that line is a link here:
<https://github.com/microsoft/vscode/wiki/How-to-Contribute>

This helped me figure out that I needed to install Node 18, global Yarn and when
that was done, I could finally run `yarn` to install the dependencies and build. 
I then switched over to the `yarn watch` command so I didn't need to pay any
mind to building the code after making my changes.

Next I used the `./scripts/code.sh` command to run the built VS Code binary and
do my testing there.
Pretty quickly though I realized that I can't access `console.log` outputs in
the output of this script easily (if at all) so I looked for a better way.

The scope of the VS Code codebase absolutely calls for bigger guns, so I found
a VS Code Debugger configuration simply called VS Code which also served to run
the built app, but in addition allowed me to attach to it and place breakpoints
at points of interest which I was investigating.

This makes the research much quicker and allows for cool stuff like hover tips
for runtime types and other stuff which is extremely handy when the debugger is
set up well.
Unfortunately setting up debug configurations still isn't as easy as placing a
`console.log` in the source code so I rarely bother to do it, but I absolutely
appreciate that it is set up for VS Code and could not imagine working in any
other way on such a massive codebase.

Anyway, since I knew I wanted to replicate a feature already seen in the new
file creation workflow, I first started by investigating how that even worked
end to end.
I mapped all of the code sites that are involved and a bit about what they do.
I paid particular interest to the part of this flow which takes care of the `/`
handling to make the path for the new file if its name contains slashes.

I started off by looking for the "New File..." string in the VS Code codebase.
I found several matches via full-text search so I renamed all of them to add a
unique number at the end and then re-run the debugger.
After hovering over the New File button in the Explorer pane I was able to find
which of the ~seven previously identical strings I was looking at.

WIP
