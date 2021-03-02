/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Private mail checkmail                      <]
Current version   [> 00.07                                       <]
Version date      [> 09-November-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/90/91 by <]
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
                 
extern void display_message(mail_block*,mail_block*);
   
void do_checkmail(int usern,int sysop)
  {
  user_block us; mail_block header,fileheader; int number;

  us.usernumber=usern;

  /* Get user just in case s/he's got new mail */
  get_user(&us);

  if (!us.mailcount)
    {
    port_txstring("\n{bfg c}You have no mail waiting for you.{std}\n",1);
    }
  else
    {
    int a,current=us.mailpointer_start,nr=us.mailcount,mailcount=us.mailcount;
    char buffer[3]; int pointers[100],flags[100];
                   
    if (current==-1)
      {
      us.mailcount=0;
      put_user(&us);
      port_txstring("\n{bfg c}You have no mail waiting for you.{std}\n",1);
      return;
      } 

    for(a=0;a<100;a++) pointers[a]=-1;

    for(a=0;a<mailcount;a++)
      {
      pointers[a]=header.message_number=current;
      get_messageh(&header);
      current=header.message_forward;           
      flags[a]=header.flags;
      }

    do
      {
      int o_number;

      port_txstring("{bfgbg cb} # From                           Date                    {std}\n",1);
      if (user.termtype!=TER_ANSI) port_txstring("{bfg b}-- ------------------------------ ------------------------\n",1);

      for(a=0;a<nr && port_rxbuffer()==0;a++)
        {
        if (pointers[a]!=-1)
          {
          header.message_number=pointers[a];
          get_messageh(&header);
          mprintf(1,"{bfg %c}%2d %-30s %s\n",(flags[a]&MSG_EXPRESS)?'r':'w',a,
                            header.from,cctime(&header.date_sent));
          }
        }
      if (port_rxbuffer()) port_txclear();
      else if (user.termtype!=TER_ANSI) port_txstring("{bfg b}-- ------------------------------ ------------------------\n",1);
 
      getnumberagain:
      port_txstring("{bfg c}Number to read (CR aborts) : ",1);
      port_readline(buffer,2,0); port_crlf();
      if (!buffer[0]) goto endmail;
      if (atoi(buffer)<0) goto getnumberagain;
      number=pointers[o_number=atoi(buffer)];
      if (number==-1)
        {
        goto getnumberagain;
        }
      else
        {
        int gotopt; char *menu;

        header.message_number=number;
        get_messageh(&header);
        display_message(&header,&fileheader);

        /* Recorded delivery? */
        if (header.flags&MSG_CONFIRM && sysop==0)
          {
          mail_block header2;

          header.flags^=MSG_CONFIRM;
          put_messageh(&header);

          memset(&header2,0,255);
          header2.message_area=0; /* Private mail */
          header2.from_id=us.usernumber;
          strcpy(header2.from,us.username);
          header2.to_id=header.from_id;
          strcpy(header2.to,header.from);
          header2.file_length=0; header2.file_location=-1;
          header2.date_entered=header2.date_sent=time(NULL);
          sprintf(tempbuffer2,"Message #%d received & read",
                                  header.message_number);
          header2.message_length=strlen(tempbuffer2)+1;
          strcpy(header2.subject,header.subject);

          if (put_message(&header2,tempbuffer2)==OK)
            {
            mprintf(1,"{std}Recorded delivery message sent\n");
            }
          }  
                      
        menu=(header.file_location>=0)?"<ARCbbs$prompts>.mail_fmenu":
                                       "<ARCbbs$prompts>.mail_menu";

        do
          {
          bbs_sendfile(menu,0,0);
          gotopt=0;

          switch(toupper(port_get()))
            {
            /* File menu */
            case 'F':
              {
              int quit=0;

              if (header.file_location>=0)
                {
                port_txstring("{bfg g}File menu{std}\n",1);
  
                /* Read menu */
                do
                  {
                  int gotopt2;
                  bbs_sendfile("<ARCbbs$prompts>.read_fm",0,0);
  
                  do
                    {
                    gotopt2=1;
                    switch(toupper(port_get()))
                      {
                      /* Queue file */
                      case 'Q':
                        {
                        port_txstring("{bfg g}Queue file{std}\n",1);
                        file_download(fileheader.file_location,
                                      fileheader.file_length,
                                      fileheader.to,0,1);
                        break;
                        }
                      /* Download file */
                      case 'D':
                        {
                        port_txstring("{bfg g}Download file{std}\n",1);
                        file_download(fileheader.file_location,
                                      fileheader.file_length,
                                      fileheader.to,1,1);
                        break;
                        }
                      /* Back to read menu */
                      case 'R':
                        {
                        port_txstring("{bfg g}Return to read menu{std}\n",1);
                        quit=1;
                        break;
                        }
                      default:
                        {
                        gotopt2=0;
                        break;
                        }
                      }
                    }
                  while(!gotopt2);
                  }
                while(!quit);
                }
              break;
              }
            /* Reply */
            case 'R':
              {
              port_txstring("{bfg g}Reply\n",1);

              if (header.flags&MSG_FIDO)
                {
                message_fidowrite(&header.fidofrom,header.from,header.subject,header.message_number);
                }
              else
                {
                mail_block header2;

                memset(&header2,0,255);
                header2.reply_from=header.message_number;
                header2.message_area=0; /* Private mail */
                header2.from_id=us.usernumber;
                strcpy(header2.from,us.username);
                header2.to_id=header.from_id;
                strcpy(header2.to,header.from);
                header2.file_length=0; header2.file_location=-1;
                header2.date_entered=header2.date_sent=time(NULL);
                mprintf(1,"\n{bfg c}Area   : {fg w}Private mail\n{fg c}Date   : {fg w}%s\n{fg c}From   : {fg w}",cctime(&header2.date_sent));
                mprintf(16+1,"%s (#%d)\n",header2.from,header2.from_id);
                mprintf(1,"{fg c}To     : {fg w}");
                mprintf(1,"%s (#%d)\n",header2.to,header2.to_id);
                strcpy(header2.subject,header.subject);

                if (entertext(&header2,0))
                  {
                  mprintf(1,"\n{bfg g}Adding message (%d bytes) as message #{fg w}",header2.message_length);
                  if (put_message(&header2,message_buffer)==OK)
                    {
                    mprintf(1,"%d{std}\n",savednr);
                    }
                  }
                }
              break;
              }
            /* Delete */
            case 'D':
              {
              port_txstring("{bfg g}Delete\n",1);
              if (port_yesno("{bfg r}Are you sure? {std}"))
                {
                delete_message(number);
                mailcount--;
                pointers[o_number]=-1;
                }      
              gotopt=1;
              break;
              }
            /* Exit read menu */
            case 13:
            case 10:
            case ' ':
            case 'A':
              {
              gotopt=1;
              break;
              }
            /* Throw to... */
            case 'T':
              {
              mail_block header2; char userbuffer[31],*ptr1,*ptr2;
              user_block userto;
  
              port_txstring("{bfg g}Throw message...\n",1);
  
              getnameagain1:
              port_txstring("\015{fg c}To     : ",1);
              port_readline(ptr1=userbuffer,30,0);
  
              /* Strip leading spaces */
              while(*ptr1==' ') ptr1++;
              if (!*ptr1)
                {
                port_txstring("\015{bfg c}To     : {fg r}Forward aborted{std}\n",1);
                break;  /* Abort with blank username */
                }
             
              /* Convert string to uppercase */
              ptr2=ptr1;
              while(*ptr2)
                {
                *ptr2=toupper(*ptr2);
                ptr2++;
                }
  
              /* If numeric first digit, get as a usernumber */
              if (*ptr1>='0' && *ptr1<='9') 
                {
                usernumber=atoi(ptr1);
                }
              else
                {
                /* Text is username, do hashing, etc */
                usernumber=hash_find(ptr1);
          
                /* Non-existant? */
                if (usernumber==-1) goto getnameagain1;
                }
  
              userto.usernumber=usernumber;
              get_user(&userto);
              if (userto.status==0) goto getnameagain1;
  
              mprintf(1,"\015{fg c}To     : {bfgbg wb}%s{std}\n",userto.username);
      
              /* Get the message again to ensure legal message text */
              get_message(&header,message_buffer);

              memset(&header2,0,255);
              header2.message_area=0; /* Private mail */
              header2.from_id=header.from_id;
              strcpy(header2.from,header.from);
              header2.to_id=usernumber;
              strcpy(header2.to,userto.username);
              header2.file_length=0; header2.file_location=-1;
              header2.date_entered=header2.date_sent=time(NULL);
              sprintf(header2.subject,"Forwarded by %s (#%d)",
                                      user.username,user.usernumber);
              header2.message_length=header.message_length;
              mprintf(1,"{bfg g}Throwing...");
              put_message(&header2,message_buffer);
                {
                mprintf(1,"thrown (Message #{fg w}%d{fg g}){std}\n",savednr);
                }      
              break;
              }
            }
          }
        while(gotopt==0);
        port_crlf();
        }
      }
    while(mailcount);
    port_txstring("\n{bfg c}You have no more mail waiting for you.{std}\n",1);
    }
  endmail:
  return;
  }
