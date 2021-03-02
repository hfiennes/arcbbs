/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Modem printf: mprintf                       <]
Current version   [> 00.04                                       <]
Version date      [> 03-October-1990                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/1990 by  <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "stdarg.h"
#include "stdio.h"
#include "port.h"
#include "portmisc.h"

int mprintf(int interrupt,char *format, ...)
  {
  char buffer[512];
  va_list arg_pointer;

  if (interrupt==0 && port_rxbuffer()) return(1);

  va_start(arg_pointer,format);
  vsprintf(buffer,format,arg_pointer);
  va_end(arg_pointer);

  return(port_txstring(buffer,interrupt));
  }
  
