This directory holds my personal patches for the software I use, and patched checkouts of that software.
Rules for writing patches:
- Always put the patches in `./patches/${NAME}/`.
- To apply, first check if the given program has been cloned to `./src/`. If so, make sure to `git pull`. If not, `git clone` it.
- The patches should be as small as possible, to promote ease of maintenance for me.
- Never bother with writing, editing, or running tests.
- Never bother with writing or editing documentation.
