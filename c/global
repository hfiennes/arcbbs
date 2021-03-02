/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Global shared bits                          <]
Current version   [> 00.05                                       <]
Version date      [> 17-December-1992                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"         /* Standard library files */
#include "config.h"          /* System configuration */
#include "servmess.h"        /* Server message info */
#include "userlog.h"         /* Userlog format */
#include "crc.h"             /* CRC generator */
#include "mail.h"            /* Mail format */
#include "servcomm.h"        /* Misc server comm commands */
#include "modcomm.h"         /* Module interface */
#include "stdarg.h"
#include "fido.h"

#include "miscbbs.h"
#include "bbs.h"

#ifdef SERVER  
extern void event_process(void);
#define window_poll() event_process()
static int mprintf(int a,char *b, ...) { return(0); }
#else
#include "mprintf.h"         /* Modem printf */
#endif             

void trim_area(int trim,int maxmsg,int verbose)
  {
  mail_area trimarea;
  mail_block header; clock_t lastpoll=clock(),lastclose=clock();
  int link,delcount=0;

  read_area(trim,&trimarea);

  if ((trimarea.areaflags&FLAG_FILEAREA)!=0) return;
  if (trimarea.count<=maxmsg) return;

  maxmsg=trimarea.count-maxmsg;
         
  /* Remove them! */
  link=trimarea.first_message;
  do
    {
    header.message_number=link;
    get_messageh(&header);
    link=header.message_forward;
    delete_message(header.message_number|(1<<31));
    delcount++;   

    if ((clock()-lastpoll)>MAXIMUM_POLL)
      {
      window_poll(); lastpoll=clock();
      }

    if ((clock()-lastclose)>500)
      {
      if (verbose) mprintf(1,"\015%5d messages removed",delcount);
      mod_ensurelookup(); mod_savemaps(); mod_ensuredata();
      lastclose=clock();
      }

    maxmsg--;
    }
  while(maxmsg>0 && link>=0);

  if (verbose)
    {
    mprintf(1,"\015%5d messages removed\n",delcount);
    }
  mod_ensurelookup(); mod_savemaps(); mod_ensuredata();
  }

void error(char *format, ...)
  {
  char buffer[512];
  va_list arg_pointer;
  FILE *erl;
  time_t now;
  struct tm *nowa;
  static char class[][10]=
    { "unknown ", "message  ", "fidoIN   ", "fidoOUT  ", "SysopUtl", "Script  " };

  va_start(arg_pointer,format);
  vsprintf(buffer+40,format,arg_pointer);
  va_end(arg_pointer);

  if ((erl=fopen("<ARCbbs$errorlog>","a"))!=NULL)
    {
    time(&now);
    nowa=localtime(&now);
 
    strftime(buffer,30,"%H:%M:%S %a %b %d",nowa);
    fprintf(erl,"%s [%-9s] %s\n",buffer,
            class[buffer[40]-97],buffer+41);
          
    fclose(erl);
    }
  }

void syslog(char *segment,char *format, ...)
  {
  char buffer[512];
  va_list arg_pointer;
  FILE *erl;
  time_t now;
  struct tm *nowa;

  va_start(arg_pointer,format);
  vsprintf(buffer+40,format,arg_pointer);
  va_end(arg_pointer);
  
  if ((erl=fopen("<ARCbbs$systemlog>","a"))!=NULL)
    {
    time(&now);
    nowa=localtime(&now);
 
    strftime(buffer,30,"%H:%M:%S %a %b %d",nowa);
    fprintf(erl,"%s [%-8s] %s\n",buffer,segment,buffer+40);
          
    fclose(erl);
    }
  }

void _fclose(FILE **f)
  {
  if (*f)
    {
    fclose(*f);
    *f=0;
    }
  }
