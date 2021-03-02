/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> User logon/new user module                  <]
Current version   [> 00.34                                       <]
Version date      [> 05-April-1993                               <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1989-1993 by    <]
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
#include "str.h"             /* Strings */
#include "version.h"

#include "miscbbs.h"
#include "bbs.h"
#include "cryton.h"

static int  modem_maxtx=9600,modem_maxrx=9600,modem_type=0,modem_ata=0,modem_setbaud=0;
static char modem_initstr[256],modem_welcome[128],modem_password[32];

char driver_name[12];
int driver_port,modem_inactivity;

extern int slavemode,snoop;

void modem_readconfig()
  {
  FILE *modemfile; char *p,*q;

  /* Read modem file */
  sprintf(tempbuffer,"Modem$port%d",portnumber);
  sprintf(tempbuffer2,"<ARCbbs$modems>.%s",getenv(tempbuffer));
 
  /* Open modem file & read config */
  if ((modemfile=fopen(tempbuffer2,"rb"))==NULL)
    {
    /* Boohoo */
    error("aCan't read modem config file '%s' (port %d)",tempbuffer2,portnumber);
    exit(0);
    }

  *modem_password=*modem_welcome=*modem_initstr=0;
  do
    {
    /* Get a line */
    mygets(tempbuffer,modemfile);

    if (tempbuffer[0]!=0 && tempbuffer[0]!=';')
      {
      /* Parse line */
      if ((p=strchr(tempbuffer,' '))!=NULL)
        {
        *p++=0; while(*p==' ' && *p!=0) p++;
        
        /* Remove trailing spaces */
        q=p+strlen(p)-1;
        while(*q==' ') *q--=0;
        }

      if (p==NULL) p="";

      if (stricmp(tempbuffer,"txspeed")==0) modem_maxtx=atoi(p);
      if (stricmp(tempbuffer,"rxspeed")==0) modem_maxrx=atoi(p);

      if (stricmp(tempbuffer,"driver")==0) strcpy(driver_name,p);
      if (stricmp(tempbuffer,"port")==0) driver_port=atoi(p);

      if (stricmp(tempbuffer,"modem")==0)
        {
        if (stricmp(p,"hayes")==0) modem_type=1;
        if (stricmp(p,"pipe")==0) modem_type=2;
        if (stricmp(p,"direct")==0) modem_type=3;
        }

      if (stricmp(tempbuffer,"baud")==0)
        {
        if (stricmp(p,"set")==0) modem_setbaud=1;
        if (stricmp(p,"lock")==0) modem_setbaud=0;
        if (stricmp(p,"hst")==0) modem_setbaud=2;
        if (stricmp(p,"hayes")==0) modem_setbaud=3;
        }

      if (stricmp(tempbuffer,"ringresponse")==0)
        {
        if (stricmp(p,"ata")==0) modem_ata=1;
        }

      if (stricmp(tempbuffer,"welcometext")==0) strcpy(modem_welcome,p);
      if (stricmp(tempbuffer,"linepassword")==0) strcpy(modem_password,p);
      if (stricmp(tempbuffer,"modeminit")==0) strcpy(modem_initstr,p);

      if (stricmp(tempbuffer,"inactivity")==0) modem_inactivity=atoi(p);
      }
    }
  while(feof(modemfile)==0);

  fclose(modemfile);
  }

void modem_init(void)
  {
  char *p=modem_initstr;

  /* Setup out port */
  port_speed(modem_maxtx,modem_maxrx);
  port_parity(5);
  port_dtr(1); port_rts(1);
  
  while(*p)
    {
    switch(*p)
      {
      case '^': port_dtr(1); break;
      case 'v': port_dtr(0); break;
      case '`': wimp_waitfor(10); break;
      case '~': wimp_waitfor(100); break;
      case '|': port_txw(13); break;
      default: port_txw(*p); break;
      }
    p++;
    }

  port_rxclear();
  }  

int answer_routine()
  {
  char buffer[80],*b;
  int t=clock(),c;

  arq=0;

  switch(modem_type)
    {
    case 0:
    case 1: /* Hayes */
      {
      notconnect:
      /* Wait for CONNECT routine. Get a string until
         it is CONNECT <baud rate> */
      /* Get a line upto 60 chars long */
      b=buffer;
      while((b-buffer)<60)
        {
        if (b==buffer && (clock()-t)>60000) return(0);
        if ((c=port_rx())>=0)
          {
          if (c==10) { *b++=0; break; }
          if (c>31) *b++=c;
          }
        window_poll();
        }
           
      if (strncmp(buffer,"RING",4)==0 && modem_ata!=0)
        {
        /* Send an ATA */
        port_txw('A');
        port_txw('T');
        port_txw('A');
        port_txw(13);
        }
      if (strncmp(buffer,"CONNECT",7)!=0) goto notconnect;

      /* Fix for X0 just in case */
      if (strcmp(buffer,"CONNECT")==0) strcpy(buffer,"CONNECT 300");
        
      /* Fix for V23 Microcom/Hayes/Pro4 modems */
      if (strncmp(buffer,"V.23 CONNECT",12)==0) strcpy(buffer,"CONNECT 1275");
      if (strcmp(buffer,"CONNECT V.23")==0) strcpy(buffer,"CONNECT 1275");
      if (strcmp(buffer,"CONNECT 75/1200")==0) strcpy(buffer,"CONNECT 1275");

      if (sscanf(buffer,"CONNECT %d",&baudrate)!=1) goto notconnect;
      if (strstr(buffer,"ARQ") ||
          strstr(buffer,"REL") ||
          strstr(buffer,"MNP")) arq=1;

      /* Fix for Microcom modems */
      if (baudrate==103) baudrate=300;
      if (baudrate==212) baudrate=1200;

      /* Set serial port to correct baud rate (check for 1275 buffering)
         If modem_setbaud is zero, then modem is buffering, so just keep current
         speed setting. */
      if ((modem_setbaud==1 || modem_setbaud==2) && arq==0)
        {
        int txset=baudrate,rxset=baudrate;

        if (baudrate==1275 || baudrate==7512) { txset=1200; rxset=75; }
        if (baudrate==14400) txset=rxset=19200;
        port_speed(txset,rxset);
        }
           
      if (modem_setbaud==3)
        {
        if (baudrate==1275 || baudrate==7512)
          {
          port_speed(1200,1200);
          }
        }
      break;
      }
    case 2: /* Pipe */
      {
      notconnect2:
      /* Wait for HELLO routine. Get a string until it is HELLO<cr> */
      /* Get a line upto 60 chars long */
      b=buffer;
      while((b-buffer)<60)
        {
        if (b==buffer && (clock()-t)>60000) return(0);
        if ((c=port_rx())>=0)
          {
          if (c>31)
            {
            *b++=c;
            port_txw(c);
            }
          else
            {
            if (b!=buffer) { *b++=0; break; }
            }
          }
        window_poll();
        }
           
      if (strnicmp(buffer,"hello",5)!=0) goto notconnect2;

      port_crlf();
      port_speed(modem_maxtx,modem_maxrx);
      baudrate=modem_maxrx;
      break;
      }
    case 3: /* Direct */
      {
      port_crlf();
      port_speed(modem_maxtx,modem_maxrx);
      baudrate=modem_maxrx;
      break;
      }
    }

  /* Set answer time for call billing */
  answer_time=clock();
       
  return(1);
  }

int logon_routine()
  {
  char usernumberbuffer[44],*ptr1,*ptr2,eventtime[20];
  struct tm event,nowtm,laston;
  time_t now; int untilevent; mail_area temparea;
  int tries=0,serial=-1;

  send_message(servertask,BBS_ICONINFO,portnumber,0,baudrate,0);
  online_users[portnumber].baudrate=baudrate;
  online_users[portnumber].usernumber=-1;
  online_users[portnumber].username[0]=0;
  strcpy(online_users[portnumber].doing,"Answer");    
  nicelogoff=0; user.termtype=TER_TTY;

  send_message(servertask,BBS_COUNTONLINE,1,0,0,0); online++;
  send_message(servertask,BBS_COUNTOTHER,-1,0,0,0);

  /* Enable carrier drop detect */
  enablecarrier=1;
  port_xonxoff(1); /* Enable Xon/Xoff */

  /* Clear o/p buffer */
  port_txclear(); port_rxclear();

  if (strcmp(getenv("ARCterm$security"),"yes")==0 && slavemode!='l')
    {
    clock_t s;

    /* Send autoseq */
    mprintf(1,"M\010N\010P\0107\010 \010");
    port_waitout();
    s=clock();                                                 
    do
      {
      while(port_rxbuffer()==0 && (clock()-s)<300) window_poll();
      if (port_rxbuffer())
        {
        if (port_rx()==6)
          {
          int hi,lo;

          while(port_rxbuffer()==0) window_poll();
          lo=port_rx();
          while(port_rxbuffer()==0) window_poll();
          hi=port_rx();
          serial=(hi<<8)+lo;
          }
        }
      }
    while((clock()-s)<300);
    }

  /* Send initial ID message */
  mprintf(1,"\012\012ARCbbs/RISC OS MultiUser BBS v%s (Compiled %s %s)\012",version_number,version_time,version_date);
  mprintf(1,"[%d%s] Connection established on port %d\012",
           baudrate,arq?"/ARQ":"",portnumber);

  /* Send logon text file */
  if (bbs_sendfile(modem_welcome," \015\003",0)!=0) port_txclear();
  
  send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                  0,"Logging on");

  /* Throw them off after 4 mins if no sucess */
  thislogon=4; notimeout=-1;

  /* Check for password */
  if (modem_password[0])
    {
    char pwbuffer[20];
    port_readline(pwbuffer,19,NOECHO|NOBOX);
    if (strcmp(pwbuffer,modem_password)) return(0); 
    }
          
  /* Get username */
  getnameagain:
  do
    {
    port_txstring("Usernumber or username (NEW for new user): ",1);
    port_readline(ptr1=usernumberbuffer,40,NOBOX);
    port_crlf();
    /* Strip leading spaces */
    while(*ptr1==' ') ptr1++;
    }
  while (!*ptr1);
                      
  /* Change to uppercase */
  ptr2=ptr1;
  while(*ptr2) { *ptr2=toupper(*ptr2); ptr2++; }
                          
  /* Strip trailing spaces */
  ptr2=(ptr1+strlen(ptr1)-1);
  while(*ptr2==' ' && ptr2>ptr1) ptr2--;
  *++ptr2=0;

  /* If numeric first digit, get as a usernumber */
  if (isdigit(*ptr1)) 
    {
    usernumber=atoi(ptr1);
    if (usernumber==0 && portnumber<9) usernumber=-1;
    }
  else
    {
    /* If 'NEW' handle new user */
    if (!strcmp(ptr1,"NEW"))
      {
      usernumber=-1;
      }
    else
      {
      /* Text is username, do hashing, etc */
      usernumber=hash_find(ptr1);
      if (usernumber==0 && portnumber<9) usernumber=-1;              

      /* Non-existant? */
      if (usernumber==-1)
        {
        if (!port_yesno("Unknown user. Are you a new user? "))
          goto getnameagain;
        }
      else
        {
        mprintf(1,"Your usernumber is #%d\012",usernumber);
        }
      }
    }
                    
  /* Get the user */
  if ((user.usernumber=usernumber)!=-1)
    {
    get_user(&user);
    if (!user.status)
      {
      if (!port_yesno("Unknown user. Are you a new user? "))
        goto getnameagain;
      usernumber=-1;
      }
    }
     
  if (usernumber==-1)
    {
    /* New user stuff */
    int exists,capitals,pos,tmp;

    /* Let server know what's happening */
    send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                    0,"New user");
    strcpy(online_users[portnumber].doing,"New user");

    /* Get template user */
    user.usernumber=0; get_user(&user);
    if (user.userlevel==0)
      {
      port_txstring("\012Sorry, new user logons are currently disabled.\012\012",1);
      wimp_waitfor(200);
      return(0);
      }

    bbs_sendfile("<ARCbbs$text>.NewUser",0,0);

    do
      {
      /* Get user info */
      do
        {
        char inname[30],*start,*end;

        port_txstring("Username                          : ",1);
        port_readline(inname,30,0);
                 
        /* Strip leading and trailing spaces */
        start=inname; while(*start==' ') start++;
        end=(start+strlen(start)-1);
        while(*end==' ' && end>start) end--;
        *++end=0;                              
        strcpy(user.username,start);

        if ((exists=hash_find(user.username))!=NODATA)
          {
          port_txstring("\012Sorry, that username is in use. If you already have an account, please\012drop the line and call again, logging in properly. Otherwise: pick another name!\012",1);
          }
        }
      while(exists!=NODATA);

      /* Make name look decent */
      for(pos=0,capitals=1;user.username[pos];pos++)
        {
        if (capitals)
          {
          user.username[pos]=toupper(user.username[pos]);
          capitals=0;
          }
        else
          {
          user.username[pos]=tolower(user.username[pos]);
          }
        if (user.username[pos]=='.' || user.username[pos]==' ') capitals=1;
        }
      port_txstring("\012Your real name                    : ",1);
      port_readline(user.realname,30,0); port_crlf();
      port_txstring("Address (NOT p/code) line 1 of 4  : ",1);
      port_readline(user.address[0],30,0); port_crlf();
      port_txstring("Address (NOT p/code) line 2 of 4  : ",1);
      port_readline(user.address[1],30,0); port_crlf();
      port_txstring("Address (NOT p/code) line 3 of 4  : ",1);
      port_readline(user.address[2],30,0); port_crlf();
      port_txstring("Address (NOT p/code) line 4 of 4  : ",1);
      port_readline(user.address[3],30,0); port_crlf();
      port_txstring("Postcode                          : ",1);
      port_readline(user.postcode,10,0); port_crlf();
      port_txstring("Telephone number                  : ",1);
      port_readline(user.telephone,30,0); port_crlf();
      do_set(1,0,0,0,user.usernumber); /* Set terminaltype */
      do_set(8,0,0,0,user.usernumber); /* Set pagelength */
      do_set(7,0,0,0,user.usernumber); /* Set more? */
      do_set(9,0,0,0,user.usernumber); /* Set cls */
      }
    while(!port_yesno("\012Is this all OK? "));

    do
      {  
      bbs_sendfile("<ARCbbs$text>.Password",0,0);
      port_readline(tempbuffer,255,HIDE|NOBOX); port_crlf();
      user.passcrc=calcrc(tempbuffer,strlen(tempbuffer));

      port_txstring("Reenter for confirmation.\012: ",1);
      port_readline(tempbuffer,255,HIDE|NOBOX); port_crlf();
      tmp=calcrc(tempbuffer,strlen(tempbuffer));

      if (tmp!=user.passcrc)
        {
        port_txstring("Passwords do not match (remember they are case sensitive!). Try again.\012",1);
        }
      }
    while(tmp!=user.passcrc);

    /* Set first logon time */
    user.t_firstlogon=user.t_lastlogon=time(NULL);
    user.m_message=user.m_file=read_msgnumber();
  
    /* Setup other system flags */
    user.mailpointer_start=user.mailpointer_end=user.morepointer=-1;
    user.mailcount=0; user.logons=user.timetoday=0;
    user.filebase=user.conference=-1;

    /* Find a free usernumber */
    user.usernumber=usernumber=find_free();

    /* Write user data/hash data */
    put_user(&user); hash_write(user.username,user.usernumber);

    /* Send specified user a message about this new user */
    if (strlen(getenv("ARCbbs$newmail"))>0)
      {
      mail_block header; user_block tempuser;
                                             
      setup_mailheader(&header,&user,0);
      header.to_id=tempuser.usernumber=atoi(getenv("ARCbbs$newmail"));
      get_user(&tempuser);
      if (tempuser.status!=0)
        {                          
        char *txtpointer=message_buffer;

        strcpy(header.to,tempuser.username);

        /* Set subject */
        strcpy(header.subject,"New user");
                       
        txtpointer+=(1+sprintf(txtpointer,"Realname : %s",user.realname));
        txtpointer+=(1+sprintf(txtpointer,"First logged on at %d baud",baudrate));

        header.message_length=(txtpointer-message_buffer);

        /* Add entry to log */
        syslog("NewUser","New user '%s' (realname '%s')",user.username,user.realname);

        if (put_message(&header,message_buffer)!=OK)
          {
          mprintf(1,"\012\012Can't inform ARCbbs$newmail user of new user logon\012\012");
          }
        }
      }          
    }
  else
    {
    int passcrc;

    /* Get password */
    if (user.passcrc)
      {   
      port_txstring("Password: ",1);
      port_readline(tempbuffer,255,HIDE|NOBOX);
      port_crlf();
      passcrc=calcrc(tempbuffer,strlen(tempbuffer));
      if (passcrc!=user.passcrc)
        {
        port_txstring("Invalid password. Remember passwords are case-sensitive.\012",1);

        syslog("BadPW","User '%s', attempt '%s'",user.username,tempbuffer);

        if (++tries==3)
          {
          /* Send message & throw them off! */
          syslog("Illegal","User '%s' - disconnected after 3 bad passwords",user.username);

          port_txstring("Illegal access attempted",1);
          wimp_waitfor(200);
          return(0);
          }
        goto getnameagain;
        }
      }
    }
    
  /* Log serial number if in use */
  if (strcmp(getenv("ARCterm$security"),"yes")==0 && serial!=-1)
    {
    FILE *a;

    if ((a=fopen("<ARCserver$Dir>.7users","a"))!=NULL)
      {
      time_t now;
      
      time(&now);
      fprintf(a,"%s Serial %5d : #%04d %s\n",cctime(&now),50000-serial,usernumber,user.username);
      fclose(a);
      }
    }

  /* Sucessful logon, print logon banner, news, etc */
  send_message(servertask,BBS_ICONINFO,portnumber,usernumber,baudrate,0);
  send_strmessage(servertask,BBS_USERID,portnumber,0,0,user.username);
  online_users[portnumber].usernumber=usernumber;
  strcpy(online_users[portnumber].username,user.username);

  /* Set logon time & last logon time */
  logon_time=clock(); last_logon=user.t_lastlogon;

  /* Check userlevel */
  if (!user.userlevel)
    {
    port_txstring("\012Logons barred for this account\012",1);
    wimp_waitfor(200);
    return(0);
    }

  /* Setup time allocated for this logon */
  gtime(&event,now=time(NULL)); gtime(&nowtm,now);
  event.tm_sec=0;  

  /* Event */
  strcpy(eventtime,getenv("ARCbbs$EventTime"));
  if (sscanf(eventtime,"%d:%d",&event.tm_hour,&event.tm_min)!=2)
    {
    event.tm_hour=2; event.tm_min=30;
    }                                          

  if (nowtm.tm_hour>event.tm_hour ||
      (nowtm.tm_hour==event.tm_hour && nowtm.tm_min>event.tm_min))
    {               
    event.tm_mday++;
    }                

  /* Make number of minutes until event */
  untilevent=(mktime(&event)-now)/60;  
                                          
  /* Have we logged on before today? / Reset Time flag */
  if ((user.f_user&USER_RESETTIME)!=0)
    {
    user.timetoday=0; put_user(&user);
    }
  else
    {
    gtime(&laston,user.t_lastlogon);
    if (nowtm.tm_yday!=laston.tm_yday || nowtm.tm_year!=laston.tm_year)
      {
      /* Different day from last logon, reset timetoday figure */
      user.timetoday=0; put_user(&user);
      }
    }

  thislogon=(user.timeallowed-user.timetoday);
  if (thislogon<1)
    {
    mprintf(1,"Sorry, you have used all your allocated time for today, call back tomorrow.\012");
    wimp_waitfor(200);
    return(0);
    }                   
   
  /* Set timeouts OK */
  notimeout=0;

  /* Adjust incase fido event is near */
  if (slavemode=='m') if (mail_event<untilevent) untilevent=mail_event;

  if (untilevent<thislogon)
    {
    thislogon=untilevent;
    mprintf(1,"Event near, your time for this logon shortened to %d minutes\012",thislogon);
    }
  else
    {
    mprintf(1,"%d minutes of logon time left today.\012",thislogon);
    }

  /* Update last logon information */
  user.t_lastlogon=time(NULL); user.logons++; readnew=readnewfiles=0;
  put_user(&user);
               
  /* Page length trap */
  if (user.pagelen==0) user.pagelen=24;
          
  /* Setup conferences */
  if (user.conference>0)
    {
    read_area(conference=user.conference,&temparea);
    strcpy(conferencename,temparea.name);
    }
  else
    {
    strcpy(conferencename,"Undefined");
    }

  if (user.filebase>0)
    {
    read_area(filebase=user.filebase,&temparea);
    strcpy(filebasename,temparea.name);
    }
  else
    {
    strcpy(filebasename,"Undefined");
    }
 
  loggedon=1;        

  /* Set menu path */
  strcpy(menupath,"Logon");
  strcpy(currentmenu,"main");
  return(1);
  }
