/*
 * module:
 *      ed_fio.c        version 1.0
 *
 * purpose:
 *      this module contains system specific routines to handle file I/O
 *
 * external entry points:
 *      openfile        open a new file for use
 *      closefile       stop using a file
 *      getline         read the next line from a file
 *      putline         write the next line to a file
 *
 * note:
 *      for simple-mindedness, these routines aren't guaranteed to work
 *      for multiple files open at a time
 */
#include "ed_rtns.h"
#include <stdio.h>

extern  int     tabwidth;       /* tab stops for this file              */

#define INBUFLEN        512     /* size of internal input/output buffer */
char    mybuf[INBUFLEN];        /* buffer where we use for input/output */
char    *inptr;                 /* pointer to start of unread data      */
int     incount;                /* number of bytes of unread data       */

/*
 * routine:
 *      openfile
 *
 * purpose:
 *      to open a file for input or output
 *
 * parms:
 *      name of file to open
 *      desired access:  R/W
 *
 * returns:
 *      a file descriptor token
 */
int openfile( name, acc )
 char *name;
 char acc;
{       int retval;

        /* make sure we aren't opening a second file */
        if (incount > 0)
        {       bug("Two files open");
                return( 0 );
        }

        /* open or recreate the file, as appropriate to the access */
        if (acc == 'R')
                retval = (int)fopen( name, "r" );
        else
                retval = (int)fopen( name, "w" );

        return( retval );
}

/*
 * routine:
 *      closefile
 *
 * purpose:
 *      to close a file that was previously opened with openfile
 *
 * parms:
 *      file descriptor token
 *
 * returns:
 *      void
 */
void closefile( fdesc )
 int fdesc;
{       
        /* note that we're through with the old file */
        incount = 0;

        /* just close the file */
        (void) fclose((FILE*)fdesc);
}

/*
 * routine:
 *      getline
 *
 * purpose:
 *      to read the next line from an open file
 *
 * parms:
 *      file descriptor token
 *      address of buffer
 *      length of buffer
 *
 * returns:
 *      number of characters placed in buffer
 *
 * note:
 *      any line termination has been removed and all tabs have been expanded
 */
int getline( fdesc, buf, len )
 int fdesc;
 char *buf;
 int len;
{       register char *s = buf;
        int col = 0;

        /* loop til we satisfy the requested length */
        while( col < len )  
        {
        if (incount <= 0)
                {       /* the input buffer needs refilling */
                        incount = fread( mybuf, 1, sizeof mybuf,(FILE*)fdesc );
                        if (incount <= 0)
                                return( 0 );
                        inptr = mybuf;
                }

                /* we are taking one more character     */
                incount--;

                /* check for ascii format effectors */
                if (*inptr == '\n')             /* newline means we're done */
                {       inptr++;
                        break;
                } else if (*inptr == '\r')      /* carriage return is ignored */
                {       inptr++;
                        continue;
                } else if (*inptr == '\t')      /* tabs are expanded */
                {       do
                        {       *s++ = ' ';
                                col++;
                        } while ( (col < len) && (col % tabwidth) );
                        inptr++;
                } else                          /* everything else is copied */
                {       *s++ = *inptr++;
                        col++;
                }
        }

        if (s == buf)
                *s++ = ' ';
                
        return( s - buf );      /* tell him what he got */
}

/*
 * routine:
 *      putline
 *
 * purpose:
 *      to write the next line to an open file
 *
 * parms:
 *      file descriptor token
 *      address of buffer
 *      length of buffer
 *
 * returns:
 *      number of characters placed in buffer
 *
 * note:
 *      any necessary line termination will be added, and trailing blanks
 *      will be stripped off.
 */
int putline( fdesc, buf, len )
 int fdesc;
 register char *buf;
 int len;
{       
        register int i;

        /* in order to add the new line, we must copy the buffer */
        for( i = 0; i < len; i++ )
                mybuf[i] = buf[i];

        /* find the last non-blank character on the line */
        while( i > 0 && buf[i-1] == ' ' )
                i--;

        /* append a newline - (in text mode, DOS handles cariage returns) */
        buf[i++] = '\n';
        
        /* and write it out to the file */
        return( fwrite( buf, i,1,(FILE*)fdesc ) );
}
