/*
 * (C) Copyright 1988, Software by Sagredo, "est Omnibus"
 *
 *      this product may be freely used, duplicated and distributed
 *      provided that no charge is made for this code, and that this
 *      copyright notice is preserved in all copies and derivative works.
 *
 * module:
 *      ed_state.h              version 1.2
 *
 * purpose:
 *      to define the variables that keep track of the state of the edit
 *      and capabilities of the terminal.
 */

/* the current file position of the cursor and the display      */
extern int      currow;         /* current row: 0 - MAXROW-1    */
extern int      curcol;         /* current col: 0 - MAXCOL-1    */
extern int      curatr;         /* current attribute code       */
extern int      firstline;      /* row # of first line on scrn  */
extern int      insmode;        /* insert mode: 0, 1            */
extern int      numlines;       /* number of lines in file      */
extern char     *filename;      /* name of the file we're in    */

/* information about physical state of the screen               */
extern int      scrnrow;        /* actual row of phys cursor    */
extern int      scrncol;        /* actual col of phys cursor    */

/* terminal capability information                              */
extern int      c_ansiatr;      /* uses ansi seg grfx rendition */
extern int      c_autoscroll;   /* cursor autoscroll on maxrow  */
extern int      c_autowrap;     /* cursor autowrap cc80->cc1    */
extern int      c_base;         /* base for rows/cols  0/1      */
extern int      c_console;      /* we are running on PC console */
extern int      c_del;          /* can delete characters        */
extern int      c_ins;          /* can insert characters        */
extern int      c_hrel;         /* can do relative cursor r/l   */
extern int      c_vrel;         /* can do relative cursor u/d   */
extern int      c_ilines;       /* can insert new lines in scrn */
extern int      c_dlines;       /* can delete lines from scrn   */
extern int      c_uindex;       /* can index up and scroll      */
extern int      c_dindex;       /* can index down and scroll    */

/* basic parameters and limits for this editing session         */
extern int      editmode;       /* auto-insert/replace/text     */
extern int      maxrows;        /* number of rows on screen     */
extern int      maxcols;        /* number of cols on screen     */
extern int      maxlines;       /* number of lines in file      */
extern int      tabwidth;       /* width of a tab stop          */
extern int      wrapmgn;        /* where to force word-wraps    */
extern int      o_autoindent;   /* wordwrap should auto-indent  */

/* information about each line in the file                      */
struct line
{       char   *l_shifts;       /* pointer to list of shifts    */
        char    l_flags;        /* other info about this line   */
        char    l_text[1];      /* the actual character buffer  */
                                /* is maxcols + 1 bytes long    */
                                /* the extra byte is handy      */
};
extern  struct line **lines;

/* macros for accessing information about a particular line     */
#define LINEBUF(l)      lines[l]->l_text        /* text         */
#define LINEFLGS(l)     lines[l]->l_flags       /* flags        */
#define LINESHIFTS(l)   lines[l]->l_shifts      /* shifts       */

#define SOFT_CR 0x01            /* l_flags - word wrapped eol   */
#define WRAP_OK 0x02            /* l_flags - ok to wrap to line */

/* information about an expanded line (ready for display)       */
extern int x_buflen;            /* number of chars in line      */
extern int x_linewid;           /* number of columns in line    */
