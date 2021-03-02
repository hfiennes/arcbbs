r/*
 * module:
 *      ed_out.c        version 1.3
 *
 * purpose:
 *      this module contains routines to handle output to the screen
 *
 * external entry points:
 *      out_init        initialize the output control tables
 *
 *      abs_cur         absolute cursor positioning
 *      rel_cur         relative cursor motion
 *
 *      clrscrn         clear the screen
 *      clrline         clear the remainder of a line
 *
 *      insline         open a line at this point
 *      clsline         close a line at this point
 *      index           scroll entire screen up or down one
 *
 *      insertion       insert a character at this point
 *      clschar         close a character at this point
 *      inschar         insert a character at this point
 *
 *      term_init       initialize terminal for use by this module
 *      set_atr         set output attribute
 *
 * internal entry points:
 *      outseq          send an output sequence
 *
 * note:
 *      most of the output string definition stuff is fairly general and
 *      will probably work with most post-1975 terminals.  The O_C_POS
 *      row/column positioning is an exception.  We will only try to do
 *      row/column positioning for normal ansi.  Anything else must do the
 *      rows and columns separately.  This is because of the way outseq
 *      is written and because of tremendous possible variety in cursor
 *      positioning mechanisms.  I suspect that this won't end up causing
 *      too much trouble.
 */
#include "ed_rtns.h"
#include "ed_state.h"
#ifdef  CONSOLE
#include <dos.h>
#endif

extern char *outcodes[];        /* array of output control codes        */

/* indices into array of output control codes                   */
#define O_SETUP         0       /* initialize terminal for use  */
#define O_C_POS         1       /* position cursor to row/col   */
#define O_C_ROW         2       /* position cursor to row       */
#define O_C_COL         3       /* position cursor to col       */
#define O_C_UP          4       /* position cursor up           */
#define O_C_DOWN        5       /* position cursor down         */
#define O_C_RIGHT       6       /* position cursor right        */
#define O_C_LEFT        7       /* position cursor left         */
#define O_CLEAR_S       8       /* home and clear screen        */
#define O_CLEAR_L       9       /* clear to end of current line */
#define O_OPEN          10      /* open lines after current     */
#define O_CLOSE         11      /* close lines (at current)     */
#define O_INS_ON        12      /* insert mode on               */
#define O_INS_OFF       13      /* insert mode off              */
#define O_DELCH         14      /* delete chars and suck left   */
#define O_INSCH         15      /* clean up terminal after use  */
#define O_CLEANUP       16      /* clean up terminal after use  */
#define O_INDX_UP       17      /* scroll screen down one line  */
#define O_INDX_DN       18      /* scroll screen up one line    */
#define O_SCR_RGN       19      /* set scroll region            */
#define O_SGR           20      /* seg graphic rendition        */

#ifdef  CONSOLE
extern int      scrnpage;       /* what screen page we're on    */
char consatrs[10] =     /* console attribute map        */
        { 0x00, 0x0b, 0x0e, 0x0d, 0x1f, 0x02, 0x8c, 0x07, 0x0f, 0x87 };
#endif

char atrmap[10] =       /* normal ansi attribute map            */
        { 0, 36, 33, 35, 44, 32, 31, 37,  1,  5 };

#define COLOR_CODE 0x03                 /* WWIV isn't ansi here */

/* state and capability variables maintained in this module     */
int     scrnrow;                        /* actual row on screen */
int     scrncol;                        /* actual col on screen */
int     scrnins;                        /* physical insert mode */
int     scrnatr;                        /* current display atr  */
int     c_ins;                          /* has real insert mode */
int     c_del;                          /* has delch function   */
int     c_hrel;                         /* can relative horiz   */
int     c_vrel;                         /* can relative vert    */
int     c_ilines;                       /* can insert lines     */
int     c_dlines;                       /* can delete lines     */
int     c_uindex;                       /* can index up, scroll */
int     c_dindex;                       /* can index dn, scroll */
int     c_region;                       /* can set scroll region*/

/*
 * routine:
 *      out_init
 *
 * purpose:
 *      to look through the defined output sequences and decide what the
 *      capabilities of our terminal are
 * 
 * parms:
 *      none
 *
 * returns:
 *      0       - toto bene
 *      1       - trouble in river city
 */
int out_init()
{       register int i;

#ifdef  CONSOLE
        /* we know what the console can do, and it doesn't use codes */
        if (c_console)
        {       c_autowrap = 1;         /* cursor auto goes to next line */
                c_autoscroll = 0;       /* but we don't scroll on last col */
                c_base     = 0;

                c_ilines   = 1;         /* we can scroll to beat the band */
                c_dlines   = 1;
                c_uindex   = 1;
                c_dindex   = 1;
                c_region   = 0;         /* and don't need this */

                c_ins      = 0;         /* but we have no ins/delch */
                c_del      = 0;

                c_hrel     = 0;         /* no relative cursor motion */
                c_vrel     = 0;

                /* make sure we don't have any output codes */
                for( i = 0; outcodes[i]; i++ )
                        outcodes[i][0] = 0;
                
                return( 0 );
        }
#endif

        /* otherwise figure out what device can do, based on codes */
        if (outcodes[O_INDX_UP][0] || outcodes[O_OPEN][0])
                c_uindex = 1;

        if (outcodes[O_INDX_DN][0] || outcodes[O_CLOSE][0])
                c_dindex = 1;

        if (outcodes[O_SCR_RGN][0])
                c_region = 1;

        if (outcodes[O_OPEN][0] || (c_uindex && c_region))
                c_ilines = 1;

        if (outcodes[O_CLOSE][0] || (c_dindex && c_region))
                c_dlines = 1;

        if (outcodes[O_INSCH][0]  ||  
                (outcodes[O_INS_ON][0] && outcodes[O_INS_OFF][0]) )
                c_ins = 1;

        if (outcodes[O_DELCH][0])
                c_del = 1;

        if (outcodes[O_C_UP][0] && outcodes[O_C_DOWN][0])
                c_vrel = 1;

        if (outcodes[O_C_LEFT][0] && outcodes[O_C_RIGHT][0])
                c_hrel = 1;

        /* it is essential that we have a cursor positioning mechanism */
        if (outcodes[O_C_POS][0] == 0 && 
            (outcodes[O_C_ROW][0] == 0  ||  outcodes[O_C_ROW][0] == 0))
        {       bug( "no absolute cursor positioning" );
                return( 1 );
        }

        /* all looks good to me */
        return( 0 );
}

/*
 * routine:
 *      outseq
 *
 * purpose:
 *      to send an output sequence, plugging in numbers if necessary
 *
 * parms:
 *      sequence number to send (index into outcodes arry)
 *      iteration factor
 *      
 * returns:
 *      void
 *
 * algorithm:
 *      while the iteration factor is unfulfilled
 *          for each byte in the sequence
 *              if it is an unescaped #
 *                      send the interation factor
 *              else send the character
 *
 * note: a negative number means "do it once, no iteration factor required",
 *      whereas a value of one will be output with the iteration factor.  This
 *      is necessary because some commands reasonably default to one, while
 *      others do not have reasonable defaults - and we would like to minimize
 *      the number of characters we send.
 */
void outseq( seq, num )
 int seq, num;
{       register char *s;
        char c;

        /* make sure there is a sequence to send */
        if (outcodes[seq][0] == 0)
                return;

        /* do it enough times to satisfy the request */
        do {    /* go through the entire string */
                s = outcodes[seq];
                while( (c = *s++) != 0 )
                {       /* look for a # - means insert rep count here */
                        if (c == '#')
                        {       if (s[1] != '#')
                                {       if (num >= 0)
                                                (void) outnum( num );
                                        num = 0;        /* count is fulfilled */
                                        continue;
                                } else  /* double # means send out a # */
                                        s++;
                        } 
                        /* just output the character */
                        outchr( c );
                }

                /* we have handled at least one of the required reps */
                if (num > 0)
                        num--;
        } while( num > 0 );
}

/*
 * routine:
 *      abs_cur
 *
 * purpose:
 *      to position the cursor to a particular location
 *
 * parm:
 *      row (0 - maxrows-1)
 *      col (0 - maxcols-1)
 *
 * returns:
 *      void
 */
void abs_cur( row, col )
 int row, col;
{
#ifdef  CONSOLE
        if (c_console)
        {       union REGS regs;

                regs.h.ah = 0x02;       /* set cursor position */
                regs.h.dh = row;
                regs.h.dl = col;
                regs.h.bh = scrnpage;
                int86( 0x10, &regs, &regs );

                scrnrow = row;
                scrncol = col;
                return;
        }
#endif

        /* note the new cursor position */
        scrnrow = row; scrncol = col;

        /* some terminals base at zero, ansi bases at one */
        row += c_base;
        col += c_base;

        if (outcodes[O_C_POS][0])
        {       /* assume standard ansi curor positioning */
                outchr( '\033' );
                outchr( '[' );
                if (row > 1)
                        (void) outnum( row );
                if (col > 1)
                {       outchr( ';' );
                        (void) outnum( col );
                }
                outchr( 'H' );
        } else
        {       /* anything else must have separate row/col positioning */
                outseq( O_C_ROW, row );
                outseq( O_C_COL, col );
        }

}

/*
 * routine:
 *      clrline
 *
 * purpose:
 *      to clear the remainder of the current line
 *
 * parm:
 *      none
 *
 * returns:
 *      void
 */
void clrline( )
{
#ifdef  CONSOLE
        if (c_console)
        {       union REGS regs;

                regs.h.ah = 0x09;       /* write character/atr */
                regs.h.al = ' ';
                regs.h.bh = scrnpage;
                regs.h.bl = consatrs[0];
                regs.x.cx = maxcols - scrncol;
                int86( 0x10, &regs, &regs );

                abs_cur( scrnrow, scrncol );
                return;
        }
#endif

        if (outcodes[O_CLEAR_L][0])
                outseq( O_CLEAR_L, -1 );
        else
        {       /* if there is no clear line, fill it with blanks */
                int save = scrncol;
                int limit = maxcols;
                register int i;

                /* writing the last byte on the last row is dangerous */
                if (c_autoscroll  &&  scrnrow == (maxrows - 1))
                        limit--;

                for( i = scrncol; i < limit; i++ )
                        outchr( ' ' );
                
                abs_cur( scrnrow, save );
        }
}

/*
 * routine:
 *      clrscrn
 *
 * purpose:
 *      to clear the terminal screen
 *
 * parm:
 *      none
 *
 * returns:
 *      void
 */
void clrscrn( )
{
#ifdef  CONSOLE
        if (c_console)
        {       union REGS regs;

                abs_cur( 0, 0 );        /* start from home      */
                set_atr( '0' );         /* default attribute    */

                regs.h.ah = 0x09;       /* write character/atr  */
                regs.h.al = ' ';
                regs.h.bl = consatrs[0];
                regs.h.bh = scrnpage;
                regs.x.cx = maxrows * maxcols;
                int86( 0x10, &regs, &regs );

                abs_cur( 0, 0 );
                return;
        }
#endif 

        if (outcodes[O_CLEAR_S][0])
        {       outseq( O_CLEAR_S, -1 );
                scrnrow = 0; scrncol = 0;
        } else
        {       /* if there is no clear screen, clear each of the lines */
                register int i;

                for( i = 0; i < maxrows; i++ )
                {       abs_cur( i, 0 );
                        clrline();
                }
                abs_cur( 0, 0 );
        }
}

/*
 * routine:
 *      rel_cur
 *
 * purpose:
 *      to perform local cursor motion
 *
 * parm:
 *      delta row (- up, + down )
 *      delta col (- left, + right )
 *
 * returns:
 *      void
 */
void rel_cur( row, col )
 register int row, col;
{
        if ((row && !c_hrel) || (col && !c_vrel))
                abs_cur( scrnrow + row, scrncol + col );
        else
        {       if (row > 0)
                        outseq( O_C_DOWN, row );
                else if (row < 0)
                        outseq( O_C_UP, -row );

                if (col > 0)
                        outseq( O_C_RIGHT, col );
                else if (col < 0)
                        outseq( O_C_LEFT, -col );

                scrnrow += row;
                scrncol += col;
        }
}


/*
 * routine:
 *      insline
 *
 * purpose:
 *      to insert a line at the current point
 *
 * parm:
 *      number of lines to open
 *
 * returns:
 *      void
 */
void insline( numline )
{
#ifdef  CONSOLE
        if (c_console)
        {       union REGS regs;

                regs.h.ah = 7;          /* scroll down */
                regs.h.al = numline;
                regs.h.ch = scrnrow;    /* starting from the current row    */
                regs.h.cl = 0;
                regs.h.dh = maxrows-1;  /* going to the last row        */
                regs.h.dl = maxcols-1;
                regs.h.bh = scrnatr;
                int86( 0x10, &regs, &regs );

                return;
        }
#endif

        if (outcodes[O_OPEN][0])
                outseq( O_OPEN, numline );
        else if (c_region && c_uindex)
        {       /* fake line insert with set scroll region */
                outseq( O_SCR_RGN, scrnrow + c_base );
                abs_cur( scrnrow, 0 );
                outseq( O_INDX_UP, numline );
                outseq( O_SCR_RGN, c_base );
                abs_cur( scrnrow, 0 );
        }
}

/*
 * routine:
 *      clsline
 *
 * purpose:
 *      to close the current line
 *
 * parm:
 *      number of lines to close
 *
 * returns:
 *      void
 */
void clsline( numline )
{
#ifdef  CONSOLE
        if (c_console)
        {       union REGS regs;

                regs.h.ah = 6;          /* scroll up */
                regs.h.al = numline;
                regs.h.ch = scrnrow;    /* starting from the current row    */
                regs.h.cl = 0;
                regs.h.dh = maxrows-1;  /* down to the end of the screen */
                regs.h.dl = maxcols-1;
                regs.h.bh = scrnatr;
                int86( 0x10, &regs, &regs );

                return;
        }
#endif

        if (outcodes[O_CLOSE][0])
                outseq( O_CLOSE, numline );
        else if (c_region && c_uindex)
        {       /* fake line insert with set scroll region */
                outseq( O_SCR_RGN, scrnrow + c_base );
                abs_cur( maxrows - 1, 0 );
                outseq( O_INDX_DN, numline );
                outseq( O_SCR_RGN, c_base );
                abs_cur( scrnrow, 0 );
        }
}

/*
 * routine:
 *      index
 *
 * purpose:
 *      to scroll the entire screen up or down multiple lines
 *
 * parm:
 *      number of lines 
 *              positive means index downwards, scroll upwards
 *              negative means index upwards, scroll downwards
 */
void index( numline )
 int numline;
{
        if (numline < 0)
        {       /* scroll screen downwards */
                numline = -numline ;
                abs_cur( 0, 0 );
                if (outcodes[O_INDX_UP][0])
                        outseq( O_INDX_UP, numline );
                else if (c_ilines)
                        insline( numline );
        } else
        {       /* scroll screen upwards */
                if (outcodes[O_INDX_DN][0])
                {       abs_cur( maxrows - 1, 0 );
                        outseq( O_INDX_DN, numline );
                } else if (c_dlines)
                {       abs_cur( 0, 0 );
                        clsline( numline );
                }
        }
}

/*
 * routine:
 *      insertion
 *
 * purpose:
 *      to turn insert mode on or off on terminal
 *
 * parm:
 *      TRUE    turn insert mode on
 *      FALSE   turn insert mode off
 *
 * returns:
 *      void
 */
void insertion( m )
 int m;
{
        /* for some terminals we fake this on a per-character basis     */
        if (c_ins && outcodes[O_INS_ON][0])
        {       outseq( m ? O_INS_ON : O_INS_OFF, -1 );
                scrnins = m;
        }
}

/*
 * routine:
 *      clschar
 *
 * purpose:
 *      to close a character at the current column
 *
 * parm:
 *      number of characters to suck up
 *
 * returns:
 *      void
 */
void clschar( numchr )
{
        /* for the console, we rewrite the line when necessary */
        if (c_del)
                outseq( O_DELCH, numchr );
}

/*
 * routine:
 *      inschar
 *
 * purpose:
 *      to close a character at the current column
 *
 * parm:
 *      character to insert
 *
 * returns:
 *      void
 */
void inschar( c )
{
        /* if we aren't in insert mode, maybe we can force an insert char */    
        if (scrnins == 0 && outcodes[O_INSCH][0])
                outseq( O_INSCH, -1 );

        outchr( c );
}

/*
 * routine:
 *      set_atr
 *
 * purpose:
 *      to set the current display attribute
 *
 * parm:
 *      character indicating attribute to be set
 */
void set_atr( c )
 int c;
{       int code;

#ifdef  CONSOLE
        /* fist time we're called, note default attribute we came in with */
        if (consatrs[0] == 0x00)
                consatrs[0] = scrnatr;

        /* console has an array to map attribute numbers into attribute bytes */
        if (c_console)
        {       if (c >= '0' && c <= '9')
                        scrnatr = consatrs[ c - '0' ];
        } else 
#endif

        /* terminals - we send the number out to and let them handle it */
        if (scrnatr != c)
        {       scrnatr = c;
                if (c_ansiatr)
                {       code = atrmap[ c - '0' ];
                        outseq( O_SGR, code );
                } else  /* wwiv has its own special way of doing this   */
                {       outchr( COLOR_CODE );
                        outchr( scrnatr );
                }
        }
}

/*
 * routine:
 *      term_init
 *
 * purpose:
 *      to initialize (or deinitialize) terminal for use by editor
 *
 * parms:
 *      0 - set up for use by editor
 *      1 - return to normal terminal use
 */
void term_init( done )
 int done;
{
        insertion( 0 );

        if (done)
        {       abs_cur( maxrows-1, 0 );
                clrline();
                outseq( O_CLEANUP, -1 );
        } else 
                outseq( O_SETUP, -1 );
}
