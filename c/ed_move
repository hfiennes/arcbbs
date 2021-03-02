/*
 * module:
 *      ed_move.c       version 1.3
 *
 * purpose:
 *      this module contains the routines to handle editor cursor motion 
 *
 * external entry points:
 *      cursor          put the cursor where it belongs
 *      csr_*           cursor motion commands
 *      repaint         redraw the entire screen
 *      update          redraw the last part of a line on the screen
 *
 * internal entry points:
 *      move            update screen position
 */
#include "ed_rtns.h"
#include "ed_state.h"

int     badcursor;              /* update has goofed up the cursor */

/*
 * routine:
 *      move
 * 
 * purpose:
 *      to move the cursor or screen
 *
 * parms:
 *      desired row and column (1 based)
 *
 * returns:
 *      void
 *
 * algorithm:
 *      check column for absurdity
 *      check row for reasonableness
 *      extend the file if necessary
 *      determine the new display start position
 *      update the display
 */
void move( row, col )
 int row, col;
{       register int i;
        register int nrows;

        /* ignore absurd columns */
        if (col < 0 || col >= maxcols)
        {       bell();
                return;
        }

        /*
         * absurd rows are allowed - since we do the sanity checking 
         *        for all of the cursor motion commands
         */
        if (row < 0)                            /* not before first line */
                row = 0;

        if (row >= numlines + maxrows - 1)      /* not after last screen */
                row = numlines + maxrows - 2;

        if (row >= maxlines)                    /* not after max file size */
                row = maxlines-1;

        /* check for downwards scrolls */
        nrows = firstline - row;
        if (nrows > 0  &&  nrows < maxrows  &&  c_uindex)
        {       /* back-scroll new lines onto the screen */
                index( -nrows );
                firstline -= nrows;

                /* fill in the new lines with the appropriate text */
                for( i = firstline + nrows - 1; i >= firstline; i-- )
                        update( i, 0, 1 );
        }

        /* check for upwards scroll */
        nrows = row - (firstline + maxrows - 1);
        if (nrows > 0 && nrows < maxrows && c_dindex)
        {       /* scroll new lines onto the bottom of the screen */
                index( nrows );
                firstline += nrows;

                /* fill in the new lines at the bottom */
                for( i = firstline+maxrows-nrows; i < firstline+maxrows; i++ )
                        update( i, 0, 1 );
        }

        /* if it isn't on the screen yet - it's repaint time */
        if (row < firstline || row >= firstline + maxrows)
        {       /* check for scroll back on brain damaged terminals */
                if (!c_uindex && (row == firstline - 1))
                        firstline = row - (maxrows/3);
                else
                        firstline = row;

                /* perform some basic sanity checks on our new choice */
                if (firstline < 0)
                        firstline = 0;
                else if (firstline > maxlines - maxrows)
                        firstline = maxlines - maxrows;

                /* redraw the entire screen */
                clrscrn();
                for( i = firstline + maxrows - 1; i >= firstline; i-- )
                        update( i, 0, 1 );
        }

        /* desired row is on the current screen - so it's down to cursors */
        if (row == currow && !badcursor)
                rel_cur( 0, col - curcol );
        else
                abs_cur( row - firstline, col);

        /* note the new cursor position for future reference */
        curcol = col;
        currow = row;
        badcursor = 0;
}

/*
 * routine:
 *      cursor
 *
 * purpose:
 *      to re-position the cursor where it belongs after it has been moved
 */
void cursor()
{       abs_cur( currow - firstline, curcol);
        badcursor = 0;
}

/*
 * routines:
 *      crs_pgup, crs_pgdn
 *
 * purpose:
 *      to page the display up and down
 *
 * notes:
 *      most cursor motion is accomplished with move, which attempts
 *      to minimize scrolling.  These operations are specifically
 *      intended to redraw the entire screen - so they duplicate some
 *      of the work that move does - but doing it a bit differently
 */
void csr_pgup() /* cursor backwards one page */
{       register int i;

        /* make sure a full page is possible */
        if (firstline > maxrows)
        {       firstline -= maxrows;
                currow -= maxrows;
                curcol = 0;
                repaint();
        } else  /* partial scroll back in first page */
                move( 0, 0 );
}

void csr_pgdn() /* cursor forwards one page */
{       register int i = firstline + maxrows;

        /* make sure a full page is possible */
        if (i <= numlines - maxrows)
        {       firstline += maxrows;
                currow += maxrows;
                curcol = 0;
                repaint();
        } else  /* partial scroll forward in last page */
                move( numlines, 0 );
}

/*
 * routine:
 *      update
 *
 * purpose:
 *      to update some or all of a particular line on the screen
 * 
 * parms:
 *      file row, col to start updating
 *      whether or not row is already blank
 */
void update( row, col, blank )
 int row, col;
{       register char *buf;
        char *shift;
        int len;

        /* find the line in question */
        buf = LINEBUF( row ) + col;
        len = valid( buf, maxcols - col );

        /* see if there is nothing to put up */
        if (blank  &&  len == 0)
                return;

        /* see if there are any attribute shifts to deal with */
        shift = LINESHIFTS( row );
        if (shift != 0)
                shift += col;

        /* turn the file row into a screen row */
        row -= firstline;

        /* check for the deadly last byte on screen */
        if (c_autoscroll  &&  row == (maxrows - 1)  &&  (len + col) == maxcols)
                len--;

        /* start at the designated part of the line */
        abs_cur( row, col );
        badcursor = 1;          /* note that we've goofed it up */

        /* if there is anything to update in the remainder, do so */
        if (len > 0)
        {
                if (shift)      /* there are attributes to deal with */
                {       register int i;
                        char atr = curatr;

                        for( i = 0; i < len; i++ )
                        {       /* make sure attribute is right for each char */
                                if (atr != shift[i])
                                {       atr = shift[i];
                                        set_atr( atr );
                                }
                                outchr( buf[i] );
                        }

                        /* and restore screen attribute to previous value */
                        if (atr != curatr)
                                set_atr( curatr );
                } else          /* no attributes on this line, use default */
                {       if (curatr != '0')
                                set_atr( '0' );

                        outline( buf, len );

                        if (curatr != '0')
                                set_atr( curatr );
                }
        }
        /* clear anything after the text we've written out */
        if (!blank  &&  len + col < maxcols)
                clrline();
}

/*
 * routine:
 *      repaint
 *
 * purpose:
 *      to redraw the screen (for initialization, resumption, or error recovery)
 */
void repaint( )
{       register int i;

        term_init( 0 );
        clrscrn();
        insertion( insmode );   /* start in the proper insert mode      */

        /* for each line of the screen */
        for( i = firstline + maxrows - 1; i >= firstline; i-- )
                update( i, 0, 1 );

        /* and lastly, position the cursor */
        cursor();
}

/*
 * routines:
 *      csr_{left,right,up,down,home,end,nl,cr,pgup,pgdn}
 *
 * purpose:
 *      trivial cursor motion
 *
 * parms:
 *      none
 *
 * returns:
 *      void
 *
 * algorithm:
 *      call move to perform the appropriate display update
 */
void csr_left() /* cursor left one column */
{       if (curcol > 0) 
                move( currow, curcol - 1 );
        else 
                move( currow - 1, maxcols - 1 );
}

void csr_right()        /* cursor right one column */
{       if (curcol < maxcols-1) 
                move( currow, curcol + 1 );
        else 
                move( currow + 1, 0 );
}

void csr_up()   /* cursor up one row */
{       
        move( currow - 1, curcol );
}

void csr_down() /* cursor down one row */
{       
        move( currow + 1, curcol );
}

void csr_home() /* cursor to start of file */
{       move( 0, 0 );
}

void csr_end()  /* cursor to end of file */
{       move( numlines-1, 0 );
}

void csr_cr()   /* cursor to start of line */
{       
        move( currow , 0 );
}

void csr_eol()  /* cursor to end of line */
{       register int i;
        
        i = valid( LINEBUF( currow ), maxcols );
        if (i == maxcols)
                i--;

        move( currow , i );
}

void csr_nl()   /* cursor to start next of line */
{       
        int nextrow = currow + 1;
        
        /* newline does more than cursor motion, what is mode dependent */
        if (editmode == 'I')            /* auto insert - split the line */
                c_brkln();
        else if (editmode == 'R')       /* auto replace - clear to eol  */
                c_clrln();
        else if (curcol >= valid( LINEBUF(currow), maxcols ))
                LINEFLGS(currow) &= ~SOFT_CR;

        /* in all cases, we end up on the new line */
        move( nextrow, 0 );
}

void csr_tab()  /* cursor to next tab stop */
{       register int newcol;

        if (editmode == 'I' || editmode == 'R')
        {       /* in an editing mode, tab generates appt number of blanks */
                do { text( ' ' ); } 
                while( (curcol < maxcols) && (curcol % tabwidth) );
        } else
        {       /* in any other mode, it is only cursor motion */
                for( newcol = curcol + 1; newcol % tabwidth; newcol++ );
                if (newcol >= maxcols)
                        newcol = 0;

                move( currow, newcol );
        }
}

void csr_btab() /* cursor to previous tab stop */
{       register int newcol = curcol;

        if (newcol == 0)
                newcol = maxcols;
        while( --newcol % tabwidth );

        move( currow, newcol );
}

/*
 * routines:
 *      csr_fword, csr_bword
 *
 * purpose:
 *      to space or backspace the cursor one word
 *
 * algorithm:
 *      forward:  skip forward over text, and then over white space
 *      backward: skip back over white space, and then over text
 *      hitting either margin wraps to the opposite margin on same line
 */
void csr_fword()        /* cursor foward to start of next word */
{       register char *s;
        register int col;
        int skipped = 0;

        /* we loop because we may have to go through multiple lines */
        while( currow < numlines && currow < maxlines )
        {       /* get the line we're working on */
                s = LINEBUF( currow );
                col = curcol;

                /* skip to the end of this word */
                if (skipped == 0 && col < maxcols && s[col] != ' ')
                {       while( ++col < maxcols  &&  s[col] != ' ');
                        skipped++;
                }

                /* skip to the start of the next word */
                while( col < maxcols  &&  s[col] == ' ')
                        col++;

                /* if we found a new word, we're done */
                if (col < maxcols)
                {       move( currow, col );
                        return;
                } 

                /* if we've hit the hard end of buffer, we're done */
                if (currow == maxlines - 1)
                {       move( currow, maxcols - 1 );
                        break;
                }

                /* otherwise, continue search on next line */
                move( currow+1, 0 );
        }

        /* we didn't find another word */
        bell();
}

void csr_bword()        /* cursor back to start of previous word */
{       register char *s; 
        register int col;

        /* we loop because we may have to go through multiple lines */
        while (currow >= 0 || curcol > 0)
        {       /* now, lets look at the line we're on */
                s = LINEBUF(currow);
                col = curcol;

                /* skip back past any white space */
                while( --col > 0  &&  s[col] == ' ');

                /* skip back to start of this word */
                while( col > 0  &&  s[col-1] != ' ' )
                col--;

                /* see if we made it to a word */
                if (col >= 0 && s[col] != ' ')
                {       move( currow, col );
                        break;
                }

                /* see if there are previous lines to go to */
                if (currow == 0)
                {       move( 0, 0 );
                        break;
                }

                /* otherwise, we should back up to prev line and try again */
                if (currow > 0)
                        move( currow - 1, maxcols - 1 );
        }
}
