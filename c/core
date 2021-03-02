/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Core non-overlay bits                       <]
Current version   [> 00.10                                       <]
Version date      [> 17-November-1992                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"         /* Standard library files */
#include "config.h"          /* System configuration */
#include "servmess.h"        /* Server message info */
#include "port.h"            /* Port driver functions */
#include "userlog.h"         /* Userlog format */
#include "crc.h"             /* CRC generator */
#include "parser.h"          /* Command parser */
#include "mail.h"            /* Mail format */
#include "portmisc.h"        /* Misc port commands */
#include "servcomm.h"        /* Misc server comm commands */
#include "mprintf.h"         /* Modem printf */
#include "arclist.h"         /* ARC file lister */
#include "modcomm.h"         /* Module interface */
#include "fido.h"            /* Fido stuff */

#include "miscbbs.h"
#include "bbs.h"

#include "stdarg.h"

int noof_userflags,noof_msgflags,noof_fileflags;
char userflag[32][22],msgflag[32][22],fileflag[32][22];
                    
int can_write(mail_area *test)
  {                                               
  if ((user.f_user&USER_SYSOP)!=0) return(1);
  if (test->editor_id==user.usernumber) return(1);
  if ((test->areaflags&FLAG_WRITEABLE)==0) return(0);
  if (user.userlevel<test->level) return(0);
  if ((test->areaflags&FLAG_FILEAREA)==0)
    {
    if ((user.f_message&test->flags)==test->flags) return(1);
    }
  else
    {
    if ((user.f_file&test->flags)==test->flags) return(1);
    }
  return(0);
  }
                                                
int can_read(mail_area *test)
  {                                               
  if ((user.f_user&USER_SYSOP)!=0) return(1);
  if (test->editor_id==user.usernumber) return(1);
  if ((test->areaflags&FLAG_READABLE)==0) return(0);
  if (user.userlevel<test->level) return(0);
  if ((test->areaflags&FLAG_FILEAREA)==0)
    {
    if ((user.f_message&test->flags)==test->flags) return(1);
    }
  else
    {
    if ((user.f_file&test->flags)==test->flags) return(1);
    }
  return(0);
  }

void mygets(char *buffer,FILE *file)
  {
  int ch,ch2;

  if (feof(file))
    {
    *buffer=0; return;
    }

  do
    {
    ch=*(buffer++)=fgetc(file);
    }
  while(ch!=10 && ch!=13 && !feof(file)); 

  /* Wipe out any following CR or LF */
  ch2=fgetc(file);
  if (ch2!=13 && ch2!=10) ungetc(ch2,file);
  else
    {
    if ((ch2==10 && ch==10) || (ch2==13 && ch==13)) ungetc(ch2,file);
    }
      
  *(buffer-1)=0;
  }
     
int timerset(int howlong)
  {
  return(clock()+(howlong*10));
  }
    
void gtime(struct tm *to,time_t from)
  {
  struct tm *ptt;
  ptt=localtime(&from);
  to->tm_sec =ptt->tm_sec;
  to->tm_min =ptt->tm_min;
  to->tm_hour=ptt->tm_hour;
  to->tm_mday=ptt->tm_mday;
  to->tm_mon =ptt->tm_mon;
  to->tm_year=ptt->tm_year;
  to->tm_wday=ptt->tm_wday;
  to->tm_yday=ptt->tm_yday;
  to->tm_isdst=ptt->tm_isdst;
  }
          
char *fido_rules(char *pseudonym)
  {
  static char return_name[31],pseud[31];

  /* Open translation file */
  if ((bbs_file[7]=fopen("<ARCbbs$miscdata>.Fido.Translate","r"))==NULL)
    {
    return(pseudonym);
    }
  
  do
    {  
    /* Read lines */
    mygets(pseud,bbs_file[7]);
    mygets(return_name,bbs_file[7]);
    
    /* Match? */
    if (strcmp(pseud,pseudonym)==0)
      {
      _fclose(&bbs_file[7]);
      return(return_name);
      }
    }
  while(pseud[0]);

  _fclose(&bbs_file[7]);
  return(pseudonym);
  }

int timeup(int when)
  {
  window_poll();
  return(clock()>=when);
  }

void conf_join(unsigned char *conf,int confnr)
  {
  *(conf+(confnr/8))|=(1<<(confnr%8));
  }

void conf_resign(unsigned char *conf,int confnr)
  {
  *(conf+(confnr/8))&=~(1<<(confnr%8));
  }

int conf_member(unsigned char *conf,int confnr)
  {
  return((*(conf+(confnr/8))&(1<<(confnr%8)))!=0);
  }

int flag_process(char *buffer)
  {
  int a;

  for(a=0;a<strlen(buffer);a+=2)
    {
    int number=toupper(buffer[a+1]);
    if (number>='0' && number<='9') number-='0';
    else number-=('A'-10);

    /* Check flag type */
    switch(buffer[a])
      {
      case 'U': /* ON User flag */
        {
        if ((user.f_user&(1<<number))==0) return(0);
        break;
        }
      case 'F': /* ON File flag */
        {
        if ((user.f_file&(1<<number))==0) return(0);
        break;
        }
      case 'M': /* ON Message flag */
        {
        if ((user.f_message&(1<<number))==0) return(0);
        break;
        }
      case 'u': /* OFF User flag */
        {
        if ((user.f_user&(1<<number))!=0) return(0);
        break;
        }
      case 'f': /* OFF File flag */
        {
        if ((user.f_file&(1<<number))!=0) return(0);
        break;
        }
      case 'm': /* OFF Message flag */
        {
        if ((user.f_message&(1<<number))!=0) return(0);
        break;
        }
      }
    }
  return(1);
  }

void do_arc(char *arcname,char *arcname1,char *command,char *files)
  {
  wimp_msgstr message;
  clock_t startarc,lastmsg;
  int showmsg=1;
                      
  lastmsg=clock();
  do
    {
    strcpy(message.data.chars,"arc");
    message.hdr.size=4*((24+strlen(message.data.chars))/4+1);
    message.hdr.your_ref=0;
    message.hdr.action=BBS_ARC;
    wimp_sendmessage(wimp_ESEND,&message,0);

    /* Extra one to catch broadcast to ourselves */  
    got_type=0; while(got_type!=BBS_ARC) window_poll(); 
    startarc=clock(); got_type=0;

    while(got_type!=BBS_ARC && (clock()-startarc)<100) window_poll();
    if (got_type!=BBS_ARC)
      {                  
      if ((clock()-lastmsg)>500)
        {
        if (showmsg==1)
          {
          mprintf(1,"{bfg r}WAIT: {fg c}Spark busy\010\010\010\010\010\010\010\010\010\010\010\010\010\010\010\010");
          showmsg=2;
          }
        else
          {
          mprintf(1,"{bfg r}wait: {fg c}Spark busy\010\010\010\010\010\010\010\010\010\010\010\010\010\010\010\010");
          showmsg=1;
          }
        lastmsg=clock();
        }
      }
    }  
  while(got_type!=BBS_ARC);
             
  if (command==NULL) command="";
       
  mprintf(1,"{fg g}compressing...\010\010\010\010\010\010\010\010\010\010\010\010\010\010");
  sprintf(message.data.chars,"arc -y%d %s %s%s %s",MAXIMUM_POLL,
          command,arcname,arcname1,files);    

  message.hdr.size=4*((24+strlen(message.data.chars))/4+1);
  message.hdr.your_ref=0;
  message.hdr.action=BBS_ARC;
  wimp_sendmessage(wimp_ESEND,&message,0);

  /* Extra one to catch broadcast to ourselves */
  got_type=0; while(got_type!=BBS_ARC) window_poll(); 
  got_type=0; while(got_type!=BBS_ARC) window_poll(); 
  }

char *file_pathname(int filenumber)
  {
  static char pathname[80];

  if (filenumber>=0)
    {
    sprintf(pathname,"<ARCbbs$filedata>.%06d.%06d.%06d",((filenumber/2500)*2500),
                   ((filenumber/50)*50),filenumber);
    }
  else
    {
    switch(filenumber)
      {
      case FILE_SCRATCHPAD:
        {
        sprintf(pathname,"<ARCbbs$temp>.Scratch_%d",portnumber);
        break;
        }
      case FILE_INBOX:
        {
        sprintf(pathname,"<ARCbbs$temp>.Inbox_%d",portnumber);
        break;
        }
      }
    }

  return(pathname);
  }

void setup_mailheader(mail_block *header,user_block *us,int area)
  {
  mail_area ma;

  read_area(area,&ma);
  memset(header,0,256);
  header->message_area=area;
  header->from_id=us->usernumber;
  strcpy(header->from,us->username);
  header->file_location=-1;
  header->reply_from=-1;
  header->date_entered=header->date_sent=time(NULL);
  header->fidofrom.zone=ouraddress.zone;
  header->fidofrom.net=ouraddress.net;
  header->fidofrom.node=ouraddress.node;
  header->fidofrom.point=ouraddress.point;
  if ((ma.areaflags&FLAG_AUTOINVISIBLE)!=0) header->flags|=MSG_SYSOP;
  }
  
void flag_initialise()
  {
  int a,b; FILE *f; char line[80];

  /* Read flag files */
  if ((f=fopen("<ARCbbs$MiscData>.Flags.Message","r"))==NULL) exit(0);
  b=0;
  do
    {
    if (!fgets(line,79,f)) break;
    if (sscanf(line,"%d %s",&a,msgflag[b]+1)==2) msgflag[b++][0]=a;
    }
  while(!feof(f));
  fclose(f); noof_msgflags=b;

  if ((f=fopen("<ARCbbs$MiscData>.Flags.File","r"))==NULL) exit(0);
  b=0;
  do
    {
    if (!fgets(line,79,f)) break;
    if (sscanf(line,"%d %s",&a,fileflag[b]+1)==2) fileflag[b++][0]=a;
    }
  while(!feof(f));
  fclose(f); noof_fileflags=b;

  if ((f=fopen("<ARCbbs$MiscData>.Flags.User","r"))==NULL) exit(0);
  b=0;
  do
    {
    if (!fgets(line,79,f)) break;
    if (sscanf(line,"%d %s",&a,userflag[b]+1)==2) userflag[b++][0]=a;
    }
  while(!feof(f));
  fclose(f); noof_userflags=b;
  }

int findflag_user(char *fname)
  {
  int a;

  for(a=0;a<noof_userflags;a++) if (strcmp(userflag[a]+1,fname)==0) return(userflag[a][0]);
  return(-1);
  }

int findflag_msg(char *fname)
  {
  int a;

  for(a=0;a<noof_msgflags;a++) if (strcmp(msgflag[a]+1,fname)==0) return(msgflag[a][0]);
  return(-1);
  }

int findflag_file(char *fname)
  {
  int a;

  for(a=0;a<noof_fileflags;a++) if (strcmp(fileflag[a]+1,fname)==0) return(fileflag[a][0]);
  return(-1);
  }

void get_item(char **pos,char *result)
  {
  char *p=*pos; int inquote=0;

  /* Skip spaces */
  while(*p==' ') p++;
  
  /* End of line? */
  if (*p==NULL)
    {
    *pos=p;
    *result=0;
    return;
    }

  /* Copy data until a ',' */
  do
    {
    switch(*result++=*p++)
      {
      case ',':
        if (inquote==0)
          {
          *(result-1)=0;
          *pos=p;
          return;
          }
        break;
      case NULL:
        *(result-1)=0;
        *pos=--p;
        return;
      case '\"':
        inquote=1-inquote;
        break;
      case '\\':
        *(result-1)=*p++;
        break;
      }
    }
  while(1);
  }
