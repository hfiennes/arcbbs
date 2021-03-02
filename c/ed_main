/*
 * (C) Copyright 1988, Software by Sagredo, "est Omnibus"
 *
 *      this product may be freely used, duplicated and distributed
 *      provided that no charge is made for this code, and that this
 *      copyright notice is preserved in all copies and derivative works.
 *
 * module:
 *      ed_main.c       version 1.2
 *
 * purpose:
 *      this module contains the main routine and parameter processing
 *
 * external entry points:
 *      main            main entry point
 * internal entry points:
 *      readcfg         read in a configuration file
 *      option          process an option
 *      atoi            convert an ascii number into an integer
 */

#include "ed_rtns.h"
#include "ed_defs.h"
#include "setjmp.h"
jmp_buf exited;

/* session control parameters */
int     maxrows  = MAX_ROW;     /* number of rows on screen     */
int     maxcols  = MAX_COL;     /* number of cols on screen     */
int     maxlines = MAX_LINES;   /* number of lines in file      */
int     tabwidth = TAB_WIDTH;   /* width of a tab stop          */
int     wrapmgn  = WRAP_MGN;    /* wordwrap margin              */
int     o_autoindent = 1;       /* should word-wrap auto-indent */

/* terminal capability parameters */
int     c_ansiatr    = 0;       /* use ansi graphic rendition   */
int     c_autowrap   = 0;       /* does cursor wrap to next col */
int     c_autoscroll = 1;       /* does nl at eos cause scroll  */
int     c_console    = 0;       /* we are using the PC console  */
int     c_base       = 1;       /* row/column base 0/1          */

/*
 * routine:
 *      main
 *
 * purpose:
 *      program initialization
 *
 * parms:
 *      number of arguments
 *      character strings for the args
 *
 * algorithm:
 *      look over the arguments
 *      initialize all the sub-systems
 *      main-loop: read characters and pass them to the input lexer
 */
extern int portnumber;

int screen_edit()
{       int i;
        int errs = 0;
        char *userfile = 0,file[128];
                                                
        readcfg("<ARCbbs$miscdata>.FSE_In");
        readcfg("<ARCbbs$miscdata>.FSE_Out");
        sprintf(file,"<ARCbbs$temp>.Edit_%d",portnumber);
        remove(file);
        userfile=file;

        /* initialize the output screen handling */
        if (tio_init()) return;

        /* validate the basic configuration information */
        if (cfg_init()) return;

        /* initialize the input lexer */
        if (inp_init()) return;

        /* initilaize output processing */
        if (out_init()) return;

        /* initialize the editing arena */
        if (ed_init( userfile )) return;
                      
        if ((i=setjmp(exited))<1)
          {
          /* pass characters to the editor until we hang up or exit */
          while( (i = readkbchar()) >= 0 )
                  inp_char( i );
          i=1;
          }

        /* we must have hung up */
        tio_clup();
        return(i-1);
}

/*
 * routine:
 *      readcfg
 *
 * purpose:
 *      to process a file of configuration parameters
 *
 * parms:
 *      name of configuration file
 *
 * returns:
 *      number of errors
 */
int readcfg( name )
 char *name;
{       int fdesc, len; 
        int lineno = 0;
        int errs = 0;
        char mybuf[MAX_COL];

        if (fdesc = openfile( name, 'R' ))
        {       while( len = getline( fdesc, mybuf, sizeof mybuf ) )
                {       mybuf[len] = 0;
                        lineno++;
                        if (option( mybuf ))
                        {       (void) outstr( "line " );
                                (void) outnum( lineno );
                                (void) outstr( ": " );
                                outline( mybuf, len );
                                (void) outstr( "\r\n" );
                                errs++;
                        }
                }
                closefile( fdesc );
        } else
                bug( "Unable to open option file" );

        return( errs );
}

/*
 * routine:
 *      option
 *
 * purpose:
 *      to parse and process a command line or config file option spec
 *
 * parms:
 *      pointer to null terminated option string
 *
 * returns:
 *      0 - toto bene
 *      1 - error in option
 *
 * note:
 *      this is a little bit strange, but we process options for both
 *      the command line and the configuration file with the same code.
 *      It saves some code, gives us the same syntax for both, and ensures
 *      that anything that can be specified in one place can also be 
 *      specified in the other.
 */
option( opt )
 char *opt;
{       int code, type;
        register char *s = opt;

        /* see what sort of option this is */
        switch( *s )
        { case '#': case '*':  /* comments lines are ignored    */
          case ' ': case 0:     /* blank lines are ignored      */
                break;

          case 'h':     /* screen height        */
                maxrows = atoi( &s[1] );
                break;

          case 'w':     /* screen width         */
                maxcols = atoi( &s[1] );
                break;

          case 'l':     /* maximum file length  */
                maxlines = atoi( &s[1] );
                break;

          case 't':     /* width of a tab       */
                tabwidth = atoi( &s[1] );
                break;

          case 'm':     /* wrap margin          */
                wrapmgn = atoi( &s[1] );
                break;
           
          case 'd':     /* start in dual model command mode */
                c_cmd();
                break;

          case 'r':     /* start in auto-replace mode */
                c_autor();
                break;

          case 'i':     /* start in auto-insert mode */
                c_autoi();
                break;

          case 'a':     /* misc terminal features */
                if (s[1] == 's')
                        c_autoscroll = (s[2] == '0') ? 0 : 1;
                else if (s[1] == 'w')
                        c_autowrap = (s[2] == '0') ? 0 : 1;
                else if (s[1] == 'i')
                        o_autoindent = (s[2] == '0') ? 0 : 1;
                else if (s[1] == 'a')
                        c_ansiatr = (s[2] == '0') ? 0 : 1;
                break;

          case 'b':     /* row/column base      */
                c_base = (s[1] - '0') ? 1 : 0;
                break;

          case 'c':     /* use PC console screen update functions */
                c_console = 1;
                maxrows = 25;
                break;

          case 'C':     /* define cmd sequence: C # "seq" */
          case 'S':     /* define esc sequence  S # "seq" */
                type = *s;
                code = atoi( &s[1] );

                /* eat white space up to the sequence */
                for( s += 2; (*s == ' ') || (*s >= '0' && *s <= '9'); s++ );

                if (*s)
                {       if ( in_def( type == 'S', code, s ) )
                                return( 1 );
                } else
                        return(1);
                break;

          case 'O':     /* define output control sequence: O # "seq" */
                code = atoi( &s[1] );

                /* eat white space up to the sequence */
                for( s += 2; (*s == ' ') || (*s >= '0' && *s <= '9'); s++ );

                if (*s)
                {       if ( out_def( code, s ) )
                                return( 1 );
                } else
                        return(1);
                break;
        }

        return( 0 );
}
