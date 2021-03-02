/*                  _____________________________________________
                  [>                                             <]
Project           [> PortNet                                     <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> who structure handling                      <]
Current version   [> 00.01                                       <]
Version date      [> 09-June-1991                                <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>    This source is COPYRIGHT (c) 1991 by     <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"         /* Standard library files */
#include "mail.h"            /* Mail format */

/*** Translate a 'who' structure to text **********************************/
/*** Entry: who              - Pointer to structure
 ***        buffer           - Where to put result
 *** Exit : No exit data
 ***/

void who_totext(who *w,char *buffer)
  {
  switch(w->address_type)
    {
    case 0: /* Internal user */
    case 1: /* PortNet user */
      {
      sprintf(buffer,"%s (%s)",w->address_text,w->address_name);
      break;
      }
    case 2: /* Fidonet user */
      {                                  
      if (w->address_data[3])
        {
        sprintf(buffer,"%s of %d:%d/%d.%d",w->address_name,
                       w->address_data[0],w->address_data[1],
                       w->address_data[2],w->address_data[3]);
        }
      else
        {
        sprintf(buffer,"%s of %d:%d/%d",w->address_name,
                       w->address_data[0],w->address_data[1],
                       w->address_data[2]);
        }
      break;
      }
    case 3: /* UUCP */
      {
      sprintf(buffer,"%s (%s)",w->address_text,w->address_name);
      break;
      }
    }
  }
