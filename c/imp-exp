/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> User message import/export module           <]
Current version   [> 00.25                                       <]
Version date      [> 12-November-1992                            <]
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

#include "version.h"
#include "miscbbs.h"
#include "bbs.h"
                          
extern int can_read(mail_area*),can_write(mail_area*),lastpoll;
extern mail_block *lastmsg;

void safeget(char *line,FILE *f)
  {
  int a,c=0;

  while((a=fgetc(f))!=EOF)
    {
    if (a==10) break;

    if (a>31)
      {
      line[c++]=a;
      if (c==250) break;
      }
    }

  line[c]=0;
  }

void message_upload(char *upname,int local)
  { 
  char line[256],type[10],upname2[64];
  int a,public_sent=0,private_sent=0,voted=0;
                         
  /* Rename it so it works with ARC */
  if (local==0)
    {
    strcpy(upname2,upname);
    upname2[strlen(upname2)-5]='-';
    rename(upname,upname2);
    upname=upname2;       
    }

  /* Open the uploaded/dearced file */
  if ((bbs_file[6]=fopen(upname,"r"))==NULL)
    {
    mprintf(1,"{bfg r}Can't open message batch file{std}\n");
    return;
    }
                               
  /* Is it an ARCfile ? */
  if (fgetc(bbs_file[6])==0x1a)
    {
    char inarcname[14],newname[8],reopenname[20];
      
    /* Read the file's name */  
    fgetc(bbs_file[6]); /* Throw away compression type */
    for(a=0;a<13;a++) inarcname[a+1]=fgetc(bbs_file[6]);
    inarcname[0]='/';                               

    /* Close the file and get ARC to rename it */
    fclose(bbs_file[6]); bbs_file[6]=0;
    sprintf(newname,"/Msg_%d",portnumber);
    do_arc(upname,inarcname,"-R",newname+1);
                    
    sprintf(reopenname,"<ARCbbs$temp>.%s",newname+1);
    remove(reopenname);

    /* Dearc the file */
    mprintf(1,"{bfg g}Removing file '%s' from archive...",inarcname+1);                  
    do_arc(upname,newname,"-x","<ARCbbs$temp>");
    mprintf(1,"done             {std}\n");                   

    /* Remove the ARCfile */
    remove(upname);

    /* Open the de-arced file */ 
    if ((bbs_file[6]=fopen(upname=reopenname,"r"))==NULL)
      {
      mprintf(1,"{bfg r}Can't open message batch file{std}\n");
      return;
      }
    }    
                                            
  mprintf(1,"{bfg g}Processing uploaded batch...{std}\n");
  fseek(bbs_file[6],0,SEEK_SET);                               

  while(!feof(bbs_file[6]))
    {
    /* Keep the bbs going */
    window_poll();

    /* Get a line from the file */
    safeget(line,bbs_file[6]);
    
    /* Check the message format */
    if (sscanf(line,"<%[^>]s>",type)==1)
      {                       
      int itype=0;

      /* Check the opcode */
      if (strcmp(type,"vote")   ==0) itype=1;
      if (strcmp(type,"mail")   ==0) itype=2;
      if (strcmp(type,"msg")    ==0) itype=3;
      if (strcmp(type,"netmail")==0) itype=4;

      switch(itype)
        {
        case 0:
          {
          mprintf(1,"{bfg r}Unrecognised object type : <%s>{std}\n",type);
          break;
          }
        case 1: /* vote */
          {                                 
          int msgnr; char vote[10];

          if (sscanf(line,"%*s #%d %s",&msgnr,vote)!=2)
            {
            mprintf(1,"{bfg r}Badly formed 'vote' object : %s{std}\n",line);
            goto closeup;
            }
          else
            { 
            /* Check msg number is in range */
            if (msgnr<=mod_getlastlookup() && msgnr>=0)
              {               
              mail_block header;

              /* Get header */ 
              header.message_number=msgnr;
              get_messageh(&header);
              if ((header.flags&MSG_VOTE)==0)
                {
                mprintf(1,"{bfg r}Message is not a voting message{std}\n",line);
                goto closeup;
                }
              else
                {
                /* Do the voting */
                switch(tolower(vote[0]))
                  {
                  case 'y':
                  case 'f':
                  case '+':
                    {
                    header.votes_for++;
                    break;
                    }
                  case 'n':
                  case 'a':
                  case '-':
                    {
                    header.votes_against++;
                    break;
                    }
                  }

                /* Rewrite header */
                put_messageh(&header);
                voted++;
                }
              }
            }
          break;
          }
        case 2: /* mail */
          {                              
          char userbuffer[31],*msgtxt=message_buffer,*pt;
          int clr,usernumber,reverse,trunc=0;
          mail_block header; user_block tempuser;
          for(clr=0;clr<256;clr++) *(clr+(char*)&header)=0;

          if ((pt=strstr(line,"->"))==0)
            {
            mprintf(1,"{bfg r}Badly formed 'mail' object : %s{std}\n",line);
            goto closeup;
            }
                            
          pt+=2; pt[30]=0;
          strcpy(userbuffer,pt);

          header.reply_from=-1; header.message_area=0;
          header.from_id=user.usernumber;
          strcpy(header.from,user.username);
          header.file_location=-1; header.file_length=0;
          header.fidofrom.zone=ouraddress.zone;
          header.fidofrom.net=ouraddress.net;
          header.fidofrom.node=ouraddress.node;
          header.fidofrom.point=ouraddress.point;

          /* Set date sent + expire 1 year afterwards */
          header.date_entered=header.date_sent=time(NULL);

          /* If numeric first digit, get as a usernumber */
          if (isdigit(userbuffer[0])) 
            {
            usernumber=atoi(userbuffer);
            }
          else
            {                          
            /* Text is username, do hashing, etc */
            usernumber=hash_find(userbuffer);

            /* Non-existant? */
            if (usernumber==-1)
              {
              mprintf(1,"{bfg r}Bad username : %s{std}\n",userbuffer);
              goto closeup;
              }
            }
          header.to_id=usernumber;
          if (usernumber!=-1)
            {
            tempuser.usernumber=usernumber;
            get_user(&tempuser);
            strcpy(header.to,tempuser.username);
            }
          else
            {
            mprintf(1,"{bfg r}Bad username : %s{std}\n",userbuffer);
            goto closeup;
            }
                 
          /* Get subject */         
          safeget(line,bbs_file[6]);  
          line[60]=0; strcpy(header.subject,line); 

          /* Read in the text */
          do
            {
            /* Get a line & store it */
            reverse=ftell(bbs_file[6]);
            safeget(pt=msgtxt,bbs_file[6]);
            if (strncmp(pt,"<mail>",6)==0 || strncmp(pt,"<msg>",5)==0 ||
                strncmp(pt,"<vote>",6)==0)
              {
              fseek(bbs_file[6],reverse,SEEK_SET);
              break;
              }

            if (*msgtxt) msgtxt+=(strlen(msgtxt)+1);
            if ((msgtxt-message_buffer)>15360) { msgtxt=pt; trunc=1; }
            }
          while(feof(bbs_file[6])==0);
          
          if (trunc) mprintf(1,"{bfg r}Warning! Message too long, it has been truncated\n");

          header.message_length=(msgtxt-message_buffer);

          put_message(&header,message_buffer);  
          private_sent++;
          break;
          }
        case 3: /* msg */
          {                              
          char userbuffer[31],*msgtxt=message_buffer,*pt;
          int clr,arean,usernumber,msgnr,reverse,trunc=0;
          mail_block header; mail_area area; user_block tempuser;
          for(clr=0;clr<256;clr++) *(clr+(char*)&header)=0;
          userbuffer[0]=0;  

          if (sscanf(line,"%*s @%d #%d",&arean,&msgnr)!=2)
            {
            msgnr=-1;
            if (sscanf(line,"%*s @%d ->",&arean)!=1)
              {
              mprintf(1,"{bfg r}Badly formed 'msg' object : %s{std}\n",line);
              goto closeup;
              }
            }              

          if (arean<1 || arean>=mod_getlastarea())
            {
            mprintf(1,"{bfg r}Bad message area : %s{std}\n",line);
            goto closeup;
            } 
                                 
          /* Read in this message area */
          read_area(arean,&area);   

          /* Check area type */
          if ((area.areaflags&FLAG_FILEAREA)!=0)
            {
            mprintf(1,"{bfg r}Area %d (%s) is not a message area{std}\n",arean,area.name);
            goto closeup;
            }

          /* Check area access */
          if (can_write(&area)==0)
            {
            mprintf(1,"{bfg r}Area %d (%s) is NOT writable by you{std}\n",arean,area.name);
            goto closeup;
            }

          /* Check msg number is in range */
          if (msgnr>area.last_message || msgnr<area.first_message) msgnr=-1;
          else
            {
            mail_block testarea;

            testarea.message_number=msgnr;
            get_messageh(&testarea);
            if (testarea.message_area!=arean) msgnr=-1;
            else strcpy(userbuffer,testarea.from);
            }                                     
                        
          if ((pt=strstr(line,"->"))==0 && userbuffer[0]==0)
            {
            mprintf(1,"{bfg r}Can't find who message is to! : %s{std}\n",line);
            goto closeup;
            }
          
          if (userbuffer[0]==0)
            {                  
            pt+=2; pt[30]=0;
            strcpy(userbuffer,pt);
            }

          header.reply_from=msgnr; header.message_area=arean;
          header.from_id=user.usernumber;
          header.file_location=-1; header.file_length=0;
          header.fidofrom.zone=ouraddress.zone;
          header.fidofrom.net=ouraddress.net;
          header.fidofrom.node=ouraddress.node;
          header.fidofrom.point=ouraddress.point;

          /* Set date sent + expire 1 year afterwards */
          header.date_entered=header.date_sent=time(NULL);

          if ((area.areaflags&FLAG_ECHOMAIL)==0)
            {
            strcpy(header.from,user.username);
            /* If numeric first digit, get as a usernumber */
            if (isdigit(userbuffer[0])) 
              {
              usernumber=atoi(userbuffer);
              }
            else
              {                          
              userbuffer[0]=tolower(userbuffer[0]);
              userbuffer[1]=tolower(userbuffer[1]);
              userbuffer[2]=tolower(userbuffer[2]);

              if (strcmp(userbuffer,"all")!=0)
                {
                /* Text is username, do hashing, etc */
                usernumber=hash_find(userbuffer);

                /* Non-existant? */
                if (usernumber==-1)
                  {
                  mprintf(1,"{bfg r}Bad username : %s{std}\n",userbuffer);
                  break;
                  }
                }
              else
                {
                usernumber=-1;
                }
              }
            header.to_id=usernumber;
            if (usernumber!=-1)
              {
              tempuser.usernumber=usernumber;
              get_user(&tempuser);
              strcpy(header.to,tempuser.username);
              }
            else
              {
              strcpy(header.to,"All");
              }
            }
          else
            {          
            int cap=1; char *ptr1=userbuffer;
            
            strcpy(header.from,fido_rules(user.username));

            while(*ptr1)
              {
              if (cap) { *ptr1=toupper(*ptr1); cap=0; }
              else *ptr1=tolower(*ptr1);
              if (*ptr1==32) cap=1;
              ptr1++;
              }   

            strcpy(header.to,userbuffer);
    
            header.flags|=MSG_FIDO;
            header.to_id=0;
            }
                 
          /* Get subject */
          safeget(line,bbs_file[6]);  
          line[60]=0; strcpy(header.subject,line); 

          /* Read in the text */
          do
            {
            /* Get a line & store it */
            reverse=ftell(bbs_file[6]);
            safeget(pt=msgtxt,bbs_file[6]);
            if (strncmp(pt,"<mail>",6)==0 || strncmp(pt,"<msg>",5)==0 ||
                strncmp(pt,"<vote>",6)==0)
              {
              fseek(bbs_file[6],reverse,SEEK_SET);
              break;
              }

            if (*msgtxt) msgtxt+=(strlen(msgtxt)+1);
            if ((msgtxt-message_buffer)>15360) { msgtxt=pt; trunc=1; }
            }
          while(feof(bbs_file[6])==0);

          if (trunc) mprintf(1,"{bfg r}Warning! Message too long, it has been truncated\n");

          header.message_length=(msgtxt-message_buffer);

          if (area.areaflags&FLAG_ECHOMAIL)
            {
            header.message_length+=sprintf(message_buffer+header.message_length,
                                         "--- %s",tearline)+1;
            if (ouraddress.point!=0)
              {
              header.message_length+=sprintf(message_buffer+header.message_length,
                                             " * Origin: %s (%d:%d/%d.%d)",
                                             ourname,ouraddress.zone,ouraddress.net,
                                             ouraddress.node,ouraddress.point)+1;
              }
            else
              {
              header.message_length+=sprintf(message_buffer+header.message_length,
                                             " * Origin: %s (%d:%d/%d)",
                                             ourname,ouraddress.zone,ouraddress.net,
                                             ouraddress.node)+1;
              }

            header.message_length+=sprintf(message_buffer+header.message_length,
                                           "SEEN-BY: %d/%d",
                                           ouraddress.net,ouraddress.node)+1;
           }

          put_message(&header,message_buffer);
          public_sent++;
          break;
          }
        }
      }
    }     
  closeup:                      
  fclose(bbs_file[6]); bbs_file[6]=0;

  mprintf(1,"{bfg g}Processing complete:\n%2d message(s) voted on\n%2d items of private mail sent\n%2d public messages sent{std}\n",voted,private_sent,public_sent);

  if (local==0)
    {
    sprintf(line,"Access %s wr/",upname);
    system(line);   /* Make sure it isn't locked */
    remove(upname); /* Delete this message packet */
    }
  }

void message_file(int arean,mail_area *area,FILE *scratch_file,int theworks)
  {
  int current_msg,t,a,pointer=0,abort=0; mail_block header,fileheader;
  clock_t lastmsg=clock();
  char *mconv,*mend,*mdest,mbuffr[30];  
     
  window_poll();

  fprintf(scratch_file,"::: Area #%d (%s)\n",arean,area->name);
 
  if (arean)
    {
    current_msg=user.m_message; readnew=1;
    if (current_msg==0) current_msg=area->last_message;
    if (current_msg<area->first_message) current_msg=area->first_message;
    if (current_msg>area->last_message) current_msg=-1;
  
    if (current_msg==-1 || area->count==0) return;
   
    do
      {
      header.message_number=current_msg;
      get_messageh(&header);
  
      if (header.message_area==arean && header.status==1)
        {
        if ((user.f_user&USER_SYSOP)!=0 || (header.flags&MSG_SYSOP)==0)
          {       
          get_message(&header,message_buffer+128);
  
          fprintf(scratch_file,"Message: #%d (Read %d %s, has %d %s, %d bytes)\n",current_msg,header.noof_reads,header.noof_reads==1?"time":"times",header.noof_replies,header.noof_replies==1?"reply":"replies",header.message_length);
          fprintf(scratch_file,"Date   : %-24s\n",cctime(&header.date_sent));
          if ((area->areaflags&FLAG_ECHOMAIL)==0)
            {
            fprintf(scratch_file,"From   : %s (#%d)\n",header.from,header.from_id);
            }
          else
            {
            fprintf(scratch_file,"From   : %s\n",header.from);
            }
  
          if (header.to_id!=-1 && (area->areaflags&FLAG_ECHOMAIL)==0)
            {
            fprintf(scratch_file,"To     : %s (#%d)\n",header.to,header.to_id);
            }
          else
            {
            fprintf(scratch_file,"To     : %s\n",header.to);
            }
  
          if (header.reply_from==-1)
            {
            fprintf(scratch_file,"Subject: %s\n\n",header.subject);
            }
          else
            {
            fprintf(scratch_file,"Subject: %s (reply to #%d)\n\n",header.subject,header.reply_from);
            }
                                                                      
          if (header.message_length==0) { abort=1; message_buffer[0]=0; }
  
          /* File message */
          mdest=message_buffer; mconv=message_buffer+128; mend=mconv+header.message_length;
          do
            {                       
            if (mconv[0]==1 && theworks==0)
              {
              while(*mconv++);
              continue;
              }
  
            if (mconv[0]==' ' && mconv[1]=='*' && theworks==0)
              {
              if (strncmp(mconv," * Origin:",10)==0 && theworks==0)
                {
                while(*mconv) *mdest++=*mconv++;
                *mdest++=10; mconv++;
                break;
                }
              }
              
            if ((t=strlen(mconv))>79)
              {
              /* Wordwrap */
              t=78;
              while(mconv[t]!=' ' && mconv[t]!='-' && mconv[t]!='.' && mconv[t]!=',' && t>65) t--;
              for(a=0;a<(t+1);a++) *mdest++=mconv[a];
              mconv+=t;
              }
            else while(*mconv) *mdest++=*mconv++;
  
            /* Remove any trailing spaces */
            if (*(mdest-1)==' ') mdest--;
            *mdest++=10; mconv++;
            }
          while(mconv<mend);
                                                             
          /* Write all as one block - it's quicker that way */
          *mdest++=10;
          fwrite(message_buffer,mdest-message_buffer,1,scratch_file);
                                         
          if ((pointer=header.noof_replies)!=0)
            {
            fprintf(scratch_file,"Latest repl%s: #%d",pointer==1?"y":"ies",header.replies[0]);
            pointer--;
            if (pointer)
              {
              int a;
              if (pointer>7) pointer=7;
              for(a=1;a<=pointer;a++)
                {
                fprintf(scratch_file,",#%d",header.replies[a]);
                }
              }
            fputc(10,scratch_file);
            }
  
          /* Voting display */
          if (header.flags&MSG_VOTE)
            {
            fprintf(scratch_file,"Votes FOR: %d, Votes AGAINST: %d.\n",header.votes_for,header.votes_against);
            }
  
          /* If linked to a file */
          if (header.file_location>=0)
            {
            fileheader.message_number=header.file_location;
            get_messageh(&fileheader);
  
            fprintf(scratch_file,"Linked to file '%s' (length %d bytes)\n",
                     fileheader.to,fileheader.file_length);
            }
          
          noof_filed++;
          }
        current_msg=header.message_forward;
        }
      else
        {
        current_msg+=1;
        } 
  
      if ((clock()-lastpoll)>=MAXIMUM_POLL)
        {
        window_poll();
        if ((lastpoll-lastmsg)>=200)
          {
          sprintf(mbuffr,"\015Filed %4d messages",noof_filed);
          mprintf(1,"%s",mbuffr);
          lastmsg=clock();
          }
        }
      }
    while(current_msg>=area->first_message && current_msg<=area->last_message &&
          current_msg!=-1);
    }
  else
    {                                                  
    /* Check for any private mail waiting */
    if ((current_msg=user.mailpointer_start)<0) return;
   
    do
      {
      header.message_number=current_msg;
      get_message(&header,message_buffer+128);
      
      if ((header.flags&MSG_DOWNLOADED)==0)
        {
        header.flags|=MSG_DOWNLOADED;
        put_messagehi(&header);

        fprintf(scratch_file,"Message: #%d (Read %d %s, has %d %s, %d bytes)\n",
                current_msg,header.noof_reads,header.noof_reads==1?"time":"times",header.noof_replies,
                header.noof_replies==1?"reply":"replies",header.message_length);
        fprintf(scratch_file,"Date   : %-24s\n",cctime(&header.date_sent));
        if ((header.flags&MSG_FIDO)==0)
          {
          fprintf(scratch_file,"From   : %s (#%d)\n",header.from,header.from_id);
          }
        else
          {
          fprintf(scratch_file,"%s of %d:%d/%d.%d",header.from,
                  header.fidofrom.zone,header.fidofrom.net,header.fidofrom.node,header.fidofrom.point);
          }
    
        fprintf(scratch_file,"To     : %s (#%d)\n",header.to,header.to_id);
        fprintf(scratch_file,"Subject: %s\n\n",header.subject);
                                                                       
        if (header.message_length==0) { abort=1; message_buffer[0]=0; }
    
        /* File message */
        mdest=message_buffer; mconv=message_buffer+128; mend=mconv+header.message_length;
        do
          {                       
          if (mconv[0]==1 && theworks==0)
            {
            while(*mconv++);
            continue;
            }
    
          if (mconv[0]==' ' && mconv[1]=='*')
            {
            if (strncmp(mconv," * Origin:",10)==0 && theworks==0)
              {
              while(*mconv) *mdest++=*mconv++;
              *mdest++=10; mconv++;
              break;
              }
            }
                
          if ((t=strlen(mconv))>79)
            {
            /* Wordwrap */
            t=78;
            while(mconv[t]!=' ' && mconv[t]!='-' && mconv[t]!='.' && mconv[t]!=',' && t>65) t--;
            for(a=0;a<(t+1);a++) *mdest++=mconv[a];
            mconv+=t;
            }
          else while(*mconv) *mdest++=*mconv++;
    
          /* Remove any trailing spaces */
          if (*(mdest-1)==' ') mdest--;
          *mdest++=10; mconv++;
          }
        while(mconv<mend);
                                                               
        /* Write all as one block - it's quicker that way */
        *mdest++=10;
        fwrite(message_buffer,mdest-message_buffer,1,scratch_file);
    
        /* If linked to a file */
        if (header.file_location>=0)
          {
          fileheader.message_number=header.file_location;
          get_messageh(&fileheader);
    
          fprintf(scratch_file,"Linked to file '%s' (length %d bytes)\n",
                   fileheader.to,fileheader.file_length);
          }

        noof_filed++;
        }                          

      current_msg=header.message_forward;

      if ((clock()-lastpoll)>=MAXIMUM_POLL)
        {
        window_poll();
        if ((lastpoll-lastmsg)>=200)
          {
          sprintf(mbuffr,"\015Filed %4d messages",noof_filed);
          mprintf(1,"%s",mbuffr);
          lastmsg=clock();
          }
        }
      }
    while(current_msg>=0);
    }

  sprintf(mbuffr,"\015Filed %4d messages",noof_filed);
  mprintf(1,"%s",mbuffr);
  }

BOOL message_save(char *toleaf,void *handle)
  {
  FILE *a; char *p,*e;

  if ((a=fopen(toleaf,"w"))!=NULL)
    {
    if (lastmsg->message_area)
      {
      /* Write header */
      fprintf(a,"Message: #%d (Read %d %s, has %d %s, %d bytes)\n",
              lastmsg->message_number,lastmsg->noof_reads,lastmsg->noof_reads==1?"time":"times",
              lastmsg->noof_replies,lastmsg->noof_replies==1?"reply":"replies",lastmsg->message_length);
      fprintf(a,"Date   : %-24s\n",cctime(&lastmsg->date_sent));
      if ((lastmsg->flags&MSG_FIDO)==0) fprintf(a,"From   : %s (#%d)\n",lastmsg->from,lastmsg->from_id);
      else fprintf(a,"From   : %s\n",lastmsg->from);
  
      if (lastmsg->to_id!=-1 && (lastmsg->flags&MSG_FIDO)==0)
        {
        fprintf(a,"To     : %s (#%d)\n",lastmsg->to,lastmsg->to_id);
        }
      else
        {
        fprintf(a,"To     : %s\n",lastmsg->to);
        }
  
      if (lastmsg->reply_from==-1)
        {
        fprintf(a,"Subject: %s\n\n",lastmsg->subject);
        }
      else
        {
        fprintf(a,"Subject: %s (reply to #%d)\n\n",lastmsg->subject,lastmsg->reply_from);
        }
      }
    else
      {
      fprintf(a,"Message: #%d (Read %d %s, has %d %s, %d bytes)\n",
              lastmsg->message_number,lastmsg->noof_reads,lastmsg->noof_reads==1?"time":"times",
              lastmsg->noof_replies,lastmsg->noof_replies==1?"reply":"replies",lastmsg->message_length);
      fprintf(a,"Date   : %-24s\n",cctime(&lastmsg->date_sent));
      if ((lastmsg->flags&MSG_FIDO)==0)
        {
        fprintf(a,"From   : %s (#%d)\n",lastmsg->from,lastmsg->from_id);
        }
      else
        {
        fprintf(a,"%s of %d:%d/%d.%d",lastmsg->from,
                lastmsg->fidofrom.zone,lastmsg->fidofrom.net,lastmsg->fidofrom.node,lastmsg->fidofrom.point);
        }
  
      fprintf(a,"To     : %s (#%d)\n",lastmsg->to,lastmsg->to_id);
      fprintf(a,"Subject: %s\n\n",lastmsg->subject);
      }

    e=(p=message_buffer)+lastmsg->message_length;
   
    while(p<e)
      {
      if (*p) fputc(*p++,a);
      else { fputc(10,a); p++; }
      }
  
    fclose(a);
    }          

  return(TRUE);
  }

int getmsgnr()
  {
  return(lastmsg->message_number);
  }
