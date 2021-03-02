/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCterm VII                                 <]
Author            [> Hugo Fiennes                                <]
Date started      [> 17-August-1990                              <]
                  [>                                             <]
Module name       [> Scrolling terminal header                   <]
Current version   [> 00.40                                       <]
Version date      [> 05-April-1993                               <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1990-1993 by    <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

extern void scr_start(void),scr_end(void),scr_plotchar(int,int,int),
            scr_init(void),scr_scrollmapup(void),scr_refresh(void),
            scr_clearline(int),scr_colourmap(void),scr_housekeep(void),
            scr_scrollmapdown(void),scr_caret(int),scr_setcaret(int),
            scr_changeterm(int),scr_print(int),tty_keyprocess(int),
            vt52_keyprocess(int),vt102_keyprocess(int),ansi_keyprocess(int),
            scr_keyprocess(int),scr_partscrollmapup(int,int),
            scr_partscrollmapdown(int,int),scr_select(int),
            scr_deselect(int),scr_direct(int,int,int),scr_closedown(void),
            scr_setflash(int),scr_totalredraw(void),scr_replaypause(void),
            scr_tab(int,int),term_setcaret(int),scr_send(int),
            vt102_setwidth(int),decompress_font(char*,int*,int),
            scr_readfonts(void),scr_toscrollback(int),scr_printscreen(void),
            term_caret(int),scr_redraw(wimp_redrawstr*,int*),
            scr_drawkeypad(void),scr_tofileproblem(os_error*),
            scr_bufferkey(int),scr_keydirect(int),scr_docursor(int),
            scr_claimkeypad(void),scr_releasekeypad(void),
            scr_carettype(void),scrcursor_setflash(int),
            scrcursor_setflash(int),scr_flash(int,void*),
            scrcursor_flash(int,void*),snoop_buffer(int),
            settitle(int,char*);
extern int  scr_process(int),ansi_process(int),vt102_process(int),
            tty_process(int),vt52_process(int),scr_getdata(int),
            scr_at(int,int),scr_xpos(void),scr_ypos(void),
            scr_width(void),hexdump_process(int),
            scr_flashprocess(int),showcontrol_process(int);
extern os_error *scr_tofile(int),*scr_torawfile(int),
            *scr_toanyfile(int,int);
extern BOOL scr_screensave(char*,void*),scr_spritesave(char*,void*);                       

extern int  scr_currentterm,scr_spoolfile,scr_spoolstrip,scr_available;
                                           
extern char isvda[16];

#define S_FLASH           0x00000002
#define S_INSERTMODE      0x00000004
#define S_TTYWRAP         0x00000008
#define S_INVERT          0x00000020
#define S_SMOOTH          0x00000080
#define S_NEWLINE         0x00000100
#define S_ORIGIN          0x00000200
#define S_LOCALECHO       0x00000800
#define S_VT100ID         0x00001000
#define S_TOFILE          0x00002000
#define S_STRIP           0x00004000
#define S_PENDINGCLS      0x00008000
#define S_WRAP            0x00100000
#define S_APPCURSOR       0x00200000
#define S_APPKEYPAD       0x00400000
#define S_AUTOPRINT       0x00800000
#define S_NOCONTROL       0x01000000
#define S_PRINTWHOLE      0x02000000
#define S_FORMFEED        0x04000000
#define S_CONTROLLER      0x08000000
#define S_BRIGHT          0x10000000
#define S_ANSWERBACK      0x20000000
#define S_132COLUMN       0x40000000
#define S_LOCK            0x80000000

#define ANSIMASK         (S_FLASH | S_INVERT | S_WRAP)
#define VT102MASK        (S_FLASH | S_INVERT | S_SMOOTH | S_NEWLINE | S_WRAP | S_APPCURSOR | S_APPKEYPAD | S_AUTOPRINT | S_NOCONTROL | S_PRINTWHOLE | S_FORMFEED | S_PRINTDISABLE | S_BRIGHT | S_ANSWERBACK | S_LOCK)

#define DISPLAY_DATA(x) scr_process(x);

typedef struct
  {
  int  *wordmap;          /* Pointer to word map for screen */
  int  *sprite;           /* Pointer to sprite */
  int  *charset;          /* Characterset */

  int  minx,maxx,
       miny,maxy;         /* Min/max x and y updated positions */

  int  currentattrib;     /* Current screen attributes */

  int  *wordmaptop;       /* Top of wordmap */
  int  *spritetop;        /* Top of sprite */

  sprite_area *spritearea;/* Pointer to sprite area */
  int  *savearea;         /* pointer to save area */

  int  x,y;               /* Current screen position */
  int  g0,g1;             /* Current sets (VT102 only) */
  int  width;             /* Screen width (VT102 only) */

  int *spriteptr;         /* Pointer to sprite */

  int  cursortype;        /* 0=block, 1=line */
  int  cursorstatus;
  } struct_scr;

#define BACKGROUND 24
#define FOREGROUND 28
#define BOLD       0x80000000
#define FLASHHIDE  0x00010000
#define UNDERLINE  0x00020000
#define FLASH      0x00040000
#define REVERSE    0x00080000
#define SELECT     0x00100000
#define DH_TOP     0x00200000
#define DH_BOTTOM  0x00400000
#define DW         0x00600000
#define CURSOR     0x00800000

#define DUB        0x00600000
#define DUBMASK    0xff9fffff

#define CURSORTIME 40
