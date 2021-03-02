/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Questionnaire processing                    <]
Current version   [> 00.08                                       <]
Version date      [> 04-November-1992                            <]
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
#include "parser.h"          /* Command parser */
#include "mail.h"            /* Mail format */
#include "portmisc.h"        /* Misc port commands */
#include "servcomm.h"        /* Misc server comm commands */
#include "mprintf.h"         /* Modem printf */
        
extern void mygets(char*,FILE*); 
extern user_block user;
extern char tempbuffer[256],*message_buffer;
extern FILE *bbs_file[10];

void questionnaire(char *qfile)
  {
  time_t now=time(NULL);
  char *m_ptr=message_buffer,*message_buffer2=message_buffer+8192,
       qsubject[61]; 
  int quser=0;

  if ((bbs_file[7]=fopen(qfile,"r"))==NULL)
    {
    mprintf(1,"{bfg r}Can't open question file!{std}\n");
    return;
    }
  
  /* First line is output data file */
  mygets(tempbuffer,bbs_file[7]);

  if (strncmp(tempbuffer,">>>",3)==0)
    {
    /* To a user */
    quser=atoi(tempbuffer+3);
    qsubject[0]=0;
    if (strchr(tempbuffer,' ')!=NULL)
      {
      strcpy(qsubject,strchr(tempbuffer,' ')+1);
      }
    }
  else
    {
    /* Open output file */
    if ((bbs_file[8]=fopen(tempbuffer,"a"))==NULL)
      {
      mprintf(1,"{bfg r}Can't open answer file! (%s){std}\n",tempbuffer);
      return;
      }
                
    /* Stick name of person answering questionnaire in file */
    fprintf(bbs_file[8],"\n\nName: %s (#%d)\nDate: %s\n\n",
            user.username,user.usernumber,cctime(&now));
    }
                   
  do
    {
    int qnr=0,ans;
    port_txstring("{bfg w}",1);

    do
      {                         
      getquest:
      /* Get question */
      mygets(tempbuffer,bbs_file[7]); 
      if (strncmp(tempbuffer,"#END",4)==0) goto theend;
      if (strncmp(tempbuffer,"#SAVE",5)==0)
        {
        mail_block header; user_block tempuser; int clr;

        for(clr=0;clr<255;clr++) *(clr+(char*)&header)=0;

        header.reply_from=-1; header.message_area=0;
        header.from_id=user.usernumber; /* So we can reply easily */
        strcpy(header.from,user.username);
        header.to_id=tempuser.usernumber=quser;
        get_user(&tempuser);
        if (tempuser.status!=0)
          {                          
          strcpy(header.to,tempuser.username);
          header.file_location=-1; header.file_length=0;

          /* Set date sent + expire 1 year afterwards */
          header.date_entered=header.date_sent=time(NULL);
          
          /* Set subject */
          strcpy(header.subject,qsubject);
                           
          header.message_length=(m_ptr-message_buffer);

          if (put_message(&header,message_buffer)!=OK)
            {
            mprintf(1,"\n\nCan't write questionnaire results to mail!\n");
            }
          }
        }
      if (strncmp(tempbuffer,"#GOTO ",6)==0)
        {      
        char label[40];

        /* Look for the goto label */
        strcpy(label,tempbuffer+6);
        fseek(bbs_file[7],0,SEEK_SET);
        do
          {
          mygets(tempbuffer,bbs_file[7]);
          if (strncmp(tempbuffer,"#LABEL ",7)==0)
            {
            if (strcmp(tempbuffer+7,label)==0) goto getquest;
            }
          }
        while(!feof(bbs_file[7]));
        mprintf(1,"{bfg r}Label %s not found!{std}\n",label);
        goto theend;
        }
      
      if (tempbuffer[0]!='#' && tempbuffer[0]!=' ')
        {
        port_txstring(tempbuffer,1); port_crlf();
        if (quser==0) 
          {
          fputs(tempbuffer,bbs_file[8]); fputc(10,bbs_file[8]);
          }
        }
      }
    while(tempbuffer[0]!=' ');
                           
    if (tempbuffer[1]=='>')
      {
      int maxlen=atoi(tempbuffer+2);              

      mprintf(1,"{bfg c}>");
      port_readline(tempbuffer,maxlen,0); port_crlf();
      if (quser)
        {
        m_ptr+=(sprintf(m_ptr," %s",tempbuffer)+1);
        }
      else
        {
        fputc(32,bbs_file[8]); fputs(tempbuffer,bbs_file[8]);
        fputc(10,bbs_file[8]);
        }
      }
    else
      {
      char *tptr;

      strcpy(message_buffer2+(80*(qnr++)),tempbuffer);
      if (tempbuffer[1]=='#')
        {
        mprintf(1," {fg c}[{fg y}%c{fg c}]{fg w}%s\n",qnr+64,strchr(tempbuffer+2,'#')+1);         
        }
      else
        {
        mprintf(1," {fg c}[{fg y}%c{fg c}]{fg w}%s\n",qnr+64,tempbuffer);         
        }

      do
        {          
        mygets(tempbuffer,bbs_file[7]);
        if (tempbuffer[0])
          {
          strcpy(message_buffer2+(80*(qnr++)),tempbuffer);
          if (tempbuffer[1]=='#')
            {
            mprintf(1," {fg c}[{fg y}%c{fg c}]{fg w}%s\n",qnr+64,strchr(tempbuffer+2,'#')+1);         
            }
          else
            {
            mprintf(1," {fg c}[{fg y}%c{fg c}]{fg w}%s\n",qnr+64,tempbuffer);         
            }
          }
        }
      while(tempbuffer[0]);
    
      /* Get answer */
      mprintf(1,"{bfg g}Select : {fg w}");
      do
        {
        ans=tolower(port_get());
        }
      while(ans<'a' || ans>('a'+qnr-1));
      port_txw(ans); port_crlf(); port_crlf();                      
      
      tptr=message_buffer2+(80*(ans-'a'));

      if (tptr[1]=='#')
        {   
        if (quser)
          {
          m_ptr+=(sprintf(m_ptr,"%s",strchr(tptr+2,'#')+1)+1);
          }
        else
          {
          fputs(strchr(tptr+2,'#')+1,bbs_file[8]);
          fputc(10,bbs_file[8]);
          }

        /* Terminate command string */     
        *strchr(tptr+2,'#')=0;

        /* Follow command if it's a goto */
        if (strncmp(tptr+1,"#GOTO ",6)==0)
          {      
          char label[40];
                         
          qnr=0;
          /* Look for the goto label */
          strcpy(label,tptr+7);
          fseek(bbs_file[7],0,SEEK_SET);
          do
            {
            mygets(tempbuffer,bbs_file[7]);
            if (strncmp(tempbuffer,"#LABEL ",7)==0)
              {
              if (strcmp(tempbuffer+7,label)==0) goto getquest;
              }
            }
          while(!feof(bbs_file[7]));
          mprintf(1,"{bfg r}Label %s not found!{std}\n",label);
          goto theend;
          }
        }
      else
        {   
        if (quser)
          {
          m_ptr+=(sprintf(m_ptr,"%s",tptr)+1);
          }
        else
          {
          fputs(tptr,bbs_file[8]);
          fputc(10,bbs_file[8]);
          }
        }
      }
    }
  while(!feof(bbs_file[7]));
                           
  theend:

  fclose(bbs_file[7]);
  if (quser==0) fclose(bbs_file[8]);
  }
