/*
 * ed_rtns.h    version 1.5
 *
 *      this module contains external declarations for all of the global 
 *      entry-points in all the modules of the editor. 
 */

/* 
 * This is bogus, but PCC based C compilers have trouble declaring and
 * initializing an array of pointers to functions that return void.
 * Consequently, all of the routines in the command switch are delared
 * with this two-way trick type.  It is normally void, except in the module
 * that initializes the command switch.
 */
#ifndef VOID
#define VOID void
#endif

/* output routines in ed_tio.c */
extern void bug(), outchr(), outline(), tio_clup();
extern int outstr(), outnum(), readkbchr(), tio_init();
extern VOID bell();

/* screen manipulation functions in ed_out.c */
extern void term_init(), set_atr();
extern void abs_cur(), rel_cur();
extern void clrscrn(), clrline(), index();
extern void clschar(), clsline(), inschar(), insline(), insertion();

/* cursor and position control functions in ed_move.c */
extern void cursor(), move(), update();
extern VOID csr_left(), csr_right(), csr_up(), csr_down(), csr_home();
extern VOID csr_end(), csr_nl(), csr_cr(), csr_tab(), csr_btab(), csr_eol();
extern VOID csr_fword(), csr_bword(), csr_pgup(), csr_pgdn(), repaint();

/* editing command routines in ed_cmd.c */
extern VOID color(), control();
extern VOID c_insmd(), c_delch(), c_insln(), c_delln(), c_delwd(), c_delwdb();
extern VOID c_bspc(), c_brkln(), c_clrln(), c_joinln(), c_undel();
extern VOID c_autoi(), c_autor(), c_cmd(), c_rplall(), c_refill();
extern int c_find(), c_rpl();

/* editing utility functions in ed_misc.c */
extern VOID save(), quit();
extern char *expand();
extern int search(), solicit();

/* text manipulation functions in ed_text.c */
extern char *shiftalloc();
extern int shiftfree(), textins();
extern void splitline(), textclr(), textcpy(), textsuck();
extern VOID text();

/* input lexing functions in ed_lex.c */
extern void inp_char();

/* file I/O routines in ed_fio.c */
extern int openfile(), getline(), putline();
extern void closefile();
