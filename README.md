# arcbbs
What I can find of !ARCbbs, my BBS software for the Acorn Archimedes (last seen ~1994). I've not tried to build it, but it did use the Acorn C compiler. Directory structure is strange, because "." was the directory separator on RISC OS and hence .c files are in c/, same with .h and .s files.

It seems I had some makefiles, but also some scripts that I ran to build bits of the server.

I'm pretty sure there are some BASIC files in here, but as filetypes weren't preserved in the tar files I extracted from disk images, YMMV. For example, !TestDoor/!RunImage is almost certainly basic. Suggestions as to how to append the type (,ffb?) in git are appreciated.


Here are the build notes from September 11th, 1992, which is, I believe, what I gave to Dave & Dave of Arcade BBS back in the day:

To start:

  Create the o directory and put a copy of a small arcbbs
  (for testing) in the directory.

  Set next slot to about 2Mb, and type:

  makebbs      (makes normal bbs)
  makebbsl     (makes local)
  makebbsm     (makes fido)
  makeserv     (makes server)
  makearea     (area editor)
  makearc      (sparkmail)
  makedoor     (door module)
  makeedit     (user editor)
  makelocald   (local downloader)
  makemod      (bbs file server module)
  makemug      (mug module - not used, precursor to doors)
  linkb        (batch upload)
  linku        (local upload)

Some other files:

fileform - proposal for a text filebase list format.
thought - you might recognise this one!
via* - via lists for the linker.
cmodhdr - mug stuff
new.* - snippets of new code I was working on, eg new fido
        module, new msg base structures.

Tips:

- Try and leave servcomm.c and mail.s alone - these are
  mailbase things.

- General mailbase layout is: area file holds number of msgs &
  first & last message pointers (into the lookup file). Messages
  in the lookup file are linked together by the .next and .last
  values (these are -1 when not applicable, ie at start or end
  of chain). Private mail is held the same way, with the start,
  end & count held in the userlog entry. Text is stored in the
  data file - it looks for a space big enough for the whole msg,
  then fills it (it uses 256 byte blocks). Differnt flags are
  used in the data freespace map for start of message text block
  and continuation (I think 3 is header and 4 is continuation).

- When a new public msg/file is entered, it adds to the end of
  the lookup file. Private mail uses the freespace map and uses
  any space inside the lookup file.

- fido.c holds all the fido stuff: for the server (import and
  export routines) and for the slave when it needs to write a
  fidomail. Really this should be changed - in the newer version,
  fido.c was only used in the server, not in the slave - the
  manual import and export from the sysop menu is really a
  hangover from a long time back. The new version also stored the
  seenby-s and paths as binary at the end of the message (one
  word, ie 2x16bits per entry) - much easier to sort, quicker,
  etc etc. I may help with this bit later, so don't do it yet!

- There are *many* duplications of eg write mail, which are
  all mostly the same. 1.70 had these all removed and replaced
  with a 'super write mail' which was flexible enough to handle
  all of them - this might be an idea here too.

If you want any help, ask!

Hugo

