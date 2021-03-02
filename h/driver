/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCterm VII                                 <]
Author            [> Hugo Fiennes                                <]
Date started      [> 05-March-1990                               <]
                  [>                                             <]
Module name       [> Driver calls                                <]
Current version   [> 00.04                                       <]
Version date      [> 08-January-1993                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>   This source is COPYRIGHT © 1992/93 by     <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

extern int  (*driver)(int,...);
extern int  driver_block[];
extern int  *driver_speedtable,driver_flags,driver_version,driver_noofspeeds;
extern char *driver_info,*driver_creator;
extern void driver_init(void);

typedef struct
  {
  int number;
  char file[16];
  char info[32];
  } drivers_block;

extern drivers_block drivers[];

#define DRIVER_PUTBYTE        0
#define DRIVER_GETBYTE        1
#define DRIVER_PUTBLOCK       2
#define DRIVER_GETBLOCK       3
#define DRIVER_CHECKTX        4
#define DRIVER_CHECKRX        5
#define DRIVER_FLUSHTX        6
#define DRIVER_FLUSHRX        7
#define DRIVER_CONTROLLINES   8
#define DRIVER_MODEMCONTROL   9
#define DRIVER_RXERRORS      10
#define DRIVER_BREAK         11
#define DRIVER_EXAMINE       12
#define DRIVER_TXSPEED       13
#define DRIVER_RXSPEED       14
#define DRIVER_WORDFORMAT    15
#define DRIVER_FLOWCONTROL   16
#define DRIVER_INITIALISE    17
#define DRIVER_CLOSEDOWN     18
#define DRIVER_POLL          19
#define DRIVER_SELECT        20

#define driver_txspeed(tx)     (*driver)(DRIVER_TXSPEED,portnumber,tx)
#define driver_rxspeed(rx)     (*driver)(DRIVER_RXSPEED,portnumber,rx)
#define driver_wordformat(w)   (*driver)(DRIVER_WORDFORMAT,portnumber,w)
#define driver_flowcontrol(f)  (*driver)(DRIVER_FLOWCONTROL,portnumber,f)
#define driver_initialise()    (*driver)(DRIVER_INITIALISE,portnumber)
#define driver_closedown()     (*driver)(DRIVER_CLOSEDOWN,portnumber)
#define driver_poll()          (*driver)(DRIVER_POLL,portnumber)

#define DFLAG_MOREPORTS      0x00000001
#define DFLAG_SPLITRATES     0x00000002
#define DFLAG_HASFIFO        0x00000004
#define DFLAG_SETBREAK       0x00000008
#define DFLAG_NEEDSPOLL      0x00000010
#define DFLAG_WONTEMPTY      0x00000020
#define DFLAG_SUPPORTSBLOCK  0x00000040
#define DFLAG_DONTOVIO       0x00000080
