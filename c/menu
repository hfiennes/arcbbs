/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Menu support                                <]
Current version   [> 00.04                                       <]
Version date      [> 13-November-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>    This source is COPYRIGHT (c) 1991 by     <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"         /* Standard library files */
#include "crc.h"             /* CRC generator */
#include "mail.h"            /* Mail format */
#include "mprintf.h"         /* Modem printf */
#include "port.h"
#include "portmisc.h"
#include "str.h"
#include "userlog.h"
#include "bbs.h"
#include "fido.h"
#include "miscbbs.h"
#include "servmess.h"
#include "servcomm.h"
#include "modcomm.h"
#include "config.h"

static int option,stack=0;
static char menustack[10][12],path[40],token[80],aborton[30];
             
void menu_initialise()
  {
  stack=0; path[0]=0;
  }

void menu_closedown()
  {
  }

static int menu_gettoken(char **t)
  {
  char *tail=*t,*p=token;

  while(*tail==' ' && *tail) tail++;
  if (*tail)
    {
    while(*tail!=' ' && *tail) *p++=*tail++;
    *p=0;
    }
  else *p=0;

  *t=tail;
  return(*token);
  }

static int c_type(char *tail)
  {
  menu_gettoken(&tail);

  if (tolower(token[0])=='n') bbs_sendfile(tail,aborton,SENDFILE_NONSTOP);
  else bbs_sendfile(tail,aborton,SENDFILE_NONSTOP);
  return(0);
  }

static int c_getoption(char *tail)
  {
  int try;

  do
    {
    try=tolower(port_get());
    if (strchr(tail,try)!=NULL)
      {
      option=try;
      return(OK);
      }
    }
  while(1);

  return(0);
  }

static int c_echo(char *tail)
  {
  mprintf(1,"%s",tail);
  return(OK);
  }

static int c_run(char *tail)
  {
  /*script_initialise();
  script_run(tail);
  script_closedown(1);*/
  return(OK);
  }

static int c_doing(char *tail)
  {
  tail=tail;
  return(OK);
  }

static int c_security(char *tail)
  {
  do
    {
    if (menu_gettoken(&tail))
      {
      if (strnicmp(token,"level>",6))
        {
        if (user.userlevel<=atoi(token+6)) return(-102);
        }
      if (strnicmp(token,"level<",6))
        {
        if (user.userlevel>=atoi(token+6)) return(-102);
        }
      if (strnicmp(token,"level=",6))
        {
        if (user.userlevel!=atoi(token+6)) return(-102);
        }
      }
    }
  while(*token);

  return(OK);
  }

static int c_usernumber(char *tail)
  {
  do
    {
    if (menu_gettoken(&tail))
      {
      if (token[0]=='!')
        {
        if (atoi(token+1)==user.usernumber) return(-102);
        }
      else
        {
        if (atoi(token)!=user.usernumber) return(-102);
        }
      }
    }
  while(*token);

  return(OK);
  }

static int c_flags(char *tail)
  {
  int a;

  do
    {
    if (menu_gettoken(&tail))
      {
      if (token[0]=='!')
        {
        a=findflag_user(token+1);

        if (a==-1) return(-102);
        if ((user.f_user&(1<<a))!=0) return(-102);
        }
      else
        {
        a=findflag_user(token);

        if (a==-1) return(-102);
        if ((user.f_user&(1<<a))==0) return(-102);
        }
      }
    }
  while(*token);
  
  return(OK);
  }

static int c_setflag(char *tail)
  {
  int a;
  
  do
    {
    if (menu_gettoken(&tail))
      {
      a=findflag_user(token);
      if (a!=-1) user.f_user|=(1<<a);
      }
    }
  while(*token);

  return(OK);
  }

static int c_clearflag(char *tail)
  {
  int a;

  do
    {
    if (menu_gettoken(&tail))
      {
      a=findflag_user(token);
      if (a!=-1) user.f_user&=~(1<<a);
      }
    }
  while(*token);
  
  return(OK);
  }

static int c_toggleflag(char *tail)
  {
  int a;

  do
    {
    if (menu_gettoken(&tail))
      {
      a=findflag_user(token);
      if (a!=-1) user.f_user^=(1<<a);
      }
    }
  while(*token);
  
  return(OK);
  }
                                               
static int c_readuser(char *tail)
  {
  tail=tail;
  return(0);
  }

static int c_writeuser(char *tail)
  {
  tail=tail;
  return(0);
  }

struct { char command[20]; int flags; int (*function)(char*); } menucommands[] =
     { { "endmenu"        ,  0, NULL         },
       { "option"         ,  1, NULL         },
       { "if"             ,  1, NULL         },
       { "restartmenu"    ,  2, NULL         },
       { "failmessage"    ,  3, NULL         },
       { "goto"           ,  4, NULL         },
       { "gosub"          ,  5, NULL         },
       { "return"         ,  6, NULL         },
       { "forcegoto"      ,  7, NULL         },
       { "displaymenu"    ,  8, NULL         },
       { "aborton"        ,  9, NULL         },
       { "path"           , 10, NULL         },
       { "logoff"         , 11, NULL         },
       { "drop"           , 12, NULL         },
       { "relogon"        , 13, NULL         },

       { "getoption"      ,100, c_getoption  },
       { "echo"           ,100, c_echo       },
       { "run"            ,100, c_run        },
       { "doing"          ,100, c_doing      },
       { "security"       ,100, c_security   },
       { "usernumber"     ,100, c_usernumber },
       { "flags"          ,100, c_flags      },
       { "setflag"        ,100, c_setflag    },
       { "clearflag"      ,100, c_clearflag  },
       { "toggleflag"     ,100, c_toggleflag },
       { "readuser"       ,100, c_readuser   },
       { "writeuser"      ,100, c_writeuser  },

       { "type"           ,100, c_type       },
       { ""               ,  0, NULL         } };
                
int do_menu(char *menuname)
  { 
  char line[128],*command,*tail,failmessage[128],menu[12],*t;
  int a,inoption,pluscount; FILE *mf;
        
  strcpy(menu,menuname);

  newmenu:
  inoption=0;
  strcpy(line,"<ARCbbs$MenuData>.");
  if (*path) strcat(line,path);
  strcat(line,menu);
  *failmessage=0; *aborton=0;
 
  if ((mf=fopen(line,"r"))==NULL)
    {
    error("aCan't open menu file %s",line); return(-1);
    }

  do
    {
    /* Get line */
    if (fgets(line,127,mf)==NULL) break;
    if (line[a=(strlen(line)-1)]==10) line[a]=0;

    /* Process it */
    if (line[0]==';') continue;

    /* Count +'s */
    pluscount=0;
    while(line[pluscount]=='+') pluscount++;

    /* What nesting level are we? */
    if (pluscount>inoption) continue;
    if (inoption>pluscount) inoption--;

    command=line+pluscount;

    while(*command && *command==' ') command++;
    if (*command==0) continue;
    tail=command;
    while(*tail && *tail!=' ') tail++;
    if (*tail)
      {
      *tail++=0; while(*tail==' ') tail++;
      if (*tail=='\"')
        {
        /* Get the data in quotes */
        t=++tail;
        while(*t!='\"' && *t) t++;
        *t=0;
        }
      else
        {
        /* Trim trailing spaces */
        t=tail+strlen(tail)-1;
        while(*t==' ') t--;
        *++t=0;
        }
      }

    /* Find command */
    a=0;
    while(strnicmp(menucommands[a].command,command,strlen(menucommands[a].command))!=0 &&
          menucommands[a].command[0]!=0) a++;
    
    if (menucommands[a].command[0]==0)
      {
      char bigbuff[256];

      strcpy(bigbuff,command);
      strcat(bigbuff," ");
      strcat(bigbuff,tail);
      do_command(bigbuff,1);
      /*mprintf(1,"Command '%s' not found\n",command);*/
      continue;
      }
                                        
    /* endmenu */
    if (menucommands[a].flags==0) break;

    switch(menucommands[a].flags)
      {
      /* 'option' command */
      case 1:
        {
        /* Check to see if option matches */
        if (strchr(tail,option)!=NULL || *tail==0)
          {
          /* Bump inoption count */
          inoption++;
          }
        break;
        }    
      /* go back to top of menu */
      case 2:
        {
        inoption=0;
        *failmessage=0;
        fseek(mf,0,SEEK_SET);
        break;
        }                      
      /* fail message */
      case 3:
        {
        strcpy(failmessage,tail);
        break;
        }
      /* goto */
      case 4:
        {
        fclose(mf);
        strcpy(menu,tail);
        goto newmenu;
        break;
        }
      /* gosub */
      case 5:
        {
        if (stack<10)
          {
          strcpy(menustack[stack++],menuname);
          fclose(mf);
          strcpy(menu,tail);
          goto newmenu;
          }
        break;
        }
      /* return */
      case 6:
        {
        if (stack)
          {
          fclose(mf);
          strcpy(menu,menustack[--stack]);
          goto newmenu;
          }
        break;
        }
      /* forcegoto */
      case 7:
        {
        stack=0;
        fclose(mf);
        strcpy(menu,tail);
        goto newmenu;
        break;
        }                     
      /* display menu */
      case 8:
        {
        strcpy(line,"<ARCbbs$MenuText>.");
        if (*path) strcat(line,path);
        strcat(line,menu);
        bbs_sendfile(line,aborton,0);
        break;
        }
      /* set aborton */
      case 9:
        {
        strcpy(aborton,tail);
        break;
        }
      /* path */
      case 10:
        {
        strcpy(path,tail);
        break;
        }
      /* normal command */
      case 100:
        {
        switch((*menucommands[a].function)(tail))
          {
          /* Decrement inoption */
          case -101:
            {
            inoption--;
            break;
            }
          /* Decrement inoption & display failmessage */
          case -102:
            {
            inoption--;
            if (*failmessage) mprintf(1,"%s",failmessage);
            break;
            }
          }
        break;
        }
      }
    }
  while(1);

  fclose(mf);
  if (stack)
    {
    strcpy(menu,menustack[--stack]);
    goto newmenu;
    }

  return(0);
  }
