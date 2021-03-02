/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> On this day...                              <]
Current version   [> 00.04                                       <]
Version date      [> 09-November-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/90/91 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"
#include "mprintf.h"
#include "port.h"

extern void gtime(struct tm*,time_t),mygets(char*,FILE*);
extern FILE *bbs_file[10];
extern char tempbuffer[256];                  
                 
void today()
  {
  struct tm now;
  char today_file[60],ourdate[5];
  int type=0;
     
  /* Get time */
  gtime(&now,time(NULL));       
  sprintf(ourdate,"%02d%02d",now.tm_mon+1,now.tm_mday);

  /* Try to open correct file */
  sprintf(today_file,"<ARCbbs$text>.Today.Today_%02d",now.tm_mon+1);
  if ((bbs_file[7]=fopen(today_file,"r"))==NULL)
    {
    mprintf(1,"{bfg r}Can't open %s{std}\n",today_file);
    return;
    }

  /* Scan through file */
  do
    {                              
    /* Get line */
    mygets(tempbuffer,bbs_file[7]);
             
    if (tempbuffer[0]!='*' && tempbuffer[0]!=0)
      {
      /* Right date? */
      if (strncmp(tempbuffer+1,ourdate,4)==0)
        {               
        int year=0;
          
        if (tempbuffer[9]!=' ' && tempbuffer[9]!='C')
          {
          if ((now.tm_wday+1)!=(tempbuffer[9]-'0')) continue;
          }
        if (tempbuffer[5]!=' ') year=atoi(tempbuffer+5);

        /* Print correct banner if necessary */
        if (tempbuffer[0]=='B' && type!='B')
          {
          mprintf(0,"{bfgbg wb}Birthdays{eol std}\n"); type='B';
          }
        if (tempbuffer[0]=='S' && type!='S')
          {
          mprintf(0,"{bfgbg wb}Special dates{eol std}\n"); type='S';
          }
        if (tempbuffer[0]=='R' && type!='R')
          {
          mprintf(0,"{bfgbg wb}Reminders{eol std}\n"); type='R';
          }
              
        /* Print line */
        if (tempbuffer[9]!='C')
          {
          if (year)
            {
            mprintf(0,"{bfg w}%2d/%02d/%04d {fg c}%s\n",now.tm_mday,
                                                    now.tm_mon+1,
                                                    year,tempbuffer+10);
            }
          else
            {
            mprintf(0,"{bfg w}%2d/%02d      {fg c}%s\n",now.tm_mday,
                                                    now.tm_mon+1,
                                                    tempbuffer+10);
            }
          }
        else
          {
          mprintf(0,"           {fg c}%s\n",tempbuffer+10);
          }
        }
      }
    }
  while(feof(bbs_file[7])==0 && tempbuffer[0]!=0 && port_rxbuffer()==0);

  if (port_rxbuffer()) port_txclear();
  
  fclose(bbs_file[7]); bbs_file[7]=0;
  }                 
