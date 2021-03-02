/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Doors interface assembler header            <]
Current version   [> 00.01                                       <]
Version date      [> 01-January-1992                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

typedef struct
  {
  int request;                 /* Request number */
  union
    {
    char b[252];
    int w[63];
    } data;
  } doors_request;

typedef union
  {
  char b[256];
  int w[64];
  } doors_reply;

extern int  doors_connect(int),doors_forceconnect(void),doors_inuse;
extern void doors_reset(void),doors_dorequest(doors_request*);
