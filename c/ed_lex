/*
 * module:
 *      ed_lex.c        version 1.0
 *
 * purpose:
 *      this module contains the input lexer and command recognition
 *      routines for the in-core full-screen mini-editor
 *
 * external entry points:
 *      inp_init        initialize the input lexer tables
 *      inp_char        process a single character of input
 *
 * internal entry points:
 *      check_seq       determine whether or not an input sequence is a command
 *
 * principal data structures:
 *      inp_class       array, indexed by ascii code, information on
 *                      how that code should be processed
 */

#include "ed_rtns.h"
#include "ed_defs.h"

#define CL_TEXT         0x01    /* this code is legal as text input     */
#define CL_COMMAND      0x02    /* this code can start a cmd sequence   */

char inp_class[128] =   /* each entry is a bit-mask, made up of CL_bits */
{       0,      0,      0,      0,      0,      0,      0,      0,
        0,      0,      0,      0,      0,      0,      0,      0,
        0,      0,      0,      0,      0,      0,      0,      0,
        0,      0,      0,      0,      0,      0,      0,      0,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,
        CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,CL_TEXT,0
};

/* command and special sequenecs, imported from ed_config.c */
extern struct ed_cmd ed_cmds[];

extern int (*commands[])();     /* commands associated with each code   */

int     editmode;               /* controls rules for recognizing cmds  */

/*
 * routine:
 *      inp_init
 *
 * purpose:
 *      to initialize the lexer tables
 *
 * parms:
 *      no formal parameters 
 *      the ed_cmds structure has been initialized with all the valid strings
 *
 * returns:
 *      number of errors encountered
 *
 * algorithm:
 *      for each command sequence
 *              note the first character in the inp_class array
 *      for each special sequence
 *              note the first character in the inp_class array
 */
int inp_init()
{       int i, j;
        int errs = 0;

        /* note the first character of each command sequence */
        for( i = 0; ed_cmds[i].c_code; i++ )
        {       for( j = 0; j < MAX_ISEQ_LEN; j++ )
                {       if (ed_cmds[i].c_string[j] & 0x80)
                        {       bug( "illegal character in cmd sequence" );
                                errs++;
                        }
                }

                /* check for sequence starting with null */
                if (ed_cmds[i].c_string[1]  &&  !ed_cmds[i].c_string[0])
                {       bug( "cmd sequence starts with null" );
                        errs++;
                }

                inp_class[ ed_cmds[i].c_string[0] ] |= CL_COMMAND;
        }

        return( errs );
}

/*
 * routine:
 *      inp_char
 *
 * purpose:
 *      to process a single character of keyboard input
 *
 * parms:
 *      the input character code
 *
 * returns:
 *      void
 *
 * algorithm:
 *      stick character on the end of the current sequence buffer
 *      determine whether or not we're in a sequence
 *      if so
 *              if sequence is complete process it
 *              if sequence is incomplete, return and await more input
 *      if not
 *              process the entire sequence buffer as input text
 *
 * notes:
 *      if we are in a long command sequence, it may be twenty bytes
 *      before we know whether or not we're in a command, so we have
 *      to buffer up all the input - so that if it turns out not to
 *      be a command, we cna process all of those input characters
 *      as text.
 *
 *      in order to support a VI type interface, we provide a dual input
 *      command mode option.  In command mode, command sequences can begin
 *      with graphics, and non-graphics that aren't commands are errors.
 *      In input mode, all commands must begin with non-graphics.
 */
void inp_char( c )
 register char c;
{       register int i;
        int class;
        static  char    inp_seq[MAX_ISEQ_LEN];  /* accumulated input sequence */
        static  int     inp_seqlen;             /* length of that sequence    */

        /* stick the character into our input sequence buffer */
        inp_seq[ inp_seqlen++ ] = c;
        class = inp_class[c];

        /* if we are in auto-insert/rpl mode, there are no text commands */
        if ((editmode == 'I' || editmode == 'R') && (class & CL_TEXT))
                class &= ~CL_COMMAND;

        /* see if we might be starting (or in) a sequence */
        if ((inp_seqlen > 1) || (class & CL_COMMAND))
        {       i = check_seq( ed_cmds, inp_seq, inp_seqlen );
                if (i > 0)
                {       (void) (*commands[i])( inp_seq, inp_seqlen );
                        inp_seqlen = 0;
                        return;
                }

                /* if sequence is not yet complete, just return */
                if (i < 0 && inp_seqlen < MAX_ISEQ_LEN - 1)
                        return;

                /* 
                 * it was a non-match or too long for a command,
                 * so beep and discard the input
                 */
                inp_seqlen = 0;
                bell();
                return;
        }

        /* if we are in command mode, there is no text insertion */
        if (editmode == 'C')
        {       inp_seqlen = 0;
                bell();
                return;
        }

        /* we are not in a special sequence, so process it all as text */
        for( i = 0; i < inp_seqlen; i++ )
        {       if (inp_class[ inp_seq[i] ] & CL_TEXT)
                        text( inp_seq[i] & 0xff );
                else
                        bell(); /* just beep for illegal input */
        }

        /* we have completely processed the input sequence buffer */
        inp_seqlen = 0;
}

/*
 * routine:
 *      check_seq
 *
 * purpose:
 *      to determine whether or not the user has entered a special sequence
 *
 * parms:
 *      table of ed_cmd structures we should search
 *      pointer to input sequence (not null terminated)
 *      length of input sequence
 *
 * returns:
 *      0       if sequence doesn't match anything in the table
 *      -1      if sequence might match, but is incomplete
 *      >0      command number of sequence that we matched
 *
 * algorithm:
 *      for each entry in table
 *          see if the first character matches
 *          if the entry is length controlled and the length is right
 *              return the associated command code
 *          else if the whole sequence matches
 *              return the associated command code
 *          else
 *              note a partial match
 *
 *      return -1 or 0, based on partial matches
 */
int check_seq( table, buf, len )
  struct ed_cmd *table;
  register char *buf;
  int len;
{       register int i;
        int partials = 0;

        /* check each entry in the table */
        for( ; table->c_code; table++ )
        {       /* if first character doesn't match, this one is wrong */
                if (buf[0] == table->c_string[0])
                {       /* check for a length controlled special sequence */
                        if (table->c_length > 0)
                        {       /* if we've fulfilled the length, bingo */
                                if (table->c_length <= len)
                                        return( table->c_code );
                                else
                                        return( -1 );
                        } else
                        {       /* look for a non-matching character */
                                for( i = 1; i < len; i++ )
                                        if (buf[i] != table->c_string[i])
                                                break;

                                /* if we found a non-match, write this off */
                                if (i < len)
                                        continue;

                                /* if whole string matches, bingo */
                                if (i == len  &&  table->c_string[i] == 0)
                                        return( table->c_code );
                        }
                        partials++;
                }
        }

        return( partials ? -1 : 0 );
}
