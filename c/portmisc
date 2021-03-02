/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Additional port commands header             <]
Current version   [> 00.67                                       <]
Version date      [> 19-February-1993                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1989-1993 by    <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"
#include "config.h"          /* System configuration */
#include "servmess.h"        /* Server message info */
#include "port.h"            /* Port driver functions */
#include "userlog.h"         /* Userlog format */
#include "parser.h"          /* Command parser */
#include "mail.h"            /* Mail format */
#include "portmisc.h"        /* Misc port commands */
#include "servcomm.h"        /* Misc server comm commands */
#include "mprintf.h"         /* Modem printf */
#include "modcomm.h"         /* Module interface */
#include "version.h"

extern _address_arch ouraddress;
extern FILE *bbs_file[10];
extern void moontxt(char*,int,int,int),today(void);
extern char *cookie(void),*timetxt(char*,int,int,int,int),
            *datetxt(char*,int,int,int);

extern int  portnumber,got_code,got_code2,got_data,baudrate,thislogon,
            bbs_currenttask,ackdivert,lastbaud,lasttime,muchatting,
            chatting,notimeout,conference,filebase,beenchatting;
extern user_block user;
extern char tempbuffer[256],lastcaller[31],currentmenu[20],menupath[80],
            conferencename[61],filebasename[61],aborts[64];
extern jmp_buf jmp_carrier,jmp_tomenu;
extern clock_t logon_time,answer_time;
extern time_t last_logon;
  
static char token[20];

static char tokenlist[][20]=
  {
  "fg","bfg","bg","bold","flash","eol","eos","cls","std","fgbg","bfgbg",
  "anykey",
  ""
  };

void get_stoken(char **tok)
  {
  char *t=*tok,*p=token;

  while(*t==' ') t++;
  if (*t!='}' && *t)
    {
    while(*t!=' ' && *t!='}' && *t) *p++=*t++;
    }
  *p=0;
  *tok=t;
  }

void port_txs(char *string)
  {
  while(*string) port_txw(*string++);
  }
                             
int colourcode(char a)
  {
  static char colours[9]="nrgybmcw";
  char *c=strchr(colours,a);

  if (c==NULL) return(0); else return(c-colours);
  }

void parse_system(char *s,int interrupt)
  {
  s++;
  switch (s[0])
    {
    case '|':
      {
      port_crlf();
      break;
      }
    case '>':
      {
      int x,y;
      char sbuff[3];

      sbuff[2]=0;
      sbuff[0]=s[1];
      sbuff[1]=s[2];
      x=atoi(sbuff);
      sbuff[0]=s[3];
      sbuff[1]=s[4];
      y=atoi(sbuff);
                                                     
      switch(user.termtype)
        {
        case TER_VT100:
        case TER_ANSI:
          {
          mprintf(interrupt,"\033[%d;%dH",y+1,x+1);
          break;
          }           
        case TER_VT52:
          {
          mprintf(interrupt,"\033Y%c%c",y+32,x+32);
          break;
          }
        case TER_TTY:
          {   
          port_txw(30); port_txw(10); port_txw(8);
          while(y--) port_txw(10);
          if (x)
            {
            x=79-x;
            while(x--) port_txw(8);
            }
          break;
          }
        }
      break;
      }
    case 'i':
      {
      switch(s[1])
        {
        case '0':
          {
          mprintf(interrupt,"ARCbbs v%s (%s %s) (c)1989/1990/1991 Hugo Fiennes",
                  version_number,version_time,version_date);
          break;
          }
        case '1':
          {
          mprintf(interrupt,"%d",baudrate);
          break;
          }
        case '2':
          {
          mprintf(interrupt,"%d",portnumber);
          break;
          }
        case '3':
          {
          mprintf(interrupt,"%d",mod_readcallcount());
          break;
          }
        case '4':
          {
          /* Send request for BBS status */
          send_message(servertask,BBS_STATUS,portnumber,0,0,0);
    
          /* Set flag */
          got_data=0;
    
          /* Reply will arrive soon */
          while(!got_data) window_poll();
    
          mprintf(interrupt,"%d",got_code);
          break;
          }
        case '5':
          {
          /* Print last caller note */
          mprintf(interrupt,lastcaller); break;
          }
        case '6':
          {
          /* Print last baud rate */
          mprintf(interrupt,"%d",lastbaud); break;
          }
        case '7':
          {
          /* Print last time on */
          mprintf(interrupt,"%d:%02d",lasttime/60,lasttime%60); break;
          }    
        case '8':
          {
          /* Print currently select conference # */
          mprintf(interrupt,"%d",conference);
          break;
          }
        case '9':
          {
          /* Print currently selected conference */
          mprintf(interrupt,conferencename); 
          break;
          }
        case 'a':
          {
          /* Print currently selected filebase # */
          mprintf(interrupt,"%d",filebase);
          break;
          }
        case 'b':
          {
          /* Print currently selected conference */
          mprintf(interrupt,filebasename); 
          break;
          }
        case 'c':
          {
          /* Print nodenumber */
          mprintf(interrupt,"%d:%d/%d.%d",ouraddress.zone,ouraddress.net,
                  ouraddress.node,ouraddress.point);
          break;
          }
        }
      break;
      }
    case '%':
      {
      port_txw('%'); break;
      }
    case 'u':
      {
      switch(s[1])
        {
        case '0':
          {
          port_txstring(user.username,interrupt); break;
          }
        case '1':
          {
          char firstname[31];
          if (sscanf(user.username,"%s ",&firstname)==0)
            strcpy(firstname,user.username);
          port_txstring(firstname,interrupt); break;
          }
        case '2':
          {
          char firstname[31];
          if (sscanf(user.username,"%*s %s",&firstname)==0)
            strcpy(firstname,user.username);
          port_txstring(firstname,interrupt);
          break;
          }
        case '3':
          {
          mprintf(interrupt,"%d",user.logons); break;
          }
        case '4':
          {
          mprintf(interrupt,"%d",user.uploads); break;
          }
        case '5':
          {
          mprintf(interrupt,"%d",user.downloads); break;
          }
        case '6':
          {
          mprintf(interrupt,"%d",user.ratio); break;
          }
        case '7':
          {
          mprintf(interrupt,"%x",user.userlevel); break;
          }
        case '8':
          {
          mprintf(interrupt,"%d",user.mailcount); break;
          }
        case 'c':
          {
          /* Print callrate */
          mprintf(interrupt,"%c",user.callrate); break;
          }
        case 's':
          {
          /* Print screen length */
          mprintf(interrupt,"%d",user.pagelen+1); break;
          }
        case 't':
          {
          /* Print terminal type */
          switch(user.termtype)
            {
            case TER_TTY:
              mprintf(1,"TTY"); break;
            case TER_VT52:
              mprintf(1,"VT52"); break;
            case TER_VT100:
              mprintf(1,"VT100"); break;
            case TER_ANSI:
              mprintf(1,"ANSI"); break;
            }
          break;
          }
        }
      break;
      }
    case 'r':
      {
      port_txstring(user.realname,interrupt); break;
      }
    case 'a':
      {
      if (s[1]<'5')
        {
        port_txstring(user.address[s[1]-'1'],interrupt);
        }
      else
        {
        port_txstring(user.postcode,interrupt);
        }
      break;
      }
    case 'p':
      {
      port_txstring(user.telephone,interrupt); break;
      }
    case '#':
      {
      mprintf(interrupt,"%d",user.usernumber);
      break;
      }
    case 's':
      {
      port_txstring(SYSTEMNAME,interrupt); break;
      }
    case 'c':
      {
      port_txstring(cookie(),interrupt+16); break;
      }
    case 'm':
      {
      struct tm *moontime;
      time_t now=time(NULL);
    
      moontime=localtime(&now);
      moontxt(tempbuffer,1900+moontime->tm_year,1+moontime->tm_mon,moontime->tm_mday);
      port_txstring(tempbuffer,interrupt); break;
      break;
      }
    case 'n':
      {
      mprintf(interrupt,"%d",user.mailcount);
      break;
      }
    case 'l':
      {
      os_regset r;
      /* Task loading of machine */
      r.r[0]=0;
      os_swi(0x400f2,&r);     /* Wimp_ReadSysInfo */
      mprintf(interrupt,"%d",r.r[0]); 
      break;
      }
    case 't':
      {
      switch (s[1])
        {
        case 'o':
          {
          today(); break;
          }
        case 'f':
          {
          showtime(user.t_firstlogon); break;
          }
        case 'l':
          {
          showtime(last_logon); break;
          }
        case 'n':
          {
          showtime(time(NULL)); break;
          }
        case '0':
          {
          struct tm *extime;
          time_t now=time(NULL);
    
          extime=localtime(&now);
          datetxt(tempbuffer,1900+extime->tm_year,1+extime->tm_mon,extime->tm_mday);
          port_txstring(tempbuffer,interrupt); break;
          }
        case '1':
          {
          struct tm *extime;
          time_t now=time(NULL);
    
          extime=localtime(&now);
          timetxt(tempbuffer,extime->tm_hour,extime->tm_min,
                         extime->tm_sec,0);
          port_txstring(tempbuffer,interrupt); break;
          }
        case '2':
          {
          struct tm *extime;
          time_t now=time(NULL);
    
          extime=localtime(&now);
          timetxt(tempbuffer,extime->tm_hour,extime->tm_min,
                         extime->tm_sec,0101010);
          port_txstring(tempbuffer,interrupt); break;
          }
        case '3':
          {
          int perc=(clock()-logon_time)/(thislogon*60);
    
          mprintf(interrupt,"%d",perc);
          break;
          }
        case '4':
          {
          int perc=(clock()-logon_time)/(thislogon*60);
    
          mprintf(interrupt,"%d",100-perc);
          break;
          }
        case '5':
          {
          int tme=(clock()-logon_time)/100;
          mprintf(interrupt,"%2d:%02d",tme/60,tme%60);
          break;
          }
        case '6':
          {
          int tme=(thislogon*60)-((clock()-logon_time)/100);
          mprintf(interrupt,"%2d:%02d",tme/60,tme%60);
          break;
          }
        }
      break;
      }
    }
  }  

void parse_command(char **commandp,int interrupt)
  {
  int a,b; char *c=*commandp;

  if (*c=='{')
    {
    port_txw('{');
    *commandp=++c;
    return;
    }

  do
    {
    get_stoken(commandp);
    if (token[0]==0) return;
    if (token[0]=='|') { port_crlf(); continue; }
    if (token[0]=='_') { parse_system(token,interrupt); continue; }

    a=0; while(tokenlist[a][0]!=0 && strcmp(tokenlist[a],token)!=0) a++;
    if (tokenlist[a][0])
      {
      switch(a)
        {
        case 0:   /* fg */
          get_stoken(commandp);
          if (user.termtype==TER_ANSI)
            {
            port_txs("\033[3"); port_txw('0'+colourcode(*token)); port_txw('m');
            }
          break;
        case 1:   /* bfg */
          get_stoken(commandp);
          if (user.termtype==TER_ANSI)
            {
            port_txs("\033[1;3"); port_txw('0'+colourcode(*token)); port_txw('m');
            }
          if (user.termtype==TER_VT100) port_txs("\033[1m");
          break;
        case 2:   /* bg */
          get_stoken(commandp);
          if (user.termtype==TER_ANSI)
            {
            port_txs("\033[4"); port_txw('0'+colourcode(*token)); port_txw('m');
            }
          break;
        case 3:  /* bold */
          if (user.termtype==TER_VT100 || user.termtype==TER_ANSI) port_txs("\033[1m");
          break;
        case 4:  /* flash */
          if (user.termtype==TER_VT100 || user.termtype==TER_ANSI) port_txs("\033[5m");
          break;
        case 5:  /* eol */
          if (user.termtype==TER_VT100 || user.termtype==TER_ANSI) port_txs("\033[0K");
          break;
        case 6:  /* eos */
          if (user.termtype==TER_VT100 || user.termtype==TER_ANSI) port_txs("\033[0J");
          break;
        case 7:  /* cls */
          if (user.f_user&USER_NOCLEAR) break;
          switch(user.termtype)
            {
            case TER_VT100:
            case TER_ANSI:
              port_txs("\033[H\033[2J");
              break;
            case TER_VT52:
              port_txs("\033Y  \033J");
              break;
            case TER_TTY:
              port_txw(12);
              break;
            }
          break;
        case 8:  /* std */
          if (user.termtype==TER_VT100 || user.termtype==TER_ANSI) port_txs("\033[0m");
          break;
        case 9:  /* fgbg */
          get_stoken(commandp);
          if (user.termtype==TER_ANSI)
            {
            port_txs("\033[3"); port_txw('0'+colourcode(token[0]));
            port_txs(";4"); port_txw('0'+colourcode(token[1]));
            port_txw('m');
            }
          break;
        case 10:  /* bfgbg */
          get_stoken(commandp);
          if (user.termtype==TER_ANSI)
            {
            port_txs("\033[1;3"); port_txw('0'+colourcode(token[0]));
            port_txs(";4"); port_txw('0'+colourcode(token[1]));
            port_txw('m');
            }
          if (user.termtype==TER_VT100) port_txs("\033[1m");
          break;
        case 11: /* anykey */
          port_get();
          break;
        }
      }
    }
  while(1);
  }

int port_txstring(char *buffer,int interrupt)
  {
  int abort=0,noc=interrupt&16;
                
  /* Mask out nocommand flag */
  interrupt&=15;
                       
  if (interrupt!=1 && port_rxbuffer()!=0) return(port_rxbuffer());

  while(*buffer && !abort)
    {
    if (*buffer==10) 
      {
      port_crlf(); buffer++; 
      }
    else
      {
      switch(*buffer++)
        {
        case '{':
          {
          if (!noc)
            {
            parse_command(&buffer,interrupt);
            if (*buffer=='}') buffer++; else continue;
            }
          else port_txw('{');
          break;
          }
        default:
          {
          port_txw(*(buffer-1));
          break;
          }
        }
      }

    if (interrupt==0) if (port_rxbuffer()) { port_txclear(); return(1); }
    }
    
  return(0);
  }

void port_txcstring(char *buffer,int length)
  {
  while(length)
    {
    port_txw(*buffer++); length--;
    }
  }

int port_yesno(char *text)
  {
  int answer;

  /* Send prompt */
  port_txstring(text,1);

  /* Get input */
  do
    {
    answer=toupper(port_get());
    }
  while(answer!='Y' && answer!='N');

  answer=(answer=='Y');
  if (answer)
    port_txstring("Yes\012",1);
  else
    port_txstring("No\012",1);

  return(answer);
  }

int port_get()
  {
  clock_t timer,lastbeep;
  int beeping,d;

  again:
  timer=clock(); lastbeep=beeping=0;

  while((clock()-timer)<INACTIVITY)
    {  
    if (portnumber>=9) timer=clock();

    if ((clock()-logon_time)>(thislogon*6000) && thislogon!=0 && notimeout==0)
      {                                                                   
      port_txstring("\012\012Sorry, but your time is up for today!\012\012",1);
      bbs_currenttask=9; thislogon=0;

      if (muchatting)
        {
        send_message(servertask,BBS_MUCONTROL,portnumber,MU_LEAVE,0,0);
        muchatting=0;
        }

      if (chatting)
        {
        send_message(servertask,BBS_CHATREQUEST,portnumber,NO_CHATTING,0,0);
        chatting=0;
        }

      strcpy(currentmenu,"timeup");
      strcpy(menupath,"Logon");
      port_crlf();
      longjmp(jmp_tomenu,0);
      }

    if ((timer-answer_time)>(thislogon*6000) && notimeout==-1)
      {                                                                   
      longjmp(jmp_carrier,1);
      }

    if ((clock()-timer)>(INACTIVITY/2) && beeping==0 && (thislogon!=0 ||
        strcmp(currentmenu,"timeup")==0)) beeping=1;
    if (beeping && (clock()-lastbeep)>500)
      {
      /* Beep at user every 5 seconds */
      port_txw(7); lastbeep=clock();
      }

    if ((d=port_rx())>=0) return(d);

    if (ackdivert)
      {
      ackdivert=0;
      port_crlf();
      longjmp(jmp_tomenu,0);
      }

    window_poll();
    
    if (beenchatting)
      {
      beenchatting=0; timer=clock();
      }
    }

  if (thislogon==0 && strcmp(currentmenu,"timeup")!=0) goto again;

  /* Inactivity timeout */
  port_txstring("\012\012\012Inactivity timeout. Dropping line.\012\012",1);
  longjmp(jmp_carrier,1);

  return(0); /* For the compiler */
  }

int port_readchar(int flags)
  {
  int data=port_get();

  if ((flags & NOECHO)==0)
    {
    /* We can echo something */
    if (flags & HIDE) port_txw('-'); else port_txw(data);
    }
  return(data);
  }

void port_readline(char *buffer,int length,int flags)
  {
  int bpointer,quitloop=0,accept;

  if ((flags & NOBOX)==0) port_field(length);

  if (flags & EXISTING)
    {
    bpointer=strlen(buffer);
    port_txstring(buffer,16+1);
    }
  else
    {
    bpointer=0;
    }

  do
    {
    int ch=port_get();
   
    if (ch>=32 && ch!=127)
      {
      if (bpointer<((flags&NOTERM)?(length-1):length))
        {
        accept=1;

        if ((flags & NUMONLY) && (ch<'0' || ch>'9')) accept=0;
         
        if (accept)
          {
          if (flags & TOUPPER) ch=toupper(ch);
          if (flags & TOLOWER) ch=tolower(ch);
          buffer[bpointer++]=ch;
          if ((flags & NOECHO)==0)
            {
            if (flags & HIDE) port_txw('-'); else port_txw(ch);
            }
          }
        }
      else
        {
        if (flags & NOTERM)
          {
          buffer[bpointer++]=ch;
          quitloop=1;
          break;
          }
        }
      }
    else
      {
      switch(ch)
        {
        /* Backspace/delete */
        case 8:
        case 127:
          {
          if (bpointer)
            {
            bpointer--;
            if ((flags & NOECHO)==0) port_txstring("\010 \010",1);
            }
          break;
          }
        /* End of line ? */
        case 13:
        case 10:
          {
          quitloop=1;
          break;
          }
        }
      }
    }
  while (!quitloop);
  buffer[bpointer]=0;
  if (user.termtype==TER_ANSI && (flags & NOBOX)==0)
    {
    if ((flags & ENDBOX)!=0)
      {
      port_txstring("{eol std}",1);
      }
    else
      {
      port_txstring("{std}",1);
      }
    }
  }

void port_crlf(void)
  {
  port_txw(13); port_txw(10);
  }

int bbs_sendfile(char *filename,char *interrupts2,int flags)
  {
  int abort=0,line=0,nonstop=(flags&SENDFILE_NONSTOP);
  char linebuff[512],interrupts[40],*i2=interrupts2,*i=interrupts;
         
  if (i)
    {
    while(*interrupts2) *i++=tolower(*interrupts2++);
    *i=0;
    }

  if ((user.f_user&USER_NOMORE)!=0) nonstop=1;

  if ((bbs_file[0]=fopen(filename,"r"))==NULL)
    {
    if (flags&SENDFILE_ERROR) mprintf(1,"\012Cannot find file %s\012",filename);
    }
  else
    {
    do
      {
      if (fgets(linebuff,511,bbs_file[0]))
        {
        line++;                          
        
        port_txstring(linebuff,1);

        if (!port_rxbuffer())
          {
          if (line==23 && nonstop==0)
            {
            int ch;
  
            line=0;
            mprintf(1,"More: [Return] continues, [N]onstop, [Ctrl-C] aborts\015");
            do
              {
              switch(ch=tolower(port_get()))
                {
                case 3:   abort=3; break;
                case 'n': nonstop=1; break;
                }
              }
            while(ch!=32 && ch!=13 && ch!=3 && ch!='n');
            mprintf(1,"                                                            \015");
            }
          }
        else
          {
          int ch;
          
          if (i2)
            {
            if ((ch=port_examinerx())!=-1)
              {
              if (strchr(interrupts,tolower(ch))!=NULL)
                {
                fclose(bbs_file[0]); bbs_file[0]=0;
                return(1);
                }
              else port_rx();
              }
            }
          else
            {
            port_txclear(); port_crlf();
            fclose(bbs_file[0]); bbs_file[0]=0;
            return(1);
            }
          }
        }
      }
    while(!feof(bbs_file[0]) && abort==0);
    fclose(bbs_file[0]); bbs_file[0]=0;
    }   

  if (abort) port_crlf();
  return(abort);
  }

void showtime(time_t stime)
  {
  port_txstring(cctime(&stime),1);
  }

char *cctime(time_t *atime)
  {
  time_t pass; char *ptr;

  pass=*atime; ptr=ctime(&pass);
  ptr[24]=0;
  return(ptr);
  }

void port_field(int len)
  {
  int a;

  if (user.termtype!=TER_ANSI) return;

  port_txstring("{bfgbg wb}",1);
  for(a=0;a<len;a++) port_txw(32);
  for(a=0;a<len;a++) port_txw(8);
  }
