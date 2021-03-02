/*
 * module:
 *      ed_cmd.c        version 1.5
 *
 * purpose:
 *      this module contains the basic commands for the in_core editor
 *
 * external entry points:
 *      c_insmd         insert mode
 *      c_bspc          destructive backspace
 *      c_del{ch,ln,wd} delete char/line/word
 *      c_undel         un-delete/restore deleted line
 *      c_insln         insert line
 *      c_brkln/joinln  break and join lines
 *      c_autor/autoi   auto insert/replace modes
 *      c_cmd           command mode
 *      c_find          find a string
 *      c_rpl           replace a string with another
 *      c_rplall        replace throughout file 
 *      c_refill        refill entire document for nicer margins
 *      color           process color/attribute shifts
 *      code            put a special code into the file
 *
 */
#include "ed_rtns.h"
#include "ed_state.h"

/*
 * routine:
 *      c_insmd
 *
 * purpose:
 *      to toggle insert mode
 */
void c_insmd()
{       
        insmode = !insmode;
        insertion( insmode );
}

/*
 * routine:
 *      c_autoi
 *
 * purpose:
 *      to go into auto-insert mode
 */
void c_autoi()
{       insmode = 1;
        insertion( insmode );
        editmode = 'I';
}

/*
 * routine:
 *      c_autor
 *
 * purpose:
 *      to go into auto-replace mode
 */
void c_autor()
{       insmode = 0;
        insertion( insmode );
        editmode = 'R';
}

/*
 * routine:
 *      c_cmd
 *
 * purpose:
 *      to exit auto-insert/replace mode
 */
void c_cmd()
{       insmode = 0;
        insertion( insmode );
        editmode = 'C';
}

/*
 * routine:
 *      c_bspc
 *
 * purpose:
 *      destructive backspace
 *
 */
void c_bspc()
{       register int col = curcol - 1;

        /* make sure we are in a column where this is meaningfull */
        if (col < 0)
        {       bell();
                return;
        }

        /* back up to the character we are going to trash */
        move( currow, col );
        if (insmode)
        {       c_delch( 1 );
        } else
        {       text( ' ' );
                move( currow, col );
        }
}

/*
 * routine:
 *      c_delch
 *
 * purpose:
 *      to delete a character from the current position
 *
 * algorithm:
 *      copy the line buffer back one character
 *      shift a blank in at the right end
 *      do a delch on the screen
 *
 * parm:
 *      0       - called from another command
 *      else    - called directly by user keystroke
 */
void c_delch( arg )
 char *arg;
{       
        /* start by updating the in-memory text */
        textsuck( currow, curcol, 1 );

        /*
         * update the screen, ideally by sucking up one character
         *      but if we can't do that, we will redraw the line
         *      because redrawing is expensive, the commands that use
         *      delch pass us a 0 argument, so that we know that they
         *      will do the redraw when they are done.
         */
        if (c_del)
                clschar( 1 );
        else if (arg)
        {       update( currow, curcol, 0 );
                cursor();
        }
}

/*
 * routine:
 *      c_delln
 *
 * purpose:
 *      to delete the current line
 *
 * algorithm:
 *      do a circular shift of line buffers to acomplish scoll in buffer
 *      shift a new clear buffer in, in place of the one being lost
 *      close one line on the screen
 *      suck a new line onto the bottom of the screen
 */
void c_delln()
{       register int i;
        struct line *wrap;

        /* make sure there is something to delete */
        if (currow >= numlines)
        {       bell();
                return;
        }

        /* do a circular shift down of the buffers above it */
        wrap = lines[currow];
        for( i = currow+1; i < numlines; i++ )
                lines[i-1] = lines[i];
        
        /*
         * replace the buffer at the end of file with the previous save text
         * put the deleted line into the save text slot
         */
        lines[i-1] = lines[maxlines];
        textclr( i-1 , 0 );
        lines[maxlines] = wrap;
        
        /* close the elided line off the screen and suck up a new one */
        i = firstline + maxrows - 1;    /* line number of last line on screen */
        if (i >= numlines)
                i = numlines - 1;       /* last non-blank line on screen */

        if (c_dlines && currow != i)
        {       clsline(1);
                /* if we suck a new line onto the screen, update it */
                if (i < numlines - 1)
                        update( i, 0, 1 );
        } else
        {       while( i >= currow )
                        update( i--, 0, 0 );
        }
        
        /* FIX take advantage of indexing when deleting the first line */
        /* note that the file has gotten a line shorter */
        if (numlines > 1)
                numlines--;

        /* put the cursor back where it belongs */
        cursor();
}

/*
 * routine:
 *      c_delwd
 *
 * purpose:
 *      to delete to the end of the current word
 *
 * algorithm:
 *      use delch to delete consecutive non-blanks
 *      if there are other words on line delete blanks leading to next one
 */
void c_delwd()
{       register char *s;
        register int i;

        /* get the buffer */
        s = LINEBUF( currow );

        /* figure out how much non-white space there is to delete */
        for( i = 0; s[curcol + i] != ' '; i++ );
        
        /* if there is something else on the line, delete up to it */
        if (valid( &s[curcol+i], maxcols - (curcol + i)))
                while( s[curcol+i] == ' ' )
                        i++;

        /* update the in-core text */
        textsuck( currow, curcol, i );

        /*
         * update the screen
         *      this is easier if we can suck characters, but if not, we can
         *      always redraw the right half of the current row
         */
        if (c_del)
                clschar( i );
        else
        {       update( currow, curcol, 0 );
                cursor();
        }
}

/*
 * routine:
 *      c_delwdb
 *
 * purpose:
 *      to delete the previous word
 *
 * algorithm:
 *      space back over any preceding white space
 *      space back over consecutive non blanks
 *      delch the number of characters we spaced back over
 */
void c_delwdb()
{       register char *s = LINEBUF( currow );
        register int col = curcol;
        int nchars = 0;

        /* if we are at the start of a word, skip back one */
        if (col > 0 && s[col] != ' ' && s[col-1] == ' ')
        {       col--;
                nchars++;
        }

        /* skip back past any white space */
        while( col > 0  &&  s[col] == ' ')
        {       col--;
                nchars++;
        }

        /* skip back to start of this word */
        while( col > 0  &&  s[col-1] != ' ' )
        {       col--;
                nchars++;
        }

        /* go to the start of the previous word and delete it */
        move( currow, col );

        /* update the in-core text */
        textsuck( currow, curcol, nchars );
        
        /* update the on-screen text */
        if (c_del)
                clschar( nchars );
        else
        {       update( currow, curcol, 0 );
                cursor();
        }
}

/*
 * routine:
 *      c_insln
 *
 * purpose:
 *      to open a new line in front of the current one
 *      leaving cursor on the new line
 */
void c_insln()
{       register int i;
        struct line *wrap;

        /* make sure we don't already have too many lines */
        if (numlines == maxlines)
        {       bell();
                return;
        }
                
        /* if we are already at the end of the file, nothing to do */
        if (currow >= numlines)
                return;

        /* do a circular shift down of the buffers above it */
        wrap = lines[ numlines ];
        for( i = numlines; i > currow; i-- )
                lines[i] = lines[i-1];
        
        /* put this re-circulated buffer on the current line */
        lines[currow] = wrap;

        /* clear the saved buffer to all blanks again (if it wasn't already) */
        textclr( currow, 0 );
        
        /* update the screen */
        i = firstline + maxrows - 1;    /* number of last line on screen */
        if (i > numlines)
                i = numlines;           /* last non-blank line on screen */

        if (c_ilines && currow != i)
                insline( 1 );           /* scroll display down one line */
        else
        {       /* redraw each line from the current row downwards */
                while( i >= currow )
                        update( i--, 0, 0 );
        }

        /* FIX - take advantage of indexing for inserts at top of screen */

        /* note that the file is now a line longer */
        if (numlines < maxlines)
                numlines++;

        /* go to a good starting place on the new line */
        move( currow, 0 );
}

/*
 * routine:
 *      c_undel
 *
 * purpose:
 *      to restore a line of deleted text
 *
 * parms:
 *      none
 *
 * returns:
 *      void
 *
 * algorithm:
 *      open a new line in the file
 *      copy in the save text
 *      update the display
 *
 * notes:
 *      the extra buffer at the end of the file contains the saved text
 */
void c_undel()
{       
        /* start with a new line */
        c_insln();

        /* copy the saved text onto the new line */
        textcpy( maxlines, 0, currow, 0, maxcols );

        /* update the screen */
        update( currow, 0, 1 );
        cursor();

        /* make sure we know how big the file is */
        if (currow > numlines)
                numlines = currow + 1;
}

/*
 * routine:
 *      c_brkln
 *
 * purpose:
 *      to split the current line into two lines
 */
void c_brkln()
{       
        /* note that this line was not word wrapped */
        LINEFLGS(currow) &= ~SOFT_CR;

        /* split it at the current column */
        splitline( curcol, 0 );
}

/*
 * routine:
 *      c_clrln
 *
 * purpose:
 *      to clear to the end of the current line
 */
void c_clrln()
{
        /* note that this end of line was forced */
        LINEFLGS(currow) &= ~SOFT_CR;

        /* clear the text in the buffer */
        textclr( currow, curcol );

        /* clear the rest of the screen */
        clrline();

        /* make sure the cursor ends up in the right place */
        cursor();
}

/*
 * routine:
 *      c_joinln 
 *
 * purpose:
 *      to join two lines together
 */
void c_joinln()
{       register char *next;
        register int i;
        int len;

        /* you can't join unless there is something on the next line */
        if (currow >= numlines - 1)
                return;

        /* if second line ended with a soft CR, let first line inherit it */
        if (LINEFLGS(currow+1) & SOFT_CR)
                LINEFLGS(currow) |= SOFT_CR;

        /* find the first non-blank on the next line    */
        next = LINEBUF( currow + 1);
        for( i = 0; i < maxcols && next[i] == ' '; i++ );

        /* trash the text on the current end of this line */
        textclr( currow, curcol );

        /* copy len is lesser of room on this line or text on next line */
        len = maxcols - ((curcol > i) ? curcol : i );
        textcpy( currow + 1, i, currow, curcol, len );

        i = curcol;     /* remember what column we have to come back to */

        /* redraw the current line as it now is */
        update( currow, curcol, 0 );

        /* delete the next line */
        move( currow + 1, 0 );
        c_delln();

        /* and lastly, go back where we belong */
        move( currow - 1, i );
}

/*
 * routine:
 *      c_find
 * 
 * purpose:
 *      to find the next instance of a particular string in the file
 *
 * parms:
 *      if called from lexer, non-zero
 *      if called from c_rplall, zero
 *
 * returns:
 *      bool - whether or not the string was found
 */
char    findstring[80];         /* last string searched for */

int c_find( arg )
 char *arg;
{       register int r, c;

        /* prompt the user for the string to search for */
        if (arg)
                solicit( "Search for: ", findstring, sizeof findstring );

        if (findstring[0] == 0)
        {       bell();
                return( 0 );
        }

        /* start search one character past cursor position */
        if (curcol < maxcols-1)
        {       c = search( currow, curcol + 1, findstring );
                if (c >= 0)
                {       move( currow, c );
                        return(1);
                }
        } 

        /* then search forward through the file */
        for( r = currow + 1; r < numlines; r++ )
        {       c = search( r, 0, findstring );
                if (c >= 0)
                {       move( r, c );
                        return(1);
                } 
        }

        /* search from start of file to currow */
        for( r = 0; r < currow; r++ )
        {       c = search( r, 0, findstring );
                if (c >= 0)
                {       move( r, c );
                        return(1);
                }
        }

        /* no joy */
        bell(); 
        return( 0 );
}

/*
 * routine:
 *      c_rpl
 * 
 * purpose:
 *      to replace an instance of one string with another
 *
 * parms:
 *      if called from lexer, non-zero
 *      if called from c_rplall, zero
 *
 * returns:
 *      bool whether or not the replacement could be performed
 */
char rplstring[80];     /* string to replace with */
int c_rpl( arg )
 char *arg;
{       

        /* make sure we have a find string */
        if (findstring[0] == 0)
        {       bell();
                return( 0 );
        }

        /* make sure we're at the find string */
        if (search( currow, curcol, findstring ) != curcol)
        {       bell();
                return( 0 );
        }

        /* get something to replace it with */
        if (arg)
                solicit( "Replace with: ", rplstring, sizeof rplstring );
        if (rplstring[0] == 0)
        {       bell();
                return( 0 );
        }

        /* delete the old stuff and insert the new stuff in memory */
        textsuck( currow, curcol, strlen( findstring ) );
        (void) textins( currow, curcol, rplstring, strlen( rplstring ), 1 );

        /* if we're a command, update the line on the text on the screen */
        if (arg)
        {       update( currow, curcol );
                cursor();
        }

        return( 1 );
}

/*
 * routine:
 *      c_rplall
 *
 * purpose:
 *      to perform a replacement throughout the file
 *
 */
void c_rplall()
{       int rplcnt = 0;
        int i;
        char doit[32];

        /* make sure we have strings to substitute */
        if (findstring[0] == 0  ||  rplstring[0] == 0)
        {       bell();
                return;
        }
        
        /* make sure he really wants to do it */
        strcpy( doit, "yes" );
        solicit( "Perform replacement throughout file? ", doit, sizeof doit );
        if (doit[0] != 'y' && doit[0] != 'Y')
                return;

        /* do the replacement as many times as possible */
        for( ;; )
        {       if (c_find( 0 ) == 0)
                        break;
                if (c_rpl( 0 ) == 0)
                        break;
                rplcnt++;
        }

        /* figure out how many replacements we made */
        i = 0;
        if (rplcnt > 100)
                doit[i++] = '0' + (rplcnt/100);
        if (rplcnt > 10)
                doit[i++] = '0' + ((rplcnt%100) / 10);
        doit[i++] = '0' + rplcnt % 10;
        doit[i] = 0;

        /* tell him what we did */
        
        strcat( doit, ", (press return to continue)" );
        solicit( "replacements performed: ", doit, sizeof doit );

        /* then redraw the screen */
        repaint();
}

/*
 * routine:
 *      c_refill
 * 
 * purpose:
 *      to refill the current paragraph to pretty margins
 *
 * parms:
 *      none
 *
 * returns:
 *      void
 */
void c_refill()
{
        bell();

        repaint();
}

/*
 * routine:
 *      color
 *
 * purpose:
 *      to insert a color/rendition change into the text
 *
 * parm:
 *      pointer to the escape sequence that requested the color change
 *              second character of string is a number 0-7
 */
void color(str, len)    /* process a color change order */
 char *str;
 int len;
{       
        /* all the dirty work is handled by text */
        if (len >= 2)
        {       curatr = str[1];
                set_atr( curatr );
        }
}

/*
 * routine:
 *      control
 *
 * purpose:
 *      to put a control code into the file
 *
 * parm:
 *      pointer to escape sequence that requested the color change
 *       \ ^ <char>     for ctl-<char>
 *       \ X X          for hexadecimal X X
 */
void control(str, len)  /* process special code as text */
 char *str;
 int len;
{       int code = 0;
        register int i;

        /* make sure we have reasonable arguments */
        if (len != 3)
                return;

        /* check for a simple control */
        if (str[1] == '^')
                code = str[2] & 0x1f;
        else    /* hexadecimal code */
                for( i = 1; i < 3; i++ )
                {       code <<= 4;
                        if (str[i] >= '0' && str[i] <= '9')
                                code += str[i] - '0';
                        else
                                code += 9 + (str[i] & 0x0f);
                }
                
        text( code );
}
