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

The text was in `src/vs/workbench/contrib/files/browser/views/explorerView.ts`
and I saw that in its context of the `registerAction2` there was also a `run`
action to go along with it.

This is a place a could place a breakpoint and start stepping over and into
stuff from there.

- `src/vs/workbench/services/commands/common/commandService.ts` `CommandService.executeCommand`

  From here we go to `_tryExecuteCommand` on the same class.

- `src/vs/workbench/services/commands/common/commandService.ts` `CommandService._tryExecuteCommand`

  This fetches the command definition from `CommandsRegistry` and the command
  definition then has a `handler` field on it which is the backing function of
  the command.
  We can hover over `.handler` when debugging that line and see the name of
  the function that's there at runtime.
  
- `src/vs/workbench/contrib/files/browser/fileActions.ts` `openExplorerAndCreate`

  This creates a new node in the Explorer view and marks it as editable which
  changes the UI of the node's label to a text box which gets submitted when
  the Enter key is pressed or when the field is blurred.
  The handler for that is the `onFinish` and in this case it calls `onSuccess`
  which determines if we're creating a file or a directory and passes the call
  to the explorer service.
  
- `src/vs/workbench/contrib/files/browser/explorerService.ts` `ExplorerService.applyBulkEdit`
- `src/vs/workbench/contrib/bulkEdit/browser/bulkEditService.ts` `BulkEditService.apply`
- `src/vs/workbench/contrib/bulkEdit/browser/bulkEditService.ts` `BulkEdit.perform`
- `src/vs/workbench/contrib/bulkEdit/browser/bulkEditService.ts` `BulkEdit._performFileEdits`
- `src/vs/workbench/contrib/bulkEdit/browser/bulkFileEdits.ts` `BulkFileEdits.apply`

  We go through this jungle of abstractions which add extra bells and whistles
  for undoability etc. and finally we come back to something specifically
  related to the new file creation flow.

- `src/vs/workbench/contrib/bulkEdit/browser/bulkFileEdits.ts` `CreateOperation.perform`

  `CreateOperation` implements applying `CreateEdit`s created above and passed
  down to it.

- `src/vs/workbench/services/workingCopy/common/workingCopyFileService.ts` `WorkingCopyFileService.create`

  This class takes care of the file operations at the level where stuff like
  file participants are executed and ultimately the file system operations are
  handled even one more level down.
  
- `src/vs/platform/files/common/fileService.ts` `FileService.createFile`

  This service implements the actual persisting to the disk.
  Through `createFile` we get to `writeFile` and in there we can see an
  interesting call to `mkdirp`.

The `mkdirp` is a dead giveaway because the Unix command for creating a path
is `mkdir -p` and the associated comment also notices this call creates the
path recursively until all of the component folders exist.

So now we know how the file creation flow manages to handle the `/`s.
It uses `mkdir` to ensure the whole path exists before persisting the actual
file, so when a slash is a part of the files name, the directories are taken
care of.

We want the same for the file rename flow so I set out to inspect how that
worked hoping a pattern would emerge and I would find a nice spot where to use
the newly found `mkdirp` function to the same effect.

I started off with the "Rename..." label followed by distinguishing the search
results with numerical suffixes again to find the right one.

That ended up being in `src/vs/workbench/contrib/files/browser/fileActions.ts`
named `TRIGGER_RENAME_LABEL` and the close-by `RENAME_ID` is shared with a
different part of the codebase here:
`src/vs/workbench/contrib/files/browser/fileActions.contribution.ts`.

In this file, we see the `registerCommandAndKeybindingRule` call has `handler`
which is the entry point for what happens when the rename context menu item is
pressed.

- `src/vs/workbench/contrib/files/browser/fileActions.ts` `renameHandler`

  This method pulls the same trick with setting the Explorer node to editable
  state and waiting for the Enter press or the blur event to start processing
  the new name.
  
  The rest seems pretty similar to what we've seen in the creation flow:
  
- `src/vs/workbench/contrib/files/browser/explorerService.ts` `ExplorerService.applyBulkEdit`
- `src/vs/workbench/contrib/bulkEdit/browser/bulkEditService.ts` `BulkEditService.apply`
- `src/vs/workbench/contrib/bulkEdit/browser/bulkEditService.ts` `BulkEdit.perform`
- `src/vs/workbench/contrib/bulkEdit/browser/bulkEditService.ts` `BulkEdit._performFileEdits`
- `src/vs/workbench/contrib/bulkEdit/browser/bulkFileEdits.ts` `BulkFileEdits.apply`

  So we do get a pattern here and the difference now is the branch which this
  time maps `RenameEdit` to `RenameOperation`.

- `src/vs/workbench/contrib/bulkEdit/browser/bulkFileEdits.ts` `RenameOperation.perform`
- `src/vs/workbench/services/workingCopy/common/workingCopyFileService.ts` `WorkingCopyFileService.move`
- `src/vs/workbench/services/workingCopy/common/workingCopyFileService.ts` `WorkingCopyFileService.doMoveOrCopy`

This looks like the perfect space to use `mkdirp` and leave the rest of the
infrastructure to do its thing.
And then I will finally get renames with slashes to work and will be able to
contribute the fix!

Wait, what is _that_?

```typescript
private async doMoveCopy(sourceProvider: IFileSystemProvider, source: URI, targetProvider: IFileSystemProvider, target: URI, mode: 'move' | 'copy', overwrite: boolean): Promise<'move' | 'copy'> {
	if (source.toString() === target.toString()) {
		return mode; // simulate node.js behaviour here and do a no-op if paths match
	}

	// validation
	const { exists, isSameResourceWithDifferentPathCase } = await this.doValidateMoveCopy(sourceProvider, source, targetProvider, target, mode, overwrite);

	// delete as needed (unless target is same resurce with different path case)
	if (exists && !isSameResourceWithDifferentPathCase && overwrite) {
		await this.del(target, { recursive: true });
	}

	// create parent folders
	await this.mkdirp(targetProvider, this.getExtUri(targetProvider).providerExtUri.dirname(target));
```

What is the `mkdirp` doing there?
Did someone add this in an unreleased version of VS Code?

I took a few moments to go back to VS Code and do a quick sanity check to make
sure this wasn't already working in the released version.

And. It. Was.

I don't know why, but I did not think to test this didn't actually work before
I even opened the issue.
I remember being bothered by this for a long time but I did not think to check
it wasn't fixed before I went to file the issue. :D

Anyway, all's not lost, there is nothing to contribute but there was enough to
be learnt that might come in handy the next time I am looking to contribute to
VS Code!

As one last bit, let's take a look at how long this has been in VS Code proper
using Git blame:

<https://github.com/microsoft/vscode/blame/f043f49fc50ac2e984327f69ff9eb9718a396114/src/vs/platform/files/common/fileService.ts#L741>

_At least_ two fricking years. :)
I could have been using this for two years now instead of instinctually
complaning to myself that this doesn't work and doing the move and rename in
separate  steps. :D

I closed the VS Code issue.
