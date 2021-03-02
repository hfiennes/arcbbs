
/* 
 * module:
 *      ed_text.c       1.2
 *
 * purpose:
 *      routines to manipulate the text in the line buffers, and in
 *      particular to handle the groaty details of attribute shifts
 *
 * contents:
 *      shiftalloc      allocate a shift buffer for a line
 *      shiftfree       try to free the shift buffer for a line
 *      splitline       routine to actually perform break line
 *      wordwrap        force a word wrap to the next line
 *      textcpy         copy text from one line to another
 *      textclr         clear text from end of line
 *      textsuck        suck text in a line
 *      textins         insert text in a line 
 *      text            put a character into the file
 *      indent          figure out the indent on specified line
 */

#include "ed_rtns.h"
#include "ed_state.h"

/*
 * routine:
 *      shiftalloc
 *
 * purpose:
 *      to allocate a shift buffer for a line
 *
 * parm:
 *      line number
 *
 * returns:
 *      pointer to new shift buffer (which has been added to line)
 *      NULL - unable to allocate shift buffer
 *
 * note:
 *      for convenience, it is perfectly OK to call this routine on a
 *      line that already has a shift buffer.
 */
char *shiftalloc( line )
 int line;
{       register char *s;
        register int i;

        /* make sure there isn't already one */
        if (s = LINESHIFTS( line ))
                return( s );

        /* try to buy one */
        s = (char *) malloc( maxcols );

        /* if we got it, initialize it */
        if (s != 0)
        {       for( i = 0; i < maxcols; i++ )
                        s[i] = '0';
                LINESHIFTS( line ) = s;
        }

        /* and tell the user what he got */
        return( s );
}

/*
 * routine:
 *      shiftfree
 *
 * purpose:
 *      try to free the shift buffer associated with a line
 *
 * parm:
 *      line number
 *
 * returns:
 *      TRUE  - buffer has been freed
 *      FALSE - buffer is still needed
 */
int shiftfree( line )
 int line;
{       register char *s = LINESHIFTS( line );
        register char *l = LINEBUF( line );
        register int i;

        /* make sure we have a buffer to free */
        if (s == 0)
                return( 0 );

        /* see if there are still any valid shifts in the buffer */
        for( i = maxcols ; i-- > 0; s++, l++ )
        {       /* default attribute shifts are boring */
                if (*s == '0')
                        continue;

                /* blanks shouldn't have attributes */
                if (*l == ' ')
                {       *s = 0;
                        continue;
                }

                /* looks like a valid shift to me */
                return( 0 );
        }

        /* 
         * this shift buffer is no longer necessary
         *      FIX - if I kept a secondary free list, this would be faster
         */
        free( LINESHIFTS( line ) );
        LINESHIFTS( line ) = 0;
        return( 1 );
}

/*
 * routine:
 *      splitline
 * 
 * purpose:
 *      to split a line at a particular point
 *
 * parms:
 *      column # of first column to be moved to new line
 *      is this a wrap (vs a simple split)
 *
 * returns:
 *      void
 *
 * algorithm:
 *      figure out what text needs to be copied
 *      add a new line to the file
 *      copy data from old line to new, deleting as we go
 *      move cursor to corresponding position on new line
 */
void splitline( first, wrap )
 int first, wrap;
{       register int i;
        int last;               /* col of last valid character on line  */
        int numchars;           /* number of characters to copy         */
        int offset;             /* distance between last and cursor     */
        int ind;                /* indent on line being split           */
        int usenext = 0;        /* insert text on next line             */

        /* make sure we have room to insert a new line */
        if (numlines >= maxlines)
        {       bell();
                return;
        }

        /* figure out what text needs to be moved */
        last = valid( LINEBUF(currow), maxcols );
        if (last > first)
                numchars = last - first;
        else
                numchars = 0;

        /* figure out the corresponding place for cursor on next line */
        offset   = curcol - first;

        /* text will have to go onto the next line */
        move( currow + 1, 0 );

        /*
         * if this is a word-wrap, we would like to try inserting the
         * wrapped text onto the next line - rather than creating a new
         * line.  We can only do this if the next line was created by a
         * wrap and there is still room on it for the wrapping word.
         */
        if (wrap && insmode && (numchars > 0) && (LINEFLGS(currow) & WRAP_OK))
        {       i = maxcols - valid( LINEBUF( currow ), maxcols );
                usenext = (i >= wrapmgn + numchars + 1);
        }

        /* 
         * if we are wrapping onto an existing line, preserve its indent
         * if we are wrapping onto a new line, preserve previous indent
         * if we are wrapping onto a new line, allocate it
         */
        if (usenext)
        {       ind = indent( currow );
        } else
        {       ind = o_autoindent ? indent( currow - 1 ) : 0;
                c_insln();
                if (wrap)
                        LINEFLGS( currow ) |= WRAP_OK;
        }

        /* if there is any text to wrap, do it */
        if (numchars > 0)
        {       /* if wrapping onto existing line, make room for it */
                if (usenext)
                        textins( currow, 0, 0, numchars+1, 1 );

                /* move text from end of previous line to start of this */
                textcpy( currow-1, first, currow, ind, numchars );
                textclr( currow-1, first );

                /* update the display to reflect the move       */
                update( currow-1, first, 0 );
                update( currow, 0, 1 );

                /* note that we have put text on this line      */
                if (currow >= numlines)
                        numlines = currow + 1;
        }
        
        /* put cursor in corresponding place (in new split lines)       */
        if (offset < 0)         /* negative offset means on old line    */
                move( currow - 1, first + offset );
        else                    /* positive offset means on new line    */
                move( currow, ind + offset );
}

/*
 * routine:
 *      wordwrap
 *
 * purpose:
 *      to wrap the last word on the current line onto the next line
 *
 * algorithm:
 *      figure out if a word-wrap is necessary prior to adding a char
 *      back up to start of word to be wrapped
 *      call splitline to break the line
 */
void wordwrap( chr )
 char chr;
{       register char *s = LINEBUF( currow );
        register int c;

        /* if we're about to put text in the zone, we must wrap */
        if (chr != ' ' && curcol >= maxcols - wrapmgn)
        {       /* find start of the word we're adding to */
                for( c = curcol; c > 0 && s[c-1] != ' '; c-- );
        } else if (insmode)
        {       /* look for text within the wrap-zone */
                for( c = maxcols; c >= maxcols - wrapmgn; c-- )
                        if (s[c-1] != ' ')
                                break;

                /* if we found none, no wrap */
                if (c < maxcols - wrapmgn)
                        return;         
                
                /* find the start of that word */
                while( c > 0 && s[c-1] != ' ' )
                        c--;
        } else  /* no wrap required at this time */
                return;

        /* if it is too far back, just wrap at the break point */
        if (c < (maxcols/2))
                c = maxcols - wrapmgn;

        /* note that this line was wrapped to an end */
        LINEFLGS(currow) |= SOFT_CR;

        /* split the line at the chosen wrap point */
        splitline( c, 1 );
}

/*
 * routine:
 *      textclr
 *
 * purpose:
 *      to clear the text at the end of a line
 *      
 * parms:
 *      row, col
 */
void textclr( row, col )
{       register char *l;
        register char *s;
        register int i;

        /* find the line and shift buffers */
        l = LINEBUF( row );
        s = LINESHIFTS( row );

        /* clear them */
        for( i = col; i < maxcols; i++ )
        {       l[i] = ' ';
                if (s)
                        s[i] = '0';
        }

        /* see if we can get rid of the attribute shift buffer */
        if (s)
                (void) shiftfree( row );
}

/*
 * routine:
 *      textcpy
 *
 * purpose:
 *      to copy text from one line to another
 *      
 * parms:
 *      source row, col
 *      dest row, col
 *      number of bytes to copy
 */
void textcpy( srow, scol, drow, dcol, count )
 int count;
{       register char *s, *d;
        register int c;

        if (count <= 0)
                return;

        s = LINEBUF( srow );
        s += scol;
        d = LINEBUF( drow );
        d += dcol;

        /* start by copying the text */
        for( c = count; c--; *d++ = *s++ );

        /* then go back for the attributes (if any) */
        s = LINESHIFTS( srow );
        if (s == 0)
                return;

        d = shiftalloc( drow );
        if (d == 0)
                return;

        /* copy all the shifts */
        s += scol; d += dcol;
        for( c = count; c--; *d++ = *s++ );

        /* just for jollys, see if see can 86 the buffer on the copied line */
        (void) shiftfree( drow );
}

/*
 * routine:
 *      textsuck
 *
 * purpose:
 *      to suck (delch) characters within a line buffer
 *
 * parms:
 *      row, col
 *      number of characters to suck
 *
 * returns:
 *      void
 */
void textsuck( row, col, count )
{       register char *l;
        register char *s;
        register int i;

        /* perform the delch in the buffer */
        l = LINEBUF( row );
        s = LINESHIFTS( row );

        for( i = col; i < maxcols-count; i++ )
        {       l[i] = l[i+count];
                if (s)
                        s[i] = s[i+count];
        }

        /* suck blanks onto the end of the line */
        while( i < maxcols )
        {       l[i] = ' ';
                if (s)
                        s[i] = curatr;
                i++;
        }

        /* maybe we can lose the shift buffer now */
        if (s  &&  curatr == '0')
                (void) shiftfree( currow );
}

/*
 * routine:
 *      textins
 *
 * purpose:
 *      to insert characters into a line buffer
 *
 * parms:
 *      row, col
 *      pointer to string to insert (if 0, insert only blanks)
 *      number of chars to insert
 *      is this an insert (vs a replace)
 *
 * returns:
 *      whether or not any text was actually moved
 *
 * notes:
 *      new text is inserted in the current attribute
 */
int textins( row, col, str, count, ins )
 char *str;
{       register char *l, *s;
        register int i;
        int movedsome = 0;

        l = LINEBUF( row );

        /* get the shift buffer, allocating a new one if necessary */
        s = LINESHIFTS( row );
        if (s == 0  &&  curatr != '0')
                s = shiftalloc( currow );

        /* if we're inserting, shift the buffer(s) to the right */
        if (ins)
                for( i = maxcols - 1; i >= col + count; i-- )
                {       l[i] = l[i-count];
                        if (s)
                                s[i] = s[i-count];
                        if (l[i] != ' ')
                                movedsome++;
                }

        /* copy in the new data into the buffers */
        for( i = 0; i < count; i++ )
        {       l[col + i] = str ? str[i] : ' ';
                if (s)  /* we must also update the attribute shift buffer */
                        s[ col ] = curatr;
        }

        /* if we're overstriking with normal text, maybe we're done w/ shifts */
        if (s && curatr == '0')
                (void) shiftfree( row );

        /* let user know if screen needs updating */
        return( movedsome );
}

/*
 * routine:
 *      text
 *
 * purpose:
 *      to add a character of text to the file
 *
 * parms:
 *      character to be added
 *
 * returns:
 *      void
 */
void text( c )
 int c;
{       int movedsome;

        /* check to see if we need to wrap this word */
        wordwrap( c );

        /* update the in-memory text */
        movedsome = textins( currow, curcol, &c, 1, insmode );

        /* see if we have added a new line */
        if (currow >= numlines)
                numlines = currow + 1;

        /* update the character on the screen */
        if (insmode)
                inschar( c );
        else
                outchr( c );

        /* update column and check for cursor wrap */
        scrncol++;
        curcol++;

        if (curcol >= maxcols)
                move( currow + 1, 0 );
        else /* check for botched screen do to no insert mode */
        if (movedsome && !c_ins)
        {       update( currow, curcol, 0 );
                cursor();
        }
}

/*
 * routine:
 *      indent
 *
 * purpose:
 *      to figure out the indentation on a line
 *
 * parms:
 *      line number
 *
 * returns:
 *      number of leading blanks on line
 */
int indent( row )
 int row;
{       register int i;
        register char *s;

        if (row >= numlines)
                return( 0 );

        s = LINEBUF( row );
        for( i = 0; s[i] == ' ' && i < maxcols; i++ );

        return( (i > maxcols/2) ? 0 : i );
}
