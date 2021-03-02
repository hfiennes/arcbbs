/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCterm VII                                 <]
Author            [> Hugo Fiennes                                <]
Date started      [> 05-March-1990                               <]
                  [>                                             <]
Module name       [> Kermit defines                              <]
Current version   [> 00.01                                       <]
Version date      [> 06-December-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>    This source is COPYRIGHT (c) 1991 by     <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#define NAMELEN 8
#define EXTLEN 3

#define MAXPACKSIZ  94      /* Maximum packet size */
#define MAXEPACKSIZ 2000    /* Maximum extended packet size */
#define SOH         1       /* Start of header */
#define CR          13      /* ASCII Carriage Return */
#define SP          32      /* ASCII space */
#define DEL         127     /* Delete (rubout) */

#define MAXTRY      5       /* Times to retry a packet */
#define MYQUOTE     '#'     /* Quote character I will use */
#define MY8BITQUOTE '&'     /* 8th-bit quote character */
#define MYPAD       0       /* Number of padding characters I will need */
#define MYPCHAR     0       /* Padding character I need (NULL) */

#define MYEOL       '\n'    /* End-Of-Line character I need */

#define MYTIME      10      /* Seconds after which I should be timed out */
#define MAXTIM      60      /* Maximum timeout interval */
#define MINTIM      2       /* Minumum timeout interval */
#define DEFTIM      10      /* Default timeout (before send init) */

/* Macro Definitions */

/*
 * tochar: converts a control character to a printable one by adding a space.
 *
 * unchar: undoes tochar.
 *
 * ctl:    converts between control characters and printable characters by
 *         toggling the control bit (ie. ^A becomes A and A becomes ^A).
 */

#define tochar(ch)  ((ch) + ' ')
#define unchar(ch)  ((ch) - ' ')
#define ctl(ch)     ((ch) ^ 64 )

#define COUNTED_STRING(s) strlen(s),s
