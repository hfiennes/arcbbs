/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Additional port commands header             <]
Current version   [> 00.11                                       <]
Version date      [> 13-November-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/90/91 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

/* Readline/Readchar options */
#define NOECHO    0x001
#define HIDE      0x002
#define EXISTING  0x004
#define NUMONLY   0x008
#define TOUPPER   0x010
#define NOBOX     0x020
#define ENDBOX    0x040
#define NOTERM    0x080
#define TOLOWER   0x100

#define SENDFILE_NONSTOP   1
#define SENDFILE_ERROR     2

#include <time.h>

extern void port_crlf(void),port_readline(char*,int,int),window_poll(void),
            port_txcstring(char*,int),showtime(time_t),port_field(int);

extern int  port_txstring(char*,int),port_yesno(char*),port_readchar(int),
            port_get(void),bbs_sendfile(char*,char*,int);

extern char *cctime(time_t*);
