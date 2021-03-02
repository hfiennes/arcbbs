/*
 * module:
 *      ed_misc.c       version 1.4
 *
 * purpose:
 *      this module contains the initialization, I/O and miscelaneous
 *      utility routines used for editing text
 *
 * external entry points:
 *      ed_init         initialize the editor
 *      save/quit       exit the editor
 *      expand          expand a line into displayable form
 *      dexpand         read a line and process attributes
 *      search          look for a string within a line
 *      solicit         prompt the user for input
 *      valid           determine how many non-blanks are in a line
 *
 * principal data structures:
 *      lines           array of buffers for each line of the file
 */

#include "ed_rtns.h"
#include "ed_state.h"
#include "setjmp.h"
extern jmp_buf exited;

/* these defines enable features as well as setting ctl seqs    */
#define SOFT_CR_CODE    0x01    /* enable soft CR handling      */
#define COLOR_CODE      0x03    /* enable addributes in files   */

/* basic editor state information maintained in this module     */
int     currow;                 /* current row: 0 - MAXROW-1    */
int     curcol;                 /* current col: 0 - MAXCOL-1    */
int     curatr;                 /* current attribute code       */
int     firstline;              /* row # of first line on scrn  */

int     insmode;                /* insert mode: 0, 1            */

int     numlines;               /* # valid lines: 0 - MAX_LINES */
char    *filename;              /* name of the file we're in    */

struct line **lines;            /* information about each line  */

int     x_buflen;               /* buffer length from expand    */
int     x_linewid;              /* number of cols from expand   */

/*
 * routine:
 *      ed_init
 *
 * purpose:
 *      to initialize the basic editor
 *
 * parms:
 *      none
 *
 * returns:
 */
int ed_init( name )
 char *name;
{       register int i, j;
        register char *s;
        int fdesc;
        struct line *lp;
        char mybuf[256];
        char *dexpand();

        /* set up the initial state */
        currow = 0; curcol = 0; firstline = 0; numlines = 0;
        curatr = '0';
        insmode = 0;

        /* allocate the array of line buffer pointers */
        lines = (struct line **) malloc( (maxlines+1) * sizeof (struct line *) );
        if (lines == (struct line **) 0)
                return(1);

        /* allocate and initialize each line buffer and line buffer pointer */
        for( i = 0; i <= maxlines; i++ )
        {       lp = (struct line *) malloc( sizeof (struct line) + maxcols );
                if (lp == (struct line *) 0)
                        return( 1 );
                lines[i] = lp;

                /* each line starts out all blanks */
                s = lp->l_text;
                for( j = 0; j < maxcols; j++ )
                        s[j] = ' ';

                lp->l_flags = 0;
                lp->l_shifts = 0;
        }

        /* note the input file name and try to open it */
        filename = name;
        if (fdesc = openfile( name, 'R' ))
        {       /* read in each line */
                for( i = 0; i < maxlines; i++ )
                {       j = getline( fdesc, mybuf, sizeof mybuf );
                        if (j == 0)
                                break;

                        /* copy it into a line buffer, handling attributes */
                        s = dexpand( mybuf, j, i );

#ifdef  SOFT_CR_CODE
                        /* find the last non-blank in the line */
                        for( j = maxcols - 1; j >=0 && s[j] == ' '; j-- );

                        /* see if it was a soft-return indication */
                        if (j >= 0 && s[j] == SOFT_CR_CODE)
                        {       LINEFLGS(i) |= SOFT_CR;
                                s[j] = ' ';
                        }
#endif
                }

                closefile( fdesc );

                /* note the size of the input file and where the end is */
                numlines = i;

                /*
                 * put the cursor at a reasonable place
                 *      small file - at the start of the line after the eof
                 *      large file - at the start of the first line
                 */
                currow = (numlines >= maxrows) ? 0 : numlines;
                firstline = 0;
        }

        /* put up the initial screen */
        repaint( );
        return( 0 );
}
        
/*
 * routine:
 *      save
 *
 * purpose:
 *      to exit the editor and save the file
 */
void save()     /* write out the file and exit */
{       register int i;
        register char *s;
        int desc;

        /* open the file for output */
        desc = openfile( filename, 'W' );
        if (desc == 0)
        {       (void) outstr("Unable to re-open file for output\r\n");
                tio_clup();
                longjmp(exited,1);
        } 

        /* copy all the edit lines into the file */
        for( i = 0; i < numlines; i++ )
        {       s = expand( i, 0 );
#ifdef  SOFT_CR_CODE
                /*
                 * if this line ends with a soft return, put the SOFT RETURN
                 * character at the end of the line before writing it out.
                 * We can do this because the line buffer is maxcols+1 bytes
                 * long - the extra byte being reserved for this kind of stuff
                 *
                 * special crock - wwiv doesn't preserve indents on word-wrapped
                 *                 lines, so we check to see if the next line
                 *                 is indented, and if it is, we don't put the
                 *                 soft cr at the end of this line
                 */
                if ((LINEFLGS(i) & SOFT_CR)  &&  indent(i+1) == 0)
                        s[x_buflen++] = SOFT_CR_CODE;
#endif
                (void) putline( desc, s, x_buflen );
        }

        closefile( desc );      /* close the file */

        tio_clup();             /* clean up and exit */
        (void) outstr( "Saving " );
        (void) outnum( numlines );
        (void) outstr( " lines\r\n" );

        longjmp(exited,2);
}

/*
 * routine:
 *      expand
 *
 * purpose:
 *      to expand the attribute shifts in a line
 *
 * parms:
 *      row, col
 *
 * returns:
 *      pointer to buffer containing text (suitable for output)
 *      x_buflen (number of bytes in that buffer)
 *      x_linewid (number of columns taken up by that text)
 */
char *expand( row, col )
 int row, col;
{       register char *l;
        register char *s;
        static char mybuf[256];

        /* make sure the row number is reasonable */
        if (row >= maxlines)
        {       x_buflen = 0;
                x_linewid = 0;
                return( " " );
        }

        /* make sure there is something to expand */
        l = LINEBUF( row );
        x_linewid = valid( l, maxcols );
        if (x_linewid <= col)
        {       x_buflen = 0;
                x_linewid = 0;
                return( " " );
        } 

        /* advance our pointers to the desired text */
        x_linewid -= col;
        l += col;
        
#ifdef  COLOR_CODE
        /* find our shift buffer, and get rid of it if we possibly can */
        s = LINESHIFTS( row );
        if (s && shiftfree( row ))
                s = 0;

        if (s)
        {       /* we have to expand attributes */
                int i;
                char atr = '0';
                x_buflen = 0;

                s += col;       /* move up to the attr for the first char */

                /* copy each character into buffer, checking attributes */
                for( i = 0; i < x_linewid; i++ )
                {       if (atr != s[i])
                        {       atr = s[i];
                                mybuf[ x_buflen++ ] = COLOR_CODE;
                                mybuf[ x_buflen++ ] = atr;
                        }

                        mybuf[ x_buflen++ ] = l[i];
                }

                /* make sure it ends in the normal attribute */
                if (atr != '0')
                {       mybuf[ x_buflen++ ] = COLOR_CODE;
                        mybuf[ x_buflen++ ] = '0';
                }

                return( mybuf );
        } else
#endif
        {       /* just return him text in the line */
                x_buflen = x_linewid;
                return( &l[col] );
        }
}

/*
 * routine:
 *      deexpand
 *
 * purpose:
 *      to take expanded text and put it into a line buf, handling attributes
 *
 * parms:
 *      buf     pointer to input line
 *      length of input buffer
 *      line#   number of the line buffer it goes into
 *
 * returns:
 *      pointer to buffer containing line text
 */
char *dexpand( buf, len, row )
 char *buf;
 int len;
 int row;
{       register int outcol = 0;
        register int incol = 0;
        char atr = '0';
        char *s, *t;
        
        t = LINEBUF( row );
        s = 0;

        while( outcol < maxcols  &&  incol < len )
        {       
#ifdef  COLOR_CODE
                if (buf[incol] == COLOR_CODE)
                {       atr = buf[++incol];
                        incol++;
                        continue;
                } 

                if (atr != '0')
                {       if (s == 0)
                                s = shiftalloc( row );
                        if (s)
                                s[outcol] = atr;
                }
#endif
                t[outcol++] = buf[incol++];
        }

        return( t );
}

/*
 * routine:
 *      quit
 *
 * purpose:
 *      to exit the editor without saving the file
 */
void quit()     /* abort file and exit */
{       char inbuf[20];

        /* initialize buffer with default value */
        strcpy( inbuf, "YES" );

        /* see if he really wants to go through with it */
        if (solicit( "Abort edit? ", inbuf, sizeof inbuf ))
        {       if (inbuf[0] != 'y' && inbuf[0] != 'Y')
                        return;
        }

        tio_clup();
        (void) outstr( "Aborting edit - not saving file\r\n" );
        longjmp(exited,1);
}

/*
 * routine:
 *      solicit
 *
 * purpose:
 *      to prompt for and read a line of input
 *
 * parms:
 *      string to prompt with
 *      address of buffer to read answer into
 *      length of buffer
 *
 * returns:
 *      length of returned string
 */
int solicit( prompt, buf, len )
 char *prompt, *buf;
 int len;
{       int col, count;
        char c;

        /* we do this on the last line of the screen */
        abs_cur( maxrows - 1, 0 );
        clrline();

        /* give him the prompt */
        col = outstr( prompt );

        /* if the buffer doesn't start with a null, that is default value */
        if (buf[0] != 0)
        {       outstr( buf );
                abs_cur( maxrows - 1, col );
        }

        /* read input until we have fulfilled length or hit a newline */
        for( count = 0; count < len - 1; )
        {       c = readkbchar();

                /* end of line terminates the input */
                if (c == '\r' || c == '\n')
                        break;

                /* only editing character is ^H backspace */
                if (c == 010 && count > 0)
                {       count--;
                        col--;
                        abs_cur( maxrows - 1, col );
                        outchr( ' ' );
                        abs_cur( maxrows - 1, col );
                        continue;
                }

                /* no other control characters allowed */
                if (c < ' ')
                {       bell();
                        continue;
                }
                
                /* its another character for the string */
                buf[count++] = c;
                outchr( c );
                col++;
        }

        /* if we got anything, null terminate it */
        if (count > 0)
                buf[count] = 0;

        /* fix the line on the screen */
        abs_cur( maxrows - 1, 0 );
        update( firstline + maxrows - 1, 0, 0 );
        cursor();
        
        /* tell him what he got */
        return( count );
}

/*
 * routine:
 *      search
 *
 * purpose:
 *      to search for a string within a line
 *
 * parms:
 *      row, col to start search
 *      string to search for
 *
 * returns:
 *      col at which pattern starts
 *      -1 -> pattern does not appear in line
 */
int search( row, col, str )
 int row, col;
 char *str;
{       register char *t;
        register int i, j;

        t = LINEBUF( row );
        for( i = col; i < maxcols; i++ )
        {       /* check for first character match */
                if (t[i] != str[0])
                        continue;

                /* check for match on the rest of the string */
                for( j = 0; str[j] && t[i+j] == str[j]; j++ );
                if (str[j] == 0)
                        return( i );
        }

        /* no joy */
        return( -1 );
}

/*
 * routine:
 *      valid
 *
 * purpose:
 *      to strip trailing blanks off a line and see what's left
 *      so we can minimize the amount of data we send to the screen
 *
 * parms:
 *      pointer to line
 *      length of line
 *
 * returns:
 *      number of characters up to  last non-blank
 */
 int valid( s, len )
  register char *s;
  register int len;
{
        while( len > 0 && s[len - 1] == ' ' )
                len--;

        return( len );
}
