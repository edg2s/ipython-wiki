Autosave in the IPython Notebook

This is a proposal for simple autosave in the IPython Notebook.

There are two basic notions of autosave:

1. old desktop-style: autosave only creates backups, user action is the only way to create a 'real' save, but can restore from autosave if newer than last 'real' save (emacs, MS Word, etc.).
2. modern / webapp-style: autosave is *real* save, but user may explicitly save a checkpoint or version, which will can be restored later (All webapps, really, OS X Versions, etc.).

This proposal is an implementation of the latter style,
which is more appropriate for a webapp like the notebook.

## Autosaving

The notebook should be automatically saved periodically.
These are real, regular saves, overwriting the file on the filesystem,
exactly as save is implemented today.

## Checkpointing

The user can manually 'save a version' that will never be clobbered by autosave.
In a primitive file-based implementation,
this amounts to a regular save, plus a copy to another location.
To avoid implementing a full VCS, a simple implementation would only support one user-specified checkpoint.
The checkpoint should be in a location that is discoverable from the notebook,
and consistent across notebook server sessions (i.e. keyed by notebook_id will not work for FileNB).

If we ever implement multiple checkpoints and history navigation,
this should probably just use git, and require that the project be a repo.

## Reverting to checkpoint

There will only be one checkpoint, and the user should have the option of reverting
the notebook state to the checkpoint.

A revert does:

- load the notebook from the checkpoint
- write it to the 'canonical' location
- leaves the checkpoint file untouched


## Deleting checkpoints

In addition to creating checkpoints, we may want a mechanism to delete checkpoints.

## Issues

- How do we choose autosave frequency?  Should it be a function of how long saves take
  (in case of large notebooks and/or slow connections)?
- should we trigger autosave on particular events, such as finishing the output of a given cell,
  rather than just a timer?

## Why not autosave backups?

The first / old-fashioned approach is more common in some contexts (particularly desktop editors).
It has the advantage of not changing the traditional save model *at all*,
only adding autosaved backups to protect against crash, etc.
I would argue that this is the wrong approach for a web-based application,
where *true* autosave is generally expected.
Having autosaved backups also makes loading a document much more complicated,
because the following logic must be followed every time a notebook is opened:

- does the notebook have an autosaved backup?
  - yes, is the backup newer than the manual saved version?
    - yes - prompt the user: "Load autosaved backup (DATE), newer than last save (DATE)?"
      - autosaved - copy autosave to real location, delete autosave
      - regular - delete autosave
    - no - delete the autosave, it shouldn't have existed in the first place

which is particularly cumbersome in a webapp, where this requires multiple HTTP requests,
and can become inconsistent if there are multiple clients viewing the same file.

Note that when you have multiple viewers of a notebook, this prompt would actually be expected to appear almost *every time* someone opens the notebook.

Compare with the proposed implementation,
where it is unambiguous which file should be opened at load time,
as revert-to-checkpoint is an explicit user action.

It is also much more difficult to have a clean 'unsaved changes, close without saving?'
dialog in a webapp than a desktop app,
so we would be much more likely to have autosaved backups that shouldn't exist littering the filesystem, which is gross.

I also think it is better to encourage users to explicitly decide to "remember this state",
rather than continue to enforce the old model,
where it is assumed that the document you are editing does not actually exist,
and it will go away if you just close it without saving.

## A  better approach

The model I have in my head of autosave is along the lines of [OS X Versions](http://support.apple.com/kb/ht4753),
implemented via git.
I do not believe it is appropriate for IPython (at least not by default),
but I will include a description here anyway.

1. every autosave is a regular filesystem save
2. checkpoint does `git add notebook.ipynb && git commit -m 'notebook checkpoint'` (OR just `git add notebook.ipynb`)
3. revert gives a view of the history of the file in git, and checks out with `git checkout <hash> notebook.ipynb`

The reasons not to do this in IPython (by default, anyway):

1. we don't want the notebook to depend on git (or any other VCS)
2. I wouldn't want to support any VCS other than git
3. creating a checkpoint when a commit may be in the middle of staging could be messy