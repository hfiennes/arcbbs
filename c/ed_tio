i/*
 * module:
 *      ed_tio.c        version 1.1
 *
 * purpose:
 *      this is a machine-dependent module to handle keyboard input and 
 *      console output for the editor
 *
 * external entry points:
 *      tio_init        initialize the output stuff
 *      tio_clup        clean up the output stuff
 *      readkbchr       read a character from the keyboard
 *      outline         output a line to the console
 *      outstr          output a string to the console
 *      outnum          output a decimal integer
 *      outchr          output a character to the console
 *      bell            ring a bell to warn of an error
 *      bug             log an error message
 *
 * internal entry points:
 *      cons_putc       tty-like output routine for PC screen
 *      kbread          input routine for PC keyboard
 *
 * note:
 *      dending on how much you worry about memory, you can compile the
 *      support for the PC console in or out with the pre-processor symbol
 *      CONSOLE.  If this is not defined, the code to support the console
 *      will be left out of the editor.
 */

#include "ed_rtns.h"
#include "ed_state.h"
#include "port.h"

/*
 * routine:
 *      tio_init
 *
 * purpose:
 *      to perform any necessary terminal/screen initialization
 *
 * returns:
 *      number of errors
 */
tio_init()
 {      int errs = 0;

        return( errs );
 }

 /*
  * routine:
  *     tio_clup
  *
  * purpose:
  *     to perform any necessary terminal/screen cleanup
  */
void tio_clup()
{       
        /* restore attribute to normal */
        set_atr( '0' );
        curatr = 0;

        /* clean up the screen */
        term_init( 1 );

}


/*
 * routine:
 *      readkbchar
 *
 * purpose:
 *      to read a character from the keyboard
 *
 * parms:
 *      none
 *
 * returns:
 *      > 0 byte read
 *      -1  hangup has taken place
 *
 * algorithm:
 *      read and buffer up a few characters, passing them back to the
 *      caller one at a time.
 */
int readkbchar()
{             
while(port_rxbuffer()==0) window_poll();
return(port_rx());
}

/*
 * routine:
 *      outline
 *
 * purpose:
 *      to send a line to the terminal
 *
 * parms:
 *      pointer to line to output
 *      number of bytes to write
 *
 * note:
 *      it is assumed (for column counting purposes) that this routine
 *      is only used to output text characters, and not escape sequences.
 */
void outline( buf, len )
 char *buf;
 int len;
{       register char *s;

        /* in either case, we must figure out what we've done */
        for( s = buf; s < &buf[len]; s++ )
        {       port_txw(*s);
                if (*s == '\r')
                        scrncol = 0;
                else if (*s == '\n')
                        scrnrow++;
                else if (*s >= ' ')
                {       if (++scrncol >= maxcols)
                        {       scrncol -= maxcols;
                        scrnrow++;
                        }       
                }
        }
}

/*
 * routine:
 *      outchr
 *
 * purpose:
 *      to send a character to the terminal
 *
 * parm:
 *      character to output
 *
 */
void outchr( c )
 char c;
{                
        port_txw(c);
}

/*
 * routine:
 *      outstr
 *
 * purpose:
 *      to send a null-terminated string to the terminal
 *
 * parm:
 *      pointer to string to send
 *
 * returns:
 *      number of bytes printed
 *
 * note:
 *      this routine assumes that outline will handle the cursor position
 *      implications of this output.
 */
int outstr( s )
 register char *s;
{       register int cnt = strlen(s);

        outline( s, cnt );
        return( cnt );
}

/*
 * routine:
 *      outnum
 *
 * purpose:
 *      to print out a decimal integer
 *
 * parm:
 *      number to print
 *
 * returns:
 *      number of characters printed
 */
int outnum( n )
 register int n;
{       int chars = 0;
        int divisor;

        /* check for zero */
        if (n == 0)
        {       outchr( '0' );
                return( 1 );
        }

        /* check for negative numbers */
        if (n < 0)
        {       outchr( '-' );
                chars++;
                n = -n;
        }

        /* limit the size of the numbers */
        if (n > 10000)
                n %= 10000;

        /* find the order of the first digit */
        for( divisor = 10000; divisor > n; divisor /= 10 );

        do {    outchr( '0' + (n / divisor) );
                chars++;
                n %= divisor;
                divisor /= 10;
        } while( divisor >= 1 );

        return( chars );
}

/*
 * routine:
 *      bell
 *
 * purpose:
 *      to ring a bell at the terminal and warn of an error
 */
void bell()
{
                outchr( 007 );
}

/*
 * routine:
 *      bug
 *
 * purpose:
 *      log a message complaining about an internal error
 *
 * parm:
 *      string describing the problem
 */
void bug( s )
 char *s;
{
        bell();
        (void) outstr( "\nERROR: " );
        (void) outstr( s );
        (void) outstr( "\r\n" ); while(port_rxbuffer()==0) window_poll();
}