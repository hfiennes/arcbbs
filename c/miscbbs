/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Misc BBS routines                           <]
Current version   [> 00.99                                       <]
Version date      [> 19-November-1992                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/
  
int noseenbys=1;

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
#include "version.h"

extern void message_upload(char*,int);
extern char menu_prompt[];

int is_super(mail_area *a)
  {
  if (user.f_user&USER_SYSOP) return(1);
  if (user.usernumber==a->editor_id) return(1);
  return(0); 
  }

int read_time(void)
  {
  os_regset r;
  os_swi(0x42,&r); /* OS_ReadMonotonicTime */
  return(r.r[0]);
  }

void do_cli(int sysop)
  {
  char command[80];
  int quit=0;

  port_crlf();

  if (sysop)
    {
    strcpy(menu_prompt,"{bfg w}ARCbbs-Sysop>{fg c}");
    }
  else
    {
    strcpy(menu_prompt,"{bfg w}ARCbbs>{fg c}");
    }

  do
    {
    port_txstring(menu_prompt,1);
    port_readline(command,79,NOBOX);
    port_crlf();

    if (strcmp(command,"QUIT")==0 || strcmp(command,"quit")==0 ||
        strcmp(command,"EXIT")==0 || strcmp(command,"exit")==0) quit=1;

    if (*command && quit==0) do_command(command,sysop);
    }
  while(!quit);
  port_crlf();
  }        

void display_message(mail_block *header,mail_block *fileheader)
  {
  mail_area area; char *ptr,*eom; int a,ll,abort=0,nonstop=(user.f_user&USER_NOMORE);
  int headersize=((header->message_area==0)?4:5),line,scrlen,
      dynamic=((user.termtype==TER_ANSI || user.termtype==TER_VT100) && (user.f_user&USER_NOCLEAR)==0);
  char flags[40];

  scrlen=dynamic?(user.pagelen-headersize-1):user.pagelen;
  if (scrlen<6) scrlen=6;
  line=(user.pagelen==scrlen)?headersize:0;
  mod_readarea(header->message_area,&area);
  get_message(header,message_buffer);

  if (dynamic && header->message_area!=0)
    {
    mprintf(1,"{_>00%02d eos _>0000}",headersize);
    }
  else
    {
    port_txstring("{cls}",1);
    }

  if (header->message_area!=0)
    {
    mprintf(1,"{bfgbg cb}%s {fg w}#%d (Reads:%d Replies:%d){eol}\n",
            area.name,header->message_number,header->noof_reads,
            header->noof_replies,
            header->message_length);
    }
  else mprintf(1,"{bg b}");

  *flags=0;
  if (header->flags&MSG_DOWNLOADED) strcat(flags," DOWNLOADED");

  mprintf(1,"{bfg c}Date   : {fg w}%-24s{fg r}%s{eol}\n",cctime(&header->date_sent),flags);
  mprintf(1,"{fg c}From   : {fg w}");
  if (header->message_area==0)
    {
    if ((header->flags&MSG_FIDO)!=0)
      {
      mprintf(16+1,"%s",header->from);
      mprintf(1," {fg y}of {fg w}%d:%d/%d.%d",header->fidofrom.zone,header->fidofrom.net,
                                        header->fidofrom.node,header->fidofrom.point);
      }
    else mprintf(16+1,"%s (#%d)",header->from,header->from_id);
    }
  else
    {
    if ((area.areaflags&FLAG_ECHOMAIL)==0) mprintf(16+1,"%s (#%d)",header->from,header->from_id);
    else mprintf(16+1,"%s",header->from);
    }
  mprintf(1,"{eol}\n");

  if (header->message_area!=0 || user.usernumber!=header->from_id)
    {
    mprintf(1,"{fg c}To     : {fg w}");
    if (header->to_id!=-1 && (area.areaflags&FLAG_ECHOMAIL)==0)
      mprintf(16+1,"%s (#%d)",header->to,header->to_id);
    else
      mprintf(16+1,"%s",header->to);
    mprintf(1,"{eol}\n");
    }

  mprintf(1,"{fg c}Subject: {fg w}");
  if (header->reply_from==-1 || header->message_area==0)
    {
    port_txstring(header->subject,16+1);
    mprintf(1,"{eol std}{bfg y}\n\n");
    }
  else
    {
    port_txstring(header->subject,16+1);
    mprintf(1," {fg c}(reply to #%d){eol std}\n\n{bfg y}",header->reply_from);
    }
 
  /* Show message */
  eom=(ptr=message_buffer)+header->message_length;

  while(ptr<eom)
    {
    if ((ll=strlen(ptr))>79)
      {
      /* Wordwrap line */
      a=78;
      while(ptr[a]!=' ' && ptr[a]!='-' && ptr[a]!='.' && ptr[a]!=',' && a>65) a--;
      ll=a; if (a>65) ll++;
      }

    if (ptr[0]!=1)
      {
      if (strncmp(ptr," * Origin:",10)==0) abort=1;
      while(ll--) port_txw(*ptr++);
      if (*ptr==0) ptr++;
      port_txw(13); port_txw(10);
      if (++line==scrlen && nonstop==0)
        {
        line=0;

        mprintf(1,"{bfg w}More: {fg c}[{fg y}Return{fg c}]{fg w} continues, {fg c}[{fg y}N{fg c}]{fg w}onstop, {fg c}[{fg y}Ctrl-C{fg c}]{fg w} aborts{fg y}\015");
        do
          {
          switch(tolower(port_get()))
            {
            case 3:   abort=2; goto ool;
            case 13:
            case 32:  goto ool;
            case 'n': nonstop=1; goto ool;
            }
          }
        while(1);

        ool:
        if (dynamic)
          {
          mprintf(1,"{_>00%02d eos _>00%02d",headersize+1,headersize+1);
          }
        else mprintf(1,"                                                            \015");
        }   
      }
    else ptr+=(ll+1);
         
    if (port_rx()==3) abort=2;
    if (abort) break;
    }
                               
  /* If a real abort, clear TX buffer */
  if (abort==2) port_txclear();
   
  /* Show reply-list */
  if ((a=header->noof_replies)!=0 && header->message_area!=0)
    {
    mprintf(1,"{fg c}Latest repl%s: {fg w}#%d",a==1?"y":"ies",header->replies[0]);
    if (--a)
      {  
      int b=1;
      if (a>7) a=7;
      while(a--) mprintf(1,",#%d",header->replies[b++]);
      }
    port_crlf();
    }

  /* Voting display */
  if (header->flags&MSG_VOTE)
    {
    mprintf(1,"{fg c}Votes {fg r}FOR{fg c}: {fg w}%d{fg c}, Votes {fg r}AGAINST{fg c}: {fg w}%d{std}.\n",
            header->votes_for,header->votes_against);
    }

  /* If linked to a file */
  if (header->file_location>=0)
    {
    fileheader->message_number=header->file_location;
    get_messageh(fileheader);

    mprintf(1,"{fg c}Linked to file '%s' (length %d bytes){std}\n",
    fileheader->to,fileheader->file_length);
    }

  if (header->flags&MSG_SYSOP)
    {
    mprintf(1,"{bfg r}Message is INVISIBLE{std}\n");
    }
  }

int do_publicreply(mail_block *header)
  {
  mail_block header2; mail_area area;

  read_area(header->message_area,&area);
  setup_mailheader(&header2,&user,header->message_area);

  if (can_write(&area)==0)
    {
    mprintf(1,"{bfg r}You are not authorized to write to this area.{std}\n");
    return(0);
    }

  if ((area.areaflags&FLAG_ECHOMAIL)!=0)
    {
    header2.flags|=MSG_FIDO;
    strcpy(header2.from,fido_rules(user.username));
    }
  else strcpy(header2.from,user.username);

  header2.reply_from=header->message_number;
  header2.to_id=header->from_id;
  strcpy(header2.to,header->from);
  strcpy(header2.subject,header->subject);

  mprintf(1,"\n{bfg c}Area   : {fg w}%s\n{fg c}Date   : {fg w}%s\n",area.name,cctime(&header2.date_sent));
  mprintf(1,"{fg c}From   : {fg w}");
  if ((area.areaflags&FLAG_ECHOMAIL)==0)
    {
    mprintf(16+1,"%s (#%d)\n",header2.from,header2.from_id);
    mprintf(1,"{fg c}To     : {fg w}");
    mprintf(16+1,"%s (#%d)\n",header2.to,header2.to_id);
    }
  else
    {
    mprintf(16+1,"%s\n",header2.from);
    mprintf(1,"{fg c}To     : {fg w}");
    mprintf(16+1,"%s\n",header2.to);
    }

  if (entertext(&header2,0))
    {
    if (area.areaflags&FLAG_ECHOMAIL)
      {
      header2.message_length+=sprintf(message_buffer+header2.message_length,"--- %s",tearline)+1;
      if (ouraddress.point)
        {
        header2.message_length+=sprintf(message_buffer+header2.message_length,
                                        " * Origin: %s (%d:%d/%d.%d)",
                                        ourname,ouraddress.zone,ouraddress.net,
                                        ouraddress.node,ouraddress.point)+1;
        }
      else
        {
        header2.message_length+=sprintf(message_buffer+header2.message_length,
                                        " * Origin: %s (%d:%d/%d)",
                                        ourname,ouraddress.zone,ouraddress.net,
                                        ouraddress.node)+1;
        }

      header2.message_length+=sprintf(message_buffer+header2.message_length,
                                      "SEEN-BY: %d/%d",
                                      ouraddress.net,ouraddress.node)+1;
      }

    mprintf(1,"{bfg g}Adding message (%d bytes) as message #{fg w}",header2.message_length);
    if (put_message(&header2,message_buffer)==OK) mprintf(1,"%d{std}\n",savednr);
    }
  return(1);
  }

int do_privatereply(mail_block *header)
  {
  mail_block header2;
              
  setup_mailheader(&header2,&user,0);
  header2.to_id=header->from_id;
  strcpy(header2.to,header->from);

  mprintf(1,"\n{bfg c}Area   : {fg w}Private mail\n{fg c}Date   : {fg w}%s\n{fg c}From   : {fg w}%s (#%d)\n{fg c}To     : {fg w}%s (#%d)\n",cctime(&header2.date_sent),header2.from,header2.from_id,header2.to,header2.to_id);
  
  if (entertext(&header2,0))
    {
    mprintf(1,"\n{bfg g}Adding message (%d bytes) as message #{fg w}",header2.message_length);
    if (put_message(&header2,message_buffer)==OK) mprintf(1,"%d{std}\n",savednr);
    } 
  return(1);
  }

void do_forme()
  {
  int count,a,abort=0,max=areamax(),current,lastpoll=clock();
  mail_area area; mail_block header,fileheader;

  /* Message scan */
  for(count=1;count<max && abort==0;count++)
    {
    if (conf_member(user.conferences,count)==0) continue;
    read_area(count,&area);
    if ((area.areaflags&FLAG_FILEAREA)!=0 || area.name[0]==0) continue;
    if (can_read(&area)==0) continue;
 
    current=user.m_message;
    if (current==0 || current>area.last_message) continue;
    if (current<area.first_message) current=area.first_message;

    while(current<=area.last_message && current>=area.first_message)
      {
      if ((clock()-lastpoll)>MAXIMUM_POLL) { window_poll(); lastpoll=clock(); }
      header.message_number=current;
      get_messageh(&header);
      if (header.message_area!=count) { current++; continue; }
      current=header.message_forward;

      if (strcmp(header.to,user.username)==0)
        {
        mprintf(1,"{bfg c}%-30s {fg g}From: %-30s\n{fg c}Read (Yes/No) or Abort? ",area.name,header.from);
        do
          {
          a=tolower(port_get());
          }
        while(a!='y' && a!='n' && a!=13 && a!='a');
        if (a=='n') { port_txstring("No\n",1); continue; }
        if (a=='a') { port_txstring("Abort\n",1); return; }
        if (a=='y' || a==13)
          {
          port_txstring("Yes\n",1);
          display_message(&header,&fileheader);
          bbs_sendfile((area.areaflags&FLAG_ECHOMAIL)?"<ARCbbs$prompts>.read_fscan":
                                                      "<ARCbbs$prompts>.read_scan",0,0);

          andagain:
          do
            {
            a=tolower(port_get());
            }
          while(a!='r' && a!='p' && a!='n');
          switch(a)
            {
            case 'r':
              port_txstring("Reply\n",1);
              do_publicreply(&header);
              break;
            case 'p':
              if ((area.areaflags&FLAG_ECHOMAIL)==0)
                {
                port_txstring("Reply\n",1);
                do_privatereply(&header);
                }
              else goto andagain;
              break;
            case 'n':
              port_txstring("Next\n",1);
              break;
            }
          }
        }
      }
    }
  }

int do_filelist(int filearea,int shortdes,int option)
  {
  int current,read_dir,gotopt,abort=0,quit=0,line=1,
      nonstop=(user.f_user&USER_NOMORE);
  mail_block fileheader; mail_area area; clock_t lastpoll=clock();

  read_area(filearea,&area);
  if ((area.areaflags&FLAG_FILEAREA)==0)
    {
    mprintf(1,"{bfg r}Area %d (%s) is not a file area.{std}\n",filearea,area.name);
    return(1);
    }

  if (shortdes)
    {
    port_txstring("{bfgbg wb}File # {fg y}Name                 {fg w}Length {fg y}Short description                       {std}\n",1);
    if (user.termtype!=TER_ANSI) port_txstring("------ -------------------- ------ ----------------------------------------\n",1);
    current=area.last_message;
    read_dir=-1;
    }
  else
    {
    /* Send prompt file */
    if (option==0)
      {
      bbs_sendfile("<ARCbbs$prompts>.fread_opt",0,0);
      }
    else
      {
      mprintf(1,"{bfg c}Scanning {fg g}%-30s\015",
                 area.name); 
      }

    do
      {
      gotopt=1;
      buglib_message("bbs: rxbuffer()=%d\n",port_rxbuffer());
      switch(toupper(option?option:port_get()))
        {
        /* Forward */
        case 'F':
          {
          port_txstring("{bfg c}Forward from first{std}\n",1);
          current=area.first_message;
          read_dir=1;
          break;
          }
        /* Backward */
        case 'B':
          {
          port_txstring("{bfg c}Backward from last{std}\n",1);
          current=area.last_message;
          read_dir=-1;
          port_crlf();
          break;
          }
        /* New */
        case 'N':
          {
          if (option==0) port_txstring("{bfg c}New files{std}\n",1);
          current=user.m_file; readnewfiles=1; read_dir=1;
          if (current==0) current=area.last_message;
          if (current<area.first_message) current=area.first_message;
          if (current>area.last_message) current=-1;
          break;
          }
        /* Abort */
        case 'A':
          {
          port_txstring("{bfg r}Abort{std}\n",1);
          return(1);
          }
        /* Default */
        default:
          {
          gotopt=0;
          break;
          }
        }
      }
    while(!gotopt);
    }

  if (current==-1)
    {
    if (option==0) mprintf(1,"No files in filearea %s\n",area.name);
    return(0);
    }   

  if (shortdes==0) port_crlf();

  do
    {
    fileheader.message_number=current;
    get_messageh(&fileheader);

    if ((clock()-lastpoll)>(MAXIMUM_POLL/2))
      {
      window_poll(); lastpoll=clock();
      }

    /* Read file info */
    if (shortdes)
      {         
      char shortd[41]; int len;

      if (fileheader.message_area!=filearea)
        {
        /* For buglets */
        mprintf(1,"\nArea mismatch: %d<>%d",fileheader.message_area,
                                                    filearea);
        return(0);
        }
 
      if (((fileheader.flags&FILE_SYSOP)==0 || is_super(&area))
          && abort==0 && fileheader.message_area==filearea &&
          fileheader.status==1)
        {
        len=strlen(fileheader.subject);
        strncpy(shortd,fileheader.subject,40);
        shortd[40]=0;

        if (fileheader.flags&FILE_SYSOP)
          {
          abort|=mprintf(0,"{bfg r}%06d {fg w}",fileheader.file_location);
          }
        else
          {
          abort|=mprintf(0,"{bfg g}%06d {fg w}",fileheader.file_location);
          }
        abort|=mprintf(16+0,"%-20s ",fileheader.to);
        abort|=mprintf(0,"{fg y}%6d {fg c}",fileheader.file_length);
        abort|=mprintf(16+0,"%s\n",shortd);
   
        if (len>40)
          {
          if ((++line)==user.pagelen && nonstop==0)
            {
            int ch;
            line=0;
            mprintf(1,"{bfg w}More: {fg c}[{fg y}Return{fg c}]{fg w} continues, {fg c}[{fg y}N{fg c}]{fg w}onstop, {fg c}[{fg y}Ctrl-C{fg c}]{fg w} aborts{fg y}\015");
            do
              {
              switch(ch=toupper(port_get()))
                {
                case 3: abort=1; break;
                case 'N': nonstop=1; break;
                }
              }
            while(ch!=32 && ch!=13 && ch!='N' && ch!=3);
            mprintf(1,"                                                            \015"); 
            }
         
          if (abort==0)
            {
            abort|=mprintf(16+0,"                                   %s",
                                fileheader.subject+40);
            abort|=mprintf(0,"{std}\n");
            }
          }

        if ((++line)==user.pagelen && nonstop==0)
          {
          int ch;
          line=0;
          mprintf(1,"{bfg w}More: {fg c}[{fg y}Return{fg c}]{fg w} continues, {fg c}[{fg y}N{fg c}]{fg w}onstop, {fg c}[{fg y}Ctrl-C{fg c}]{fg w} aborts{fg y}\015");
          do
            {
            switch(ch=toupper(port_get()))
              {
              case 3: abort=1; break;
              case 'N': nonstop=1; break;
              }
            }
          while(ch!=32 && ch!=13 && ch!='N' && ch!=3);
          mprintf(1,"                                                            \015"); 
          }
        quit=abort;
        }
      }
    else
      {
      if (fileheader.message_area==filearea && fileheader.status==1 &&
          ((fileheader.flags&FILE_SYSOP)==0 || is_super(&area)))
        {
        get_message(&fileheader,message_buffer);
        quit=longoptions(&fileheader,&area,&read_dir);
        }
      }

    if (fileheader.message_area==filearea && fileheader.status==1)
      {
      current=(read_dir==1)?fileheader.message_forward:
                            fileheader.message_backward;
      }
    else
      {
      current+=read_dir;
      }
    }
  while(current!=-1 && quit==0 && current<=area.last_message &&
        current>=area.first_message);

  return(quit);
  }

int longoptions(mail_block *fileheader,mail_area *area,int *read_dir)
  {
  int len=fileheader->message_length,dltime,abort,pointer,ch,gotopt,quit=0,
      countit=(area->areaflags&FLAG_FREEDOWNLOAD)==0,privileged=0;
  char *menu;           
  
  if (is_super(area)) privileged=1;

  /* Check for archive extensions */
  menu=((fileheader->to)+strlen(fileheader->to)-4);
  if (menu[0]=='.')
    {
    /* .ZIP */
    if (toupper(menu[1])=='Z' && toupper(menu[2])=='I' &&
        toupper(menu[3])=='P') fileheader->flags|=FILE_ARC;
    /* .PAK */
    if (toupper(menu[1])=='P' && toupper(menu[2])=='A' &&
        toupper(menu[3])=='K') fileheader->flags|=FILE_ARC;
    /* .ARC */
    if (toupper(menu[1])=='A' && toupper(menu[2])=='R' &&
        toupper(menu[3])=='C') fileheader->flags|=FILE_ARC;
    /* .LZH */
    if (toupper(menu[1])=='L' && toupper(menu[2])=='Z' &&
        toupper(menu[3])=='H') fileheader->flags|=FILE_ARC;
    /* .ARJ */
    if (toupper(menu[1])=='A' && toupper(menu[2])=='R' &&
        toupper(menu[3])=='J') fileheader->flags|=FILE_ARC;
    }

  if (baudrate!=0)
    {
    dltime=(fileheader->file_length*100)/(baudrate*10);
    }
  else dltime=0;

/*
100245 spark.arc (123456 bytes) downloaded 157 times
Uploaded by Hugo Fiennes (#1), on Thu Oct 26 20:35:58 1989
File is binary and private. Approximate download time: 20:30.
<short desc>
<long desc>
*/  

  if (mprintf(0,"{bfg g}%06d {fg w}%-20s {fg y}%6d {fg c}(downloaded %3d times)\n",
              fileheader->file_location,fileheader->to,
              fileheader->file_length,fileheader->to_id)) goto options;
  if (mprintf(0,"Uploaded by {fg w}%s {fg c}({fg w}#%d{fg c}), on %s{std}\n",
              fileheader->from,fileheader->from_id,
              cctime(&fileheader->date_sent))) goto options;
  if (mprintf(0,"{bfg c}File is ")) goto options;
  if (fileheader->flags&FILE_ARC)
    {
    if (mprintf(0,"an ARCfile")) goto options;
    }
  else
    {
    if (mprintf(0,(fileheader->flags&FILE_BINARY)!=0?"binary":"ASCII")) goto options;
    }

  if (mprintf(0,(fileheader->flags&FILE_PRIVATE)!=0?" and private.":
                                                     ".")) goto options;
  if (mprintf(0," Approximate download time: {fg w}%d:%02d{std}\n",
              dltime/60,dltime%60)) goto options;
  if (mprintf(0,"{bfg c}Short description: {fg w}")) goto options;
  if (mprintf(16+0,"%s\n",fileheader->subject)) goto options;
 
  if (fileheader->message_length)
    {                                    
    mprintf(1,"{fg c}Description:{std}\n");
    abort=pointer=0;

    /* Show description */
    while(len-- && abort==0)
      {
      if ((ch=message_buffer[pointer++])!=0) port_txw(ch); else port_crlf();     
      abort|=port_rxbuffer();
      }
    port_crlf();
    }

  if (fileheader->flags&FILE_SYSOP)
    {
    mprintf(1,"{bfg r}File is INVISIBLE{std}\n");
    }

  options:

  menu=privileged?"<ARCbbs$prompts>.fread_sa":"<ARCbbs$prompts>.fread_a";
  bbs_sendfile(menu,0,0);

  do
    {
    gotopt=1;

    /* Options! */
    switch(toupper(port_get()))
      {
      /* Queue */
      case 'Q':
        {
        port_txstring("{bfg g}Queue\n",1);
        file_download(fileheader->file_location,fileheader->file_length,
                      fileheader->to,0,countit);

        /* Inc dload count here I suppose... */
        fileheader->to_id++;
        put_messageh(fileheader);

        /* So we come back to same prompt */
        bbs_sendfile(menu,0,0);
        gotopt=0;
        break;
        }
      /* Download */
      case 'D':
        {
        port_txstring("{bfg g}Download\n",1);
        file_download(fileheader->file_location,fileheader->file_length,
                      fileheader->to,1,countit);

        /* Inc dload count here I suppose... */
        fileheader->to_id++;
        put_messageh(fileheader);

        /* So we come back to same prompt */
        bbs_sendfile(menu,0,0);
  
        gotopt=0;
        break;
        }
      /* Erase */
      case 'E':
        {
        port_txstring("{bfg g}Erase\n",1);
        if (privileged==0 &&
            user.usernumber!=fileheader->from_id)
          {
          port_txstring("{bfg r}Access violation{std}\n",1);
          }
        else
          {
          if (port_yesno("{bfg r}Erase: are you sure? "))
            {
            delete_message(fileheader->message_number);
            if (port_yesno("\n{bfg r}Remove physical file? ")) remove(file_pathname(fileheader->file_location));
            }
          port_txstring("{std}",1);
          }
        break;
        }
      /* Move */
      case 'M':
        {
        port_txstring("{bfg g}Move file\n",1);
        if (privileged==0)
          {
          port_txstring("{bfg r}Access violation{std}\n",1);
          }
        else
          {
          char numb[10]; mail_block move=*fileheader;

          mprintf(1,"\nEnter filebase number #:");
          port_readline(numb,9,0); port_crlf();

          if (*numb==0) break;
          mprintf(1,"Moving file...");
          delete_message(fileheader->message_number);
          move.message_area=atoi(numb);

          if (put_message(&move,message_buffer)==OK)
            {
            mprintf(1,"Done{std}\n");
            }
          }
        break;
        } 
      /* Rename */
      case 'R':
        {
        if (privileged)  
          {                                        
          port_txstring("Rename\n{bfg g}New name: ",1);
          port_readline(fileheader->to,20,EXISTING);
          port_crlf();
          put_messageh(fileheader);
          bbs_sendfile(menu,0,0);
          }
        gotopt=0;
        break;
        }          
      /* Change description */
      case 'C':
        {
        if (privileged)  
          {                                        
          port_txstring("Change description\n",1);
          if (entertext(fileheader,1))
            {                        
            if (fileheader->block_forward>=0)
              {
              _mail_freeup(fileheader->block_forward);
              } 
            if (fileheader->message_length>0)
              {
              fileheader->block_forward=mod_lockdata(fileheader->message_length);
              if (fileheader->block_forward==-1) break;
              _mail_buffertodata(message_buffer,
                                 fileheader->message_length,
                                 fileheader->block_forward);
              }
            put_messageh(fileheader);
            mprintf(1,"\n{bfg r}Description changed\n");
            }
          bbs_sendfile(menu,0,0);
          }
        gotopt=0;
        break;
        }
      /* Sysop access toggle */
      case '+':
        {
        if (privileged)
          {
          fileheader->flags^=FILE_SYSOP;

          mprintf(1,"{bfg r}%s{std}\n",fileheader->flags&FILE_SYSOP?"Invisible":"Visible");

          /* Rerewrite file data */
          put_messageh(fileheader);
          bbs_sendfile(menu,0,0);
          }                 
        gotopt=0;
        break;
        }
      /* Toggle */
      case 'T':
        {
        *read_dir=-(*read_dir);
        port_txstring((*read_dir)==1?"Forward\n":"Backward\n",1);
        break;
        }
      /* View arc file */
      case 'V':
        {                       
        port_txstring("View archive\n\n",1);
        show_arc(file_pathname(fileheader->file_location),fileheader->to);
        port_crlf();
        bbs_sendfile(menu,0,0);
        gotopt=0;
        break;
        }
      /* Abort */
      case 'A':
        {
        port_txstring("{bfg r}Abort{std}\n",1);
        quit=1; break;
        }
      case '?':
        {
        bbs_sendfile(menu,0,0);
        gotopt=0;
        break;
        }
      /* Next message */
      case 13:
      case 'N':
      case ' ':
        {
        port_txstring("{bfg g}Next{std}\n",1);
        break;
        }
      default:
        {
        gotopt=0;
        break;
        }
      }
    }
  while(gotopt==0);

  return(quit);
  }

int message_read(int arean,mail_area *area,int option)
  {
  int current_msg,read_dir,gotopt; mail_block header,fileheader;
  char *menu; clock_t lastpoll=clock();
 
  /* Send prompt file */
  if (option==0) 
    {
    bbs_sendfile("<ARCbbs$prompts>.read_opt",0,0);
    }
  else
    {
    mprintf(1,"{bfg c}Scanning {fg g}%-30s\015",
              area->name); 
    }

  do
    {
    gotopt=1;
    switch(toupper(option?option:port_get()))
      {
      /* Forward */
      case 'F':
        {
        char msgnr[10];
        port_txstring("{bfg c}Forward from (CR=first) {fg y}#",1);
        port_readline(msgnr,8,NUMONLY);
        current_msg=atoi(msgnr);
        if (msgnr[0]==0) current_msg=area->first_message;
        if (current_msg>area->last_message) current_msg=area->last_message;
        if (current_msg<area->first_message) current_msg=area->first_message;
        read_dir=1;
        port_crlf();
        break;
        }
      /* Backward */
      case 'B':
        {
        char msgnr[10];
        port_txstring("{bfg c}Backward from (CR=last) {fg y}#",1);
        port_readline(msgnr,8,NUMONLY);
        current_msg=atoi(msgnr);    
        if (msgnr[0]==0) current_msg=area->last_message;
        if (current_msg==0) current_msg=area->last_message;
        if (current_msg>area->last_message) current_msg=area->last_message;
        if (current_msg<area->first_message) current_msg=area->first_message;
        read_dir=-1;
        port_crlf();
        break;
        }
      /* Goto */
      case 'G':
        {
        char msgnr[10];
        port_txstring("{bfg c}Goto message {fg y}#",1);
        port_readline(msgnr,8,NUMONLY);
        current_msg=atoi(msgnr); 
        if (current_msg<area->first_message) current_msg=area->first_message;
        if (current_msg>area->last_message) current_msg=area->last_message;
        read_dir=1;
        port_crlf();
        break;
        }
      /* New */
      case 'N':
        {
        if (option==0) port_txstring("{bfg c}New messages{std}\n",1);
        current_msg=user.m_message; readnew=1; read_dir=1;
        if (current_msg==0) current_msg=area->last_message;
        if (current_msg<area->first_message) current_msg=area->first_message;
        if (current_msg>area->last_message) current_msg=-1;
        break;
        }
      /* Abort */
      case 'A':
       {
        port_txstring("{bfg r}Abort{std}\n",1);
        return(1);
        }
      /* Default */
      default:
        {
        gotopt=0;
        break;
        }
      }
    }
  while(!gotopt);

  if (current_msg==-1 || area->count==0)
    {
    if (option==0) mprintf(1,"{bfg c}Sorry, there are no messages to read in {fg r}%s{fg c}.{std}\n",area->name);
    return(0);
    }            

  port_crlf();

  do
    {
    if ((clock()-lastpoll)>MAXIMUM_POLL)
      {
      window_poll(); lastpoll=clock();
      }

    header.message_number=current_msg;
    get_messageh(&header);
      
    if (header.message_area==arean && header.status==1)
      {
      if (is_super(area) || (header.flags&MSG_SYSOP)==0)
        {
        display_message(&header,&fileheader);
     
        if ((area->areaflags&FLAG_ECHOMAIL)==0)
          {
          if (is_super(area))
            {
            if (header.flags&MSG_VOTE) menu="<ARCbbs$prompts>.read_sv";
            else menu="<ARCbbs$prompts>.read_s";
            if (header.file_location>=0) menu="<ARCbbs$prompts>.read_sf";
            }
          else
            {
            if (header.flags&MSG_VOTE) menu="<ARCbbs$prompts>.read_v";
            else menu="<ARCbbs$prompts>.read_";
            if (header.file_location>=0) menu="<ARCbbs$prompts>.read_f";
            }
          }
        else
          {
          if (is_super(area))
            {
            menu="<ARCbbs$prompts>.read_es";
            }
          else
            { 
            menu="<ARCbbs$prompts>.read_e";
            }
          }

        bbs_sendfile(menu,0,0);

        do
          {
          gotopt=1;

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
                  int gotopt;
                  bbs_sendfile("<ARCbbs$prompts>.read_fm",0,0);

                  do
                    {
                    gotopt=1;
                    switch(toupper(port_get()))
                      {
                      /* Queue file */
                      case 'Q':
                        {
                        mail_area filearea;

                        read_area(fileheader.message_area,&filearea);
                        port_txstring("{bfg g}Queue file{std}\n",1);
                        file_download(fileheader.file_location,
                                      fileheader.file_length,
                                      fileheader.to,0,
                                      (filearea.areaflags&FLAG_FREEDOWNLOAD)==0);
                        break;
                        }
                      /* Download file */
                      case 'D':
                        {
                        mail_area filearea;

                        read_area(fileheader.message_area,&filearea);
                        port_txstring("{bfg g}Download file{std}\n",1);
                        file_download(fileheader.file_location,
                                      fileheader.file_length,
                                      fileheader.to,1,
                                      (filearea.areaflags&FLAG_FREEDOWNLOAD)==0);
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
                        gotopt=0;
                        break;
                        }
                      }
                    }
                  while(!gotopt);
                  }
                while(!quit);
                bbs_sendfile(menu,0,0);
                }
              gotopt=0;
              break;
              }
            /* Reply */
            case 'R':
              {
              port_txstring("{bfg g}Reply\n",1);
              if (!do_publicreply(&header)) gotopt=0;
              break;
              }
            /* Private reply */
            case 'P':
              {
              if ((area->areaflags&FLAG_ECHOMAIL)==0)
                {
                port_txstring("{bfg g}Private reply\n",1);
                do_privatereply(&header);
                }
              else gotopt=0;
              break;
              }
            /* Toggle forward/backward */
            case 'T':
              {
              read_dir=-read_dir;
              port_txstring("{bfg g}Read direction toggled to {fg w}",1);
              if (read_dir==1) port_txstring("FORWARD{std}\n",1);
              else port_txstring("BACKWARD{std}\n",1);
              break;
              }
            /* Goto */
            case 'G':
              {
              char msgnr[10];

              port_txstring("{bfg g}Goto message #",1);
              port_readline(msgnr,9,NUMONLY);
              current_msg=atoi(msgnr)-1;
              if (current_msg>area->last_message) current_msg=area->last_message;
              if (current_msg<area->first_message) current_msg=area->first_message;
              read_dir=1;
              port_crlf();
              goto goto_new; 
              }
            /* Delete */
            case 'D':
              {
              port_txstring("{bfg g}Delete\n",1);
              if (is_super(area)==0 &&
                   user.usernumber!=header.from_id &&
                   user.usernumber!=header.to_id)
                {
                port_txstring("\n{bfg r}Access violation{std}\n",1);
                }
              else
                {
                if (port_yesno("{bfg r}Are you sure? {std}"))
                  {
                  delete_message(current_msg);
                  }
                port_crlf();
                }
              goto nextmsg;
              break;
              }   
            /* Vote */
            case 'V':
              {
              if (header.flags&MSG_VOTE)
                {
                int ch;

                port_txstring("{bfg g}Vote\n{fg r}FOR{fg g} or {fg r}AGAINST{fg g}? {fg r}",1);
                do
                  {
                  ch=toupper(port_get());
                  }
                while(ch!='F' && ch!='A');
                if (ch=='F')
                  {
                  port_txstring("For{fg g}\n",1);
                  header.votes_for++;
                  }
                else
                  {
                  port_txstring("Against{fg g}\n",1);
                  header.votes_against++;
                  }
                /* Rerewrite message data */
                put_messageh(&header);
                }
              break;
              }
            /* Sysop access toggle */
            case '+':
              {
              if (is_super(area))
                {
                header.flags^=MSG_SYSOP;

                mprintf(1,"{bfg r}%s{std}\n",(header.flags&MSG_SYSOP)?"Invisible":"Visible");

                /* Rerewrite message data */
                put_messageh(&header);
                goto nextmsg;
                }
              else gotopt=0;
              break;
              }
            /* Change subject */
            case 'C':
              {
              if (is_super(area))
                {
                port_txstring("Change subject\n",1);
                port_readline(header.subject,60,EXISTING);
                put_messageh(&header);
                port_txstring("\nSubject changed\n",1);
                }
              else gotopt=0;
              break;
              }
            /* Move message */
            case 'M':
              {
              if (is_super(area))
                {
                char buff[5]; int darea; mail_area ar;
                mail_block mb; char *nm;

                port_txstring("Move message\nArea to move message to: ",1);
                port_readline(buff,4,NUMONLY);
                port_crlf();
                if ((darea=atoi(buff))==0)
                  {
                  port_txstring("Move aborted\n",1);
                  break;
                  }

                read_area(darea,&ar);
                if (ar.areaflags&FLAG_FILEAREA)
                  {
                  port_txstring("Can't move to a filearea!\n",1);
                  break;
                  }
                              
                if (ar.areaflags&FLAG_ECHOMAIL)
                  {
                  port_txstring("Can't move to an echomail area!\n",1);
                  break;
                  }

                setup_mailheader(&mb,&user,darea);
                strcpy(mb.from,header.from);               /*added*/
                mb.from_id=header.from_id;                 /*added*/
                strcpy(mb.to,header.to);                   /*altered*/
                mb.to_id=header.to_id;                     /*altered*/
                strcpy(mb.subject,header.subject);
                nm=message_buffer;
                nm+=sprintf(nm,"** Crossposted from %s by %s **",area->name,user.username)+1;
                *nm++=0;
                get_message(&header,nm);
                mb.message_length=(nm-message_buffer)+header.message_length;
                put_message(&mb,message_buffer);

                delete_message(current_msg);
                port_txstring("Message moved\n",1);
                goto nextmsg;
                }
              else gotopt=0;
              break;
              }
            /* Abort/quit */
            case 'A':
              {
              port_txstring("{bfg r}Abort{std}\n",1);
              return(1);
              }
            case 'Q':
              {
              port_txstring("{bfg r}Quit{std}\n",1);
              return(1);
              }
            /* Next message */
            case 13:
            case 'N':
            case ' ':
              {
              port_txstring("{bfg g}Next{std}\n",1);
              goto nextmsg;
              break;
              }
            default:
              {
              gotopt=0;
              break;
              }
            }
          }
        while(!gotopt);
        }
      nextmsg:
      current_msg=(read_dir==1)?header.message_forward:header.message_backward;
      }
    else
      {       
      goto_new:
      current_msg+=read_dir;
      }
    }
  while(current_msg>=area->first_message && current_msg<=area->last_message &&
        current_msg!=-1);

  if (option==0) port_txstring("{bfg c}End of messages{std}\n",1);

  return(0);
  }

void message_write(int arean,mail_area *area,int to)
  {
  char userbuffer[31],*ptr1,*ptr2;
  mail_block header; user_block tempuser;

  setup_mailheader(&header,&user,arean);

  mprintf(1,"\n{bfg c}Area   : {fg w}%s\n{fg c}Date   : {fg w}%s\n",area->name,
             cctime(&header.date_sent));
   
  if ((area->areaflags&FLAG_ECHOMAIL)==0)
    {                            
    strcpy(header.from,user.username);
    mprintf(1,"{fg c}From   : {fg w}%s (#%d)\n",header.from,user.usernumber);
    if (to==-1)
      {
      getnameagain1:
      port_txstring("\015{fg c}To     : ",1);
      port_readline(ptr1=userbuffer,30,0);

      /* Strip leading spaces */
      while(*ptr1==' ') ptr1++;
      if (!*ptr1)
        {
        port_txstring("\015{bfg c}To     : {fgbg rb}Send aborted{std}\n",1);
        return;  /* Abort with blank username */
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
        if (!(strcmp(ptr1,"ALL")==0 && arean!=0))
          {
          /* Text is username, do hashing, etc */
          usernumber=hash_find(ptr1);

          /* Non-existant? */
          if (usernumber==-1) goto getnameagain1;
          }
        else
          {
          usernumber=-1;
          }
        }
      }
    else
      {
      usernumber=to;
      }
    header.to_id=usernumber;
    if (usernumber!=-1)
      {
      tempuser.usernumber=usernumber;
      get_user(&tempuser);
      if (tempuser.status!=1) goto getnameagain1;
      strcpy(header.to,tempuser.username);
      mprintf(1,"\015{bfg c}To     : {fgbg wb}%s (#%d){std}\n",header.to,header.to_id);
      }
    else
      {
      strcpy(header.to,"All");
      port_txstring("\015{bfg c}To     : {fgbg wb}All{std}\n",1);
      }
    }
  else
    {          
    int cap=1;
    strcpy(header.from,fido_rules(user.username));
    mprintf(1,"{fg c}From   : {fg w}%s\n",header.from);
    
    port_txstring("\015{bfg c}To     : ",1);
    port_readline(ptr1=userbuffer,30,0);

    /* Strip leading spaces */
    while(*ptr1==' ') ptr1++;
    if (!*ptr1)
      {
      port_txstring("\015{bfg c}To     : {fgbg rb}Send aborted{std}\n",1);
      return;  /* Abort with blank username */
      }

    /* Convert string to prettycase */
    ptr2=ptr1;
    while(*ptr2)
      {
      if (cap) { *ptr2=toupper(*ptr2); cap=0; }
      else *ptr2=tolower(*ptr2);
      if (*ptr2==32) cap=1;
      ptr2++;
      }   

    strcpy(header.to,ptr1);
    mprintf(1,"\015{bfg c}To     : {fgbg wb}%s{std}\n",header.to);
    
    header.flags|=MSG_FIDO;
    header.to_id=0;
    }

  header.subject[0]=0;

  if (entertext(&header,0))
    {
    if (area->areaflags&FLAG_ECHOMAIL)
      {
      header.message_length+=sprintf(message_buffer+header.message_length,
                                     "--- %s",tearline)+1;
      if (ouraddress.point)
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

    mprintf(1,"\n{bfg g}Adding message (%d bytes) as message #{fg w}",header.message_length);
    if (put_message(&header,message_buffer)==OK)
      {
      mprintf(1,"%d{std}\n",savednr);
      }
    }
  }

void sysop_chat()
  {
  char typebuffer[20];
  clock_t lastkey=0;
  int typepos=0,timestop=(clock()-logon_time);

  chatting=1;
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                  0,"Sysop chat");

  got_data=0;

  do
    {                   
    /* Time is stopped */
    logon_time=(clock()-timestop);
    if (got_data)
      {
      got_data=0;
      if (got_type==BBS_CHATDATA)
        {
        port_txstring(got_string,1);
        }
      if (got_type==BBS_CHATREQUEST && got_code==NO_CHATTING)
        {
        port_txstring("\n+++ Chat terminated\n",1);
        chatting=0; beenchatting=1;
        send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                  0,"Left chat");

        return;
        }
      }

    if (port_rxbuffer())
      {
      int key=port_get();
      if (key>31 || key==13 || key==8)
        {
        switch(key)
          {    
          case 13:
          case 10:
            {
            port_txw(13);
            port_txw(10);
            typebuffer[typepos++]=10;
            break;
            }
          case 8: case 127:
            {
            port_txstring("\010 \010",1);
            typebuffer[typepos++]=127;
            break;
            }
          default:
            {
            if (key>31)
              {
              port_txw(key);
              typebuffer[typepos++]=key;
              }
            else
              {
              /* Predefined strings */
              }
            break;
            }
          }
        }
      }

    if ((clock()-lastkey)>50 || typepos==19)
      {
      typebuffer[typepos]=0;
      send_strmessage(servertask,BBS_CHATDATA,portnumber,0,0,typebuffer);
      typepos=0; lastkey=clock();
      }
    window_poll();
    }
  while(1);
  }

void showonline()
  {
  int a;

  port_txstring("{bfgbg wb}  # {fg c}BaudR {fg w}User# {fg c}Username                       {fg w}Where           {std}\n",1);

  for(a=0;a<16;a++)
    {         
    if (online_users[a].username[0])
      {
      mprintf(1,"{bfg c} %2d {fg w}%5d {fg c}%5d {fg w}%-30s {fg c}%s\n",a,
             online_users[a].baudrate,online_users[a].usernumber,
             online_users[a].username,online_users[a].doing);
      }   
    }

  port_txstring("{std}",1);
  }

