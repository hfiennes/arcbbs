/*
 * module:
 *      ed_config.c     version 1.5
 *
 * purpose:
 *      this module contains the command and special sequence tables
 *
 * external entry points:
 *      cfg_init        initialize the input lexer tables
 *      in_def          define an input command sequence
 *      out_def         define an output command sequence
 *      help            display a help screen
 *
 * internal entry points
 *      showctl         show a string of control codes
 *      readctl         turn ascii sequence into control codes
 *
 * principal data structures:
 *      ed_cmds         array describing command sequences
 *      commands        array mapping command codes to routines
 *      outcodes        array mapping functions to output sequences
 *
 * notes:
 *      to add a new editing action:
 *          add the function to appropriate module
 *          add function declaration to ed_rtns.h with type VOID
 *          add call to function to commands array
 *          add descriptive string to cmdnames
 *          add default value to ed_cmds array
 *
 *      to add a new output action:
 *          add a new definition to front ed_out.c for its outcodes index
 *          add default string to outbufs (at that index)
 *          add new entry to outcodes (if necessary)
 *          add code that uses the new string to ed_out.c
 */

#define VOID    int     /* this is totally bogus - but so is PCC        */

#include "ed_defs.h"
#include "ed_rtns.h"

/* 
 * names of the commands - used by help to associate meanings with command
 *      numbers.  Consequently indices into this array must agree with the
 *      corresponding indices into the "commands" array of functions.
 *
 * an entry of zero will not print out in the help display
 */
char *cmdnames[MAX_ICODE] = 
{       0,              "cursor left",  "cursor right", "cursor up",
        "cursor down",  "cursor home",  "cursor end",   "newline",
        "cariage ret",  "tab forward",  "backtab",      "back word",
        "fwd word",     "page up",      "page down",    "insmode",
        "del char",     "ins line",     "del line",     "del nxt word",
        "del prv word", "break line",   "join line",    "auto-insrt",
        "auto-rplc",    "cmd mode",     "abort",        "save",         
        "refresh",      "help info",    "color change", "spcl char",
        "restore",      "clr to eol",   "end of line",  "backspace",
        "find",         "replace",      "global rplc",  "refill"
};

/* 
 * This array actually associates editing functions with input code numbers.
 * If you change the order of any of these entries it will break existing
 * input configuration tables.
 */
extern VOID help();
int (*commands[MAX_ICODE])() =
{       bell,           /* 0 - always illegal   */
        csr_left,       /* 1 - cursor left one  */
        csr_right,      /* 2 - cursor right one */
        csr_up,         /* 3 - cursor up one    */
        csr_down,       /* 4 - cursor down one  */
        csr_home,       /* 5 - cursor top left  */
        csr_end,        /* 6 - cursor bot left  */
        csr_nl,         /* 7 - cursor newline   */
        csr_cr,         /* 8 - cursor return    */
        csr_tab,        /* 9 - cursor tab right */
        csr_btab,       /* 10 - cursor tab left */
        csr_bword,      /* 11 - cursor back word*/
        csr_fword,      /* 12 - cursor fwd word */
        csr_pgup,       /* 13 - cursor back page*/
        csr_pgdn,       /* 14 - cursor fwd page */

        c_insmd,        /* 15 - insert mode     */
        c_delch,        /* 16 - delete char     */
        c_insln,        /* 17 - insert line     */
        c_delln,        /* 18 - delete line     */
        c_delwd,        /* 19 - delete word fwd */
        c_delwdb,       /* 20 - delete word bak */
        c_brkln,        /* 21 - break line      */
        c_joinln,       /* 22 - join lines      */

        c_autoi,        /* 23 - auto-insert     */
        c_autor,        /* 24 - auto-replace    */
        c_cmd,          /* 25 - command mode    */

        quit,           /* 26 - abort edit      */
        save,           /* 27 - exit and save   */
        repaint,        /* 28 - repaint screen  */
        help,           /* 29 - help text       */

/* these are not properly commands, bur rather escape sequences */
        color,          /* 30 - change color    */
        control,        /* 31 - octal escape    */

/* and we keep on adding them */
        c_undel,        /* 32 - undelete        */
        c_clrln,        /* 33 - clr to eol      */
        csr_eol,        /* 34 - cursor to eol   */
        c_bspc,         /* 35 - backspace       */
        c_find,         /* 36 - find            */
        c_rpl,          /* 37 - replace         */
        c_rplall,       /* 38 - global replace  */
        c_refill,       /* 39 - refill          */
};

/*
 * these are the default command sequences - although they can be overriden
 *      by a parameter file.  These sequences are reasonable for the DOS
 *      console and make good use of the function keys and cursor pad.  Dialin
 *      users won't be able to use it - but they will probably have lots of
 *      opinions about what commands they want - and this is the richest set
 *      and works very nicely for the sysop
 */
struct ed_cmd ed_cmds[MAX_CMDS + 1] =
{       /* normal commands - entire sequence required                   */
        1,      0,      "\033[D",       /* cursor left      LEFT        */
        2,      0,      "\033[C",       /* cursor right     RIGHT       */
        3,      0,      "\033[A",       /* cursor up        UP          */
        4,      0,      "\033[B",       /* cursor down      DOWN        */
        5,      0,      "\033[G",       /* cursor firstln   CTL-HOME    */
        6,      0,      "\033[k",       /* cursor lastln    CTL-END     */
        7,      0,      "\n",           /* new line                     */
        7,      0,      "\r",           /* cariage return line nl       */
        8,      0,      "\033[H",       /* left margin      HOME        */
        9,      0,      "\t",           /* tab right        TAB         */
        10,     0,      "\033[Z",       /* tab left         SHIFT TAB   */
        11,     0,      "\033[d",       /* cursor back word CTL-LEFT    */
        12,     0,      "\033[c",       /* cursor forward word CTL-RIGHT*/
        13,     0,      "\033[U",       /* page back        PAGE-UP     */
        14,     0,      "\033[V",       /* page fwd         PAGE-DN     */
        15,     0,      "\033[@",       /* insert mode      INS         */
        16,     0,      "\177",         /* delete character DEL         */
        17,     0,      "\033[1",       /* insert line      F1          */
        18,     0,      "\033[2",       /* delete line      F2          */
        19,     0,      "\004",         /* delete word right ^D         */
        20,     0,      "\027",         /* delete word left ^W          */
        21,     0,      "\033[3",       /* break lines      F3          */
        22,     0,      "\033[4",       /* join lines       F4          */
        26,     0,      "\013\021",     /* abort edit       ^K^Q        */
        27,     0,      "\013\004",     /* exit and save    ^K^D        */
        28,     0,      "\002",         /* repaint screen   ^B          */
        29,     0,      "\017",         /* help             ^O          */
        30,     2,      "\020#",        /* color shift      ^P #        */
        30,     2,      "\021#",        /* color shift      ^Q #        */
        31,     3,      "\\XX",         /* special code     \^# or \##  */
        32,     0,      "\033[5",       /* undo             F5          */
        33,     0,      "\033[6",       /* clear to eol     F6          */
        34,     0,      "\033[K",       /* cursor to eol    END         */
        35,     0,      "\010",         /* backspace        backspace   */
        36,     0,      "\033[7",       /* find             F7          */
        37,     0,      "\033[9",       /* replace          F8          */
        38,     0,      "\033[0",       /* replace all      F10         */
        39,     0,      "\033[8",       /* refill           F8          */
};

/*
 * these codes determine how we make the terminal do output control functions
 *      the default sequences assume only what is supported by ansi.sys.  This
 *      is pretty close to the minimum necessary to support the editor.
 */
char outbufs[MAX_OCODE][MAX_OSEQ_LEN] =
{       
        "",                             /* terminal initialization      */
        "\033[#;#H",                    /* absolute cursor position     */
        "",                             /* cursor to row                */
        "",                             /* cursor to col                */
        "",                             /* cursor up                    */
        "",                             /* cursor down                  */
        "\033[#C",                      /* cursor right                 */
        "\033[#D",                      /* cursor left                  */
        "\033[H\033[2J",                /* clear screen                 */
        "\033[K",                       /* clear to end of line         */
        "",                             /* open line                    */
        "",                             /* close line                   */
        "",                             /* insert mode on               */
        "",                             /* insert mode off              */
        "",                             /* delch                        */
        "",                             /* insert one character         */
        "",                             /* terminal cleanup on exit     */
        "",                             /* index upwards and scroll     */
        "\012",                         /* index downwards and scroll   */
        "",                             /* scroll region                */
        "\033[#m",                      /* set graphic rendition        */
        "",                             /* UNUSED                       */
        "",                             /* UNUSED                       */
        "",                             /* UNUSED                       */
};

/* 
 * the indirection here is so that the external array can be an array
 * of pointers, which people can use without knowing the length of the
 * buffers that are pointed to.
 */
char *outcodes[] =
{       outbufs[0],     outbufs[1],     outbufs[2],     outbufs[3],
        outbufs[4],     outbufs[5],     outbufs[6],     outbufs[7],
        outbufs[8],     outbufs[9],     outbufs[10],    outbufs[11],
        outbufs[12],    outbufs[13],    outbufs[14],    outbufs[15],
        outbufs[16],    outbufs[17],    outbufs[18],    outbufs[19],
        outbufs[20],    outbufs[21],    outbufs[22],    outbufs[23],
        0       /* last entry must be null */
};

/* 
 * these identification strings are used only in this module, but they are
 *       defined in ed_vers.c to simplify changing them and recompiling.
 */
extern char *version, *release, *copyright;

/*
 * routine:
 *      cfg_init
 *
 * purpose:
 *      to check the command configuration table for errors
 *      assign values to the various edit parameters
 *
 * parms:
 *      none
 *
 * returns:
 *      number of errors encountered
 */
int cfg_init()
{       register int i, j;
        int errs = 0;
        int helpcode = -1;
        char *helpstr = 0;

        /* make sure there are no holes in the command table */
        for( i = 0; i < MAX_ICODE; i++ )
        {       if (commands[i] == 0)
                {       bug( "uninitialized commands" );
                        errs++;
                }
                if (commands[i] == help)
                        helpcode = i;
        }

        /* make sure all the command strings have legal codes */
        for( i = 0; ed_cmds[i].c_code; i++ )
        {       
                if (ed_cmds[i].c_code < 0 || ed_cmds[i].c_code >= MAX_ICODE)
                {       bug( "illegal command code" );
                        errs++;
                }

                if (ed_cmds[i].c_code == helpcode)
                        helpstr = ed_cmds[i].c_string;
        }

        /* make sure there is a help command sequence defined */
        if (helpstr == 0)
        {       bug( "no help command defined" );
                errs++;
        } else
        {       /* print out program version and how to get help */
                (void) outstr( version );
                (void) outstr( release );
                (void) outstr( " - For help type '" );
                (void) showctl( helpstr );
                (void) outstr( "'\r\n" );
        }

        /* make sure the output sequences are all reasonable */
        for( i = 0; i < MAX_OCODE; i++ )
        {       if (outcodes[i])
                {       j = strlen( outcodes[i] );
                        if ( j < 0 || j >= MAX_OSEQ_LEN )
                        {       bug( "bad outcode length" );
                                errs++;
                        }
                } else
                {       bug( "null outcode" );
                        errs++;
                }
        }

        return( errs );
}

/*
 * routine:
 *      in_def
 *
 * purpose:
 *      to define a new control sequence
 *
 * parm:
 *      escape/command indication
 *      code
 *      pointer to string describing the sequence
 *
 * returns:
 *      0 - toto bene
 *      1 - couldn't do definition
 */
int in_def( esc, code, str )
 int esc;
 int code;
 char *str;
{       register struct ed_cmd *ep;
        static int userdefs = 0;

        /* check code for sanity */
        if (code <= 0 || code >= MAX_ICODE)
                return(1);

        /* if this is first user defined code, zero the cmd array */
        if (userdefs++ == 0)
        {       register int i;
                for( ep = ed_cmds; ep <= &ed_cmds[ MAX_CMDS ]; ep++ )
                {       ep->c_code = 0;
                        for( i = 0; i < MAX_ISEQ_LEN; i++ )
                                ep->c_string[i] = 0;
                }
        }

        /* find a free slot to put it in */
        for( ep = ed_cmds; ep < &ed_cmds[ MAX_CMDS ]; ep++ )
                if (ep->c_code == 0)
                {       ep->c_code = code;
                        ep->c_length = readctl(str, ep->c_string, MAX_ISEQ_LEN);
                        if (ep->c_length < 0)
                                return( 1 );
                        if (!esc)
                                ep->c_length = 0;
                        return( 0 );
                }

        /* we didn't find a free one */
        bell();
        (void) outstr( "\r\nToo many commands in configuration file\r\n");
        return( 1 );
}

/*
 * routine:
 *      out_def
 *
 * purpose:
 *      to define a new output control sequence
 *
 * parm:
 *      code
 *      pointer to string describing the sequence
 *
 * returns:
 *      0 - toto bene
 *      1 - couldn't do definition
 */
int out_def( code, str )
 int code;
 char *str;
{       register int i, j;
        static int userdefs = 0;

        /* check code for sanity */
        if (code < 0 || code >= MAX_OCODE)
                return(1);

        /* if he is defining sequences, lose the defaults */
        if (userdefs++ == 0)
                for( i = 0; i < MAX_OCODE; i++ )
                        for( j = 0; j < MAX_OSEQ_LEN; j++ )
                                outcodes[i][j] = 0;

        /* read in the control sequence */
        if (readctl( str, outcodes[code], MAX_OSEQ_LEN ) < 0)
                return( 1 );
        else
                return( 0 );
}

/*
 * routine:
 *      showctl
 *
 * purpose:
 *      to print an escape sequence in a readable form
 *
 * parm:
 *      string to be printed
 *
 * returns:
 *      number of characters printed
 */
showctl( s )
 register char *s;
{       register int bytes = 0;

        while( *s )
        {       if (*s < ' ')
                {       outchr( '^' );
                        outchr( '@' + *s );
                        bytes += 2;
                } else if (*s == 0x7f)
                {       outchr( '^' );
                        outchr( '?' );
                        bytes += 2;
                } else
                {       outchr( *s );
                        bytes++;
                }

                s++;
        }

        return( bytes );
}

/*
 * routine:
 *      readctl
 *
 * purpose:
 *      to read a character string describing an escape sequence
 *
 * parms:
 *      pointer to string
 *      pointer to buffer for sequence
 *      maximum buffer length
 *      
 * returns:
 *      number of bytes in sequence
 *      -1 -> bad sequence definition
 */
int readctl( str, buf, max )
 register char *str;
 register char *buf;
 int max;
{       register int len = 0;
        int i;
        char esc;

        /* first character of the sequence is the delimiter */
        esc = *str++;

        while( *str && *str != esc )
        {       /* check for being too long */
                if (len >= max-1)
                {       bug( "Excessively long sequence" );
                        return( -1 );
                }

                /* ^ introduces a control character */
                if (*str == '^')
                {       str++;
                        if (*str != '^')
                        {       buf[len++] = *str++ & 0x1f;
                                continue;
                        }
                        /* double ^ falls through as a single one */
                }

                /* \ introduces a 3 digit octal code */
                if (*str == '\\')
                {       str++;
                        if (*str != '\\')
                        {       buf[len++] = ((str[0] - '0') << 6) +
                                             ((str[1] - '0') << 3) +
                                             ((str[2] - '0') << 0);
                                str+= 3;
                                continue;
                        }
                        /* double \ falls through as a single one */
                }

                /* everything else is a normal character */
                buf[len++] = *str++;
        }

        
        /* make sure there is room for the null */
        if (len >= max)
        {       bug( "sequence too long" );
                return( -1 );
        } else
        {       for( i = len; i < max; i++ )
                        buf[i] = 0;
                return( len );
        }
}

/*
 * routine:
 *      help
 *
 * purpose:
 *      print out the menu of available commands
 */
VOID help( str )
 char *str;
{       register struct ed_cmd *cp;
        int column = 0;
        extern int maxcols;

        clrscrn();
        (void) outstr( version );
        (void) outstr( release );
        (void) outstr( copyright );
        (void) outstr( "\r\n\nCommand Sequences:\r\n" );

        for( cp = ed_cmds; cp->c_code; cp++ )
        {       /* if it has no name, we can't describe it */
                if (cmdnames[ cp->c_code ] == 0)
                        continue;

                /* else output string and its description */
                column += outstr( "  '" );
                column += showctl( cp->c_string );
                column += outstr( "' ");
                while( column % 20 )
                        column += outstr( "." );

                column += outstr( " " );
                column += outstr( cmdnames[ cp->c_code ] );

                /* try to output two commands per row */
                if (column < (maxcols/2))
                {       while( column < (maxcols/2) )
                                column += outstr( " " );
                } else
                {       (void) outstr( "\r\n" );
                        column = 0;
                }
        }

        (void) outstr( "\r\nPress any key to continue\r\n" );
        (void) readkbchar();
        repaint();
}
