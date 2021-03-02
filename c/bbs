/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs/Archimedes                           <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Slave module                                <]
Current version   [> 01.64                                       <]
Version date      [> 02-May-1993                                 <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1989-1993 by    <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

/* 16kB root stack force */  
int __root_stack_size=16*1024;

#define KEYIN event->data.key.chcode
#define MAINMODULE
#define timepoll if((clock()-lastpoll)>MAXIMUM_POLL)window_poll

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
#include "modcomm.h"         /* Module support */
#include "fido.h"            /* Fidonet interface */
#include "heap.h"            /* Heap */
#include "miscbbs.h"         /* BBS misc routines */
#include "alarm.h"           /* Alarms for cursors */
#include "menu.h"            /* Menus */
#include "version.h"
#include "bbs.h"        
#include "scr.h"             /* ANSI terminal */
#include "driver.h"          /* Block driver */
#include "doors.h"           /* Door support */
#include "doorsa.h"
#include "script.h"          /* Script language */
#include "logon.h"           /* User logon */
#include "str.h"             /* String handling */

extern void questionnaire(char*),key_insert(int),message_upload(char*,int);
extern int  port_outsize;

/* Driver blocks */
extern int  (*driver_load(int*,char*))(int,...);
int driver_block[1024];

/* WIMP stuff */
wimp_w  local_handle;
extern int *vdusprites,savednr;
extern char *vdusprite;

/* PreDeclarations */
int  main(int,char**),flag_process(char*);

void errorlog(os_error*,char*),window_init(void),
     bbs_dostuff(void),window_poll(void),
     wimp_waitfor(int),do_command(char*,int),open_swindow(wimp_w,int,int),
     conf_join(unsigned char*,int),conf_resign(unsigned char*,int),
     queue_show(void),order_disk(void),slave_msgproc(wimp_eventstr*,void*),
     do_local_menu(void *,char*);

int  conf_member(unsigned char*,int);

char *file_pathname(int);

/* Global variables */
int     portnumber,bbs_currenttask=0,enablecarrier=0,chatting=0,
        baudrate=0,usernumber=0,online=0,got_data,got_code,
        muchatting=0,oldx=0,oldy=0,snoop=0,readnew=0,maxspeed_tx,files_done,
        readnewfiles=0,download_queue[64],download_queuesize[64],
        queue_length=0,got_code3,thislogon,nicelogoff,got_type,
        got_code2,msginterrupt=0,ackdivert=0,
        setbaud=1,lastbaud=0,lasttime=0,loggedon=0,notimeout=0,
        scratchpad=0,conference=-1,filebase=-1,beenchatting=0,xoff=0,
        noof_filed,arq=0,gateway=0,gatewayport,lastpoll=0,
        mail_event,slavemode=0;
static int quit=0;
mail_block *lastmsg;
wimp_t  servertask=0,ourtask; 
jmp_buf jmp_wimpexit,jmp_carrier,jmp_tomenu;
FILE    *bbs_file[10]={ 0,0,0,0,0,0,0,0,0,0 };

FILE    *infile=NULL;
menu    local_menu;
char    tempbuffer[256],tempbuffer2[256],currentmenu[20],noaccess[80],
        whitesp[]=" \t",breakch[]=";,",quote[]="'\"",menu_options[32],
        menu_replies[32][32],menu_commands[32][40],
        got_string[81],download_queuename[64][21],
        menu_prompt[256],menupath[80],menu_available[32],
        menu_elsecommands[32][40],welcomename[80],welcomepassword[20],
        mumsg[100],
        lastcaller[31]={ "Port initialised" },
        conferencename[61]= { "Unselected" },
        filebasename[61]= { "Unselected" },
        __message_buffer[16384],
        *_message_buffer,
        *message_buffer;
user_block user;        
user_list *online_users;
clock_t answer_time,logon_time;
time_t last_logon;

char    commands[][14]= {    "GOTO", /* GOTO <menuname>                     */
                           "LOGOFF", /* LOGOFF                              */
                             "DROP", /* DROP                                */
                          "RELOGON", /* RELOGON                             */
                             "TYPE", /* TYPE <filename>                     */
                           "TYPE_F", /* TYPE_F <filename>                   */
                           "ONLINE", /* ONLINE                              */
                             "READ", /* READ <messagebase>                  */
                            "WRITE", /* WRITE <messagebase> [<userid>]      */
                           "UPLOAD", /* UPLOAD <filebase>                   */
                         "DOWNLOAD", /* DOWNLOAD <filebase>                 */
                            "DOING", /* DOING <string>                      */
                         "FILELIST", /* FILELIST <filebase> <SHORT|LONG>    */
                             "PAGE", /* PAGE                                */
                             "CHAT", /* CHAT <channel>                      */
                        "CHECKMAIL", /* CHECKMAIL [<userid>]                */
                         "SETFLAGS", /* SETFLAGS <userflags>                */
                         "CLRFLAGS", /* CLRFLAGS <userflags>                */
                         "TOGFLAGS", /* TOGFLAGS <userflags>                */
                           "GOTO_P", /* GOTO_P <password> <menuname>        */
                           "PROMPT", /* PROMPT <string>                     */
                        "SHOWQUEUE", /* SHOWQUEUE                           */
                              "SET", /* SET <option>                        */
                             "PATH", /* PATH <menupath>                     */
                             "ECHO", /* ECHO <string>                       */
                         "NOACCESS", /* NOACCESS <string>                   */
                               "IF", /* IF [F<flags>|L<><level>|U<user>]    */
                                     /*                          <command>  */
                              "CLI", /* CLI                                 */
                            "UTILS", /* UTILS                               */
                             "HELP", /* HELP                                */
                                "?", /* ?                                   */
                         "CALLRATE", /* CALLRATE <L|a|b>                    */
                         "CALLCOST", /* CALLCOST                            */
                      "DOING_ALERT", /* DOING_ALERT <string>                */
                            "ALERT", /* ALERT                               */
                             "TIME", /* TIME [+value|-value|value]          */
                          "GOTO_IP", /* GOTO_IP <password> <menuname>       */
                          "NEWSCAN", /* NEWSCAN <FILE|MESSAGE>              */
                     "FILESEARCH_G", /* FILESEARCH_G                        */
                       "FILESEARCH", /* FILESEARCH <filearea>               */
                        "S_MESSAGE", /* S_MESSAGE                           */
                            "S_ARC", /* S_ARC [PC|ARC]                      */
                       "S_DOWNLOAD", /* S_DOWNLOAD                          */
                           "ANYKEY", /* ANYKEY                              */
                      "CONF_MEMBER", /* CONF_MEMBER <FILE|MESSAGE>          */
                        "CONF_LIST", /* CONF_LIST <FILE|MESSAGE>            */
                        "CONF_JOIN", /* CONF_JOIN <FILE|MESSAGE> [<confnr>] */
                      "CONF_RESIGN", /* CONF_RESI <FILE|MESSAGE> [<confnr>] */
                      "CONF_SELECT", /* CONF_SELE <FILE|MESSAGE> [<confnr>] */
                         "USERLIST", /* USERLIST                            */
                       "CLEARQUEUE", /* CLEARQUEUE                          */
                        "DOWNQUEUE", /* DOWNQUEUE                           */
                         "QUESTION", /* QUESTION <filename>                 */
                        "ORDERDISK", /* ORDERDISK                           */
                               "SE", /* SE                                  */
                             "CALL", /* CALL <portnr>                       */
                          "SETMAIL", /* SETMAIL                             */
                         "SETSPEED", /* SETSPEED <portnr> <speed>           */
                          "NEWMENU", /* NEWMENU <menufile>                  */
                        "SCANFORME", /* SCANFORME                           */
                             "DOOR", /* DOOR <doornr>                       */
                            "OSCLI", /* OSCLI <command>                     */
                           "SCRIPT", /* SCRIPT <name>                       */
                                " " };

char        sets[][12]= {"PASSWORD", /* PASSWORD                            */
                         "TERMINAL", /* TERMINAL                            */
                            "RATIO", /* RATIO <value>                       */
                             "TIME", /* TIME <value>                        */
                          "UPLOADS", /* UPLOADS [+value|-value|value]       */
                        "DOWNLOADS", /* DOWNLOADS [+value|-value|value]     */
                        "USERLEVEL", /* USERLEVEL <value>                   */
                             "MORE", /* MORE [ON|OFF]                       */
                       "PAGELENGTH", /* PAGELENGTH <value>                  */
                      "CLEARSCREEN", /* CLEARSCREEN [ON|OFF]                */
                                " " };

    /* function to remove any temporary files */

void tidy_up()
    {
    char fname[40]; int a;

    for (a=0;a<10;a++) _fclose(&bbs_file[a]);

    sprintf(fname,"<ARCbbs$Temp>.Batch_%d",portnumber);
    remove(fname);
    sprintf(fname,"<ARCbbs$Temp>.Scratch_%d",portnumber);
    remove(fname);
    sprintf(fname,"<ARCbbs$Temp>.Temp_%d",portnumber);
    remove(fname);
    sprintf(fname,"<ARCbbs$Temp>.Msg_%d",portnumber);
    remove(fname);

    script_closedown(FALSE);
    }

int main(int argc,char *argv[])
  {
  int quit;
        
  if (argc!=3) return(0);
        
  _message_buffer=__message_buffer;
  message_buffer=(_message_buffer+9);

  /* Read flags */
  flag_initialise();

  /* Reset doors */
  doors_reset();

  /* Read port number from environment string */
  portnumber=atoi(argv[2]);

  /* Get slave mode */
  slavemode=tolower(*argv[1]);

  /* Read modem config */
  modem_readconfig();

  /* Load port driver */
  if ((driver=driver_load(driver_block,driver_name))==NULL)
    {
    werr(1,"Can't load block driver '%s'",driver_name);
    }

  /* Initialise the driver */
  (*driver)(DRIVER_INITIALISE,driver_port);
  (*driver)(DRIVER_POLL,driver_port);
  (*driver)(DRIVER_WORDFORMAT,driver_port,0);

  port_txclear(); port_outsize=port_txbuffer();
  atexit(tidy_up);

  /* Setup fidonet */
  fido_setup();

  /* Initialise wimp */
  window_init();
                        
  /* Initialise script */
  script_initialise();

  /* Wait for server id */
  while(servertask==0) window_poll();

  online_users=(void*)mod_getstatuspointer();           

  /* Send message to clear baud rate, etc */
  send_message(servertask,BBS_ICONINFO,portnumber,0,0,0);
  online_users[portnumber].baudrate=-1;                                
  online_users[portnumber].username[0]=0;

  /* Increment 'other' count */
  send_message(servertask,BBS_COUNTOTHER,1,0,0,0);

  if ((quit=setjmp(jmp_wimpexit))==0)
    {
    do
      {
      /*** Deal with call */
      bbs_dostuff();
      }
    while(slavemode!='m');
    }

  /* Blank out in user table */
  online_users[portnumber].baudrate=-1;                                
  online_users[portnumber].username[0]=0;

  /* If script was running, kill it */
  if (script_cankill) script_bomb();

  /*** Mailports exit after call */
  if (slavemode=='m')
    {
    /* If it was a mail call, then exit as call is over */
    port_dtr(0); port_rts(0); port_xonxoff(0);
    wimp_waitfor(100); port_rxclear();
    port_dtr(1); port_rts(1);
    longjmp(jmp_wimpexit,2);
    }

  /* Tidy up any temporary files */
  tidy_up();

  wimp_closedown();
  return(0);
  }

void window_init()
  {
  static char buffer[20],buffer2[40];
  wimp_wind *window;

  /* Start our task up */
  sprintf(buffer,"ARCbbs_%d%c",portnumber,toupper(slavemode));
  wimpt_init(buffer);             /* Setup task */
  res_init("ARCbbs");

  alarm_init(); template_init(); dbox_init();
  ourtask=wimpt_task();

  /* Create the window */
  window=template_syshandle("local");
  wimp_create_wind(window,&local_handle);
  win_activeinc();

  win_register_event_handler(local_handle,slave_msgproc,0);
  win_claim_unknown_events(local_handle);       
  win_claim_idle_events(local_handle);
  event_setmask(0);

  /* Add menu to logon window */
  local_menu=menu_new("ARCbbs",">Open spoolfile,Close,>Save as text,>Save as sprite,>Save message");
  event_attachmenu(local_handle,local_menu,do_local_menu,0);

  switch(tolower(slavemode))
    {
    case 'l':
      sprintf(buffer2,"ARCbbs %d (Local)",portnumber);
      break;
    case 'n':
      sprintf(buffer2,"ARCbbs %d (Online)",portnumber);
      break;
    case 'm':
      sprintf(buffer2,"ARCbbs %d (Online/Mailport)",portnumber);
      break;
    }
  settitle(local_handle,buffer2);   

  if (slavemode=='l') { snoop=1; scr_init(); open_swindow(local_handle,0,0); }
  }

void open_swindow(wimp_w wind,int x,int y)
  {
  wimp_wstate state;
  wimp_caretstr caret;

  wimp_get_wind_state(wind,&state);

  /* Open window to full size, ontop of everything else */
  state.o.x=x;
  state.o.y=y;
  state.o.behind=-1;

  errorlog(wimp_open_wind(&state.o),"OpenWindI");

  /* Get a caret */
  caret.w=local_handle;
  caret.i=-1;
  caret.x=caret.y=0;
  caret.height=1<<24;
  caret.index=0;
  wimp_set_caret_pos(&caret);
  }

void window_poll()
  {
  /* Poll wimp */
  event_process();

  /* Reselect our port */
  port_select(portnumber);

  /* If wimp is quit, jump to the program exit routine */
  if (quit) longjmp(jmp_wimpexit,quit);    

  /* If carrier detect enabled, check carrier and longjmp if it's gone */
  if (enablecarrier) if (!port_dcd()) longjmp(jmp_carrier,1);

  lastpoll=clock();
  }

void slave_msgproc(wimp_eventstr *event,void *handle)
  {
  doors_request doors_req;

  /* Reselect our port */
  port_select(portnumber);

  switch(event->e)
    {
    case 0:             /* Null event */
      {
      if (snoop)
        {
        int dummy_time,pending_alarm=alarm_next(&dummy_time);

        /* Deal with alarms */
        if (pending_alarm!=0 && dummy_time<=alarm_timenow()) alarm_callnext();

        if (infile!=NULL && port_rxbuffer()!=255)
          {                 
          while(port_rxbuffer()<255 && infile!=NULL)
            {
            int k=fgetc(infile);
            if (k!=EOF) key_insert(k); else _fclose(&infile);
            }
          }
        scr_housekeep();
        }
      break;
      }
    case wimp_EOPEN:             /* Open_Window_Request */
      {
      errorlog(wimp_open_wind(&event->data.o),"OpenWind");
      break;
      }
    case wimp_ECLOSE:             /* Close_Window_Request */
      {
      errorlog(wimp_close_wind(event->data.o.w),"CloseWind");
      if (event->data.o.w==local_handle)
        {
        snoop=0;
        }
      if (slavemode=='l') quit=2;
      break;
      }
    case wimp_EKEY:
      {       
      if (KEYIN!=0x1cc)
        {                                     
        if (xoff!=0)
          {
          xoff=0;
          }
        else
          {
          if (KEYIN==19) xoff=1;
          else
            {
            if (KEYIN==0x18a) KEYIN=9; /* Fix for TAB */  
            if (KEYIN==163) KEYIN=156; /* Fix for £ */
            switch(KEYIN)
              {
              case 0x18c: key_insert(27); key_insert('['); key_insert('D'); break;
              case 0x18d: key_insert(27); key_insert('['); key_insert('C'); break;
              case 0x18f: key_insert(27); key_insert('['); key_insert('A'); break;
              case 0x18e: key_insert(27); key_insert('['); key_insert('B'); break;
              default:    key_insert(KEYIN); break;
              }
            }
          }
        }
      else
        {
        /* Not ours, wimp can handle it */
        wimp_processkey(KEYIN);
        }
      break;
      }
    case wimp_EREDRAW:
      {
      wimp_redrawstr r;
      int more;

      if (snoop!=0)
        {
        r.w=event->data.o.w;
        wimp_redraw_wind(&r,&more);
        if (event->data.o.w==local_handle) scr_redraw(&r,&more);
        }
      break;
      }
    case wimp_EBUT: /* Mouse click */
      {
      if (snoop!=0) scr_setcaret(-1);
      break;
      }
    case wimp_ELOSECARET:
      {                          
      if (event->data.c.w==local_handle && snoop!=0) scr_setcaret(0);
      break;
      }
    case wimp_EGAINCARET:
      {                         
      if (event->data.c.w==local_handle && snoop!=0) scr_setcaret(1);
      break;
      }
    case 17:
    case 18:
      {
      switch(event->data.msg.hdr.action)
        {
        /* Pickup on quit message */
        case 0:   
        case BBS_QUIT:
          {
          if (servertask==0) exit(0);
          quit=(event->data.msg.hdr.action==0)?1:2;
          break;
          }
        case BBS_SERVERIS:
          {
          servertask=event->data.msg.hdr.task;
          break;
          }
        /* Drop user immediately */
        case BBS_DROPUSER:
          {
          /* Drop user longjmp */
          if (bbs_currenttask>0)
            {
            longjmp(jmp_carrier,1);
            }
          else
            {
            if (bbs_currenttask==0)
              {
              /* Put modem online in answer mode */
              port_txstring("ATA",1);
              port_txw(13);
              }
            }
          break;
          }
        case BBS_SNOOP:
          {
          snoop=1;
          if (scr_available==0) scr_init();
          open_swindow(local_handle,0,0);
          break;
          }
        case BBS_MUDATA:
          {
          if (muchatting)
            {
            strcpy(mumsg,event->data.msg.data.chars+12);
            }
          break;
          }
        case BBS_MESSAGE:
          {
          if (msginterrupt==0)
            {
            char *pointer=event->data.msg.data.chars+12;

            if (*pointer==1 && (user.f_user&USER_NOPAGE)!=0) break;
            else pointer++;

            mprintf(16+1,"%s",pointer);
            }
          break;
          }
        case BBS_TIME:
          {
          if (thislogon) thislogon+=event->data.msg.data.words[0];
          break;
          }
        case BBS_RATIO:
          {
          get_user(&user);
          user.uploads++;
          put_user(&user);
          break;
          }
        case BBS_FORCECHAT:
          {
          mprintf(1,"\n\n+++ Sysop breaking into chat\n");
          sysop_chat();
          break;
          }
        case BBS_DIVERT:
          {
          strcpy(currentmenu,event->data.msg.data.chars+12);
          bbs_currenttask=2; /* Interpret menu */
          ackdivert=1;
          break;
          }
        case BBS_READUSER:
          {
          get_user(&user);
          break;
          }
        case wimp_MDATALOAD:
          {
          char infilename[212];
          dbox d; int a;

          strcpy(infilename,event->data.msg.data.dataload.name);

          /* Send DataLoadAck */
          event->data.msg.hdr.your_ref=event->data.msg.hdr.my_ref;
          wimp_sendmessage(4,&event->data.msg,event->data.msg.hdr.task);
              
          d=dbox_new("dowot");
          dbox_showstatic(d);
          a=dbox_fillin(d);
          dbox_dispose(&d);
          if (a==0)
            {
            infile=fopen(infilename,"r");
            break;
            }
          if (a==1)
            {
            message_upload(infilename,1);
            }
          break;
          }     
        case wimp_PALETTECHANGE: /* Palette/mode change */
        case wimp_MMODECHANGE:
          {
          if (scr_available)
            {
            scr_colourmap();
            scr_totalredraw();
            }
          break;
          }
        default:
          {
          if (event->data.msg.hdr.action>BBS_CHUNK && event->data.msg.hdr.action<(BBS_CHUNK+64))
            {  
            got_code=event->data.msg.data.words[0];
            got_code2=event->data.msg.data.words[1];
            got_code3=event->data.msg.data.words[2];
            got_data=1; got_type=event->data.msg.hdr.action;
            if (got_code==EXCUSE || got_type==BBS_CHATDATA)
              {
              strcpy(got_string,event->data.msg.data.chars+12);
              }
            }
          break;
          }
        }
      break;
      }
    }                               

  /* Check for door forcing */
  if (doors_inuse==0)
    {
    int doorstat=doors_readstatus(portnumber);
    if (doorstat>0 && doorstat<255 && msginterrupt==0) doors_forceconnect();
    }
  
  /* Check for door requests */
  if (doors_getrequest(portnumber,&doors_req)==0) doors_dorequest(&doors_req);

  return;
  }
 
void errorlog(os_error *err,char *part)
  {
  os_error *err2;
  if (err==NULL) return;
  err2=wimp_reporterror(err,3,part);
  err2=wimp_closedown();
  exit(0);
  }
   
void bbs_dostuff(void)
  {
  /* Setup longjmp so that carrier drop can be gotten at */
  if (setjmp(jmp_carrier)!=0)
    {
    int minsonline=((clock()-logon_time)/6000);
    
    enablecarrier=0;

    if (chatting)
      {
      send_message(servertask,BBS_CHATREQUEST,portnumber,NO_CHATTING,0,0);
      chatting=0;
      }

    if (muchatting)
      {
      send_message(servertask,BBS_MUCONTROL,portnumber,MU_LEAVE,0,0);
      muchatting=0;
      }

    if (gateway)
      {
      port_select(gatewayport);
      port_rts(0); port_dtr(0);
      gateway=0;
      }

    /* Dumpout server stuff to disk */
    dumpfiles();                   

    /* If script was running, kill it */
    if (script_cankill) script_bomb();

    /* Clear door connects */
    doors_reset();

    /* Setup last caller info */
    lasttime=((clock()-logon_time)/100); lastbaud=baudrate;
    strcpy(lastcaller,user.username);

    /* Put entry into log */
    if (loggedon)
      {
      /* Set user 'new' read time & other flags */
      get_user(&user);
      user.timetoday+=minsonline;   
      user.conference=conference;
      user.filebase=filebase;
      put_user(&user);

      sprintf(message_buffer,"%-30s %c:%5d:%02d %04d %24s %d:%02d",
              user.username,nicelogoff?'L':'D',baudrate,portnumber,
              user.usernumber,cctime(&user.t_lastlogon),minsonline/60,
              minsonline%60);
      tolog(message_buffer); loggedon=0;
      }

    wimp_waitfor(200);

    /* Send new display info to server */
    send_message(servertask,BBS_ICONINFO,portnumber,0,0,0);
    if (online)
      {
      send_message(servertask,BBS_COUNTONLINE,-1,0,0,0); online--;
      send_message(servertask,BBS_COUNTOTHER,1,0,0,0);
      }

    tidy_up(); 
    return;
    }

  /*** This bit repeated until we get a good logon */
  do
    {
    /* Initialise port */
    window_poll(); 

    /* Set no timeout! */
    thislogon=loggedon=0; notimeout=1; nicelogoff=0;
    online_users[portnumber].username[0]=0;
    queue_length=0; scratchpad=0; conference=filebase=-1; beenchatting=0;
    online_users[portnumber].baudrate=-1;                                
       
    tidy_up();

    switch(slavemode)
      {
      case 'n': /* Normal BBS line */
        {
        do
          {
          port_xonxoff(0); port_rxclear();
          port_dtr(0); port_rts(0);
          wimp_waitfor(100);
          port_dtr(1); port_rts(1);

          modem_init();
          }
        while(answer_routine()==0);

        break;
        }
      case 'm': /* Mailport */
        {
        FILE *modemfile;

        /* Open SPAWN file to get other info */
        sprintf(tempbuffer,"<ARCbbs$Mailer>.BBS_%d",portnumber);
        if ((modemfile=fopen(tempbuffer,"r"))==NULL) exit(0);

        /* Get line */
        mygets(tempbuffer2,modemfile);
        sscanf(tempbuffer2,"%*s %d %*d %d %s",&baudrate,&mail_event,tempbuffer+128);
        if (strstr(tempbuffer+128,"Arq")!=NULL ||
            strstr(tempbuffer+128,"Rel")!=NULL) arq=1;

        /* Close file */
        fclose(modemfile);
        
        break;
        }
      case 'l': /* Local */
        {
        baudrate=19200;
        break;
        }
      }

    /* Reset door module */
    doors_reset();
    }
  while(logon_routine()==0);
  bbs_currenttask=2;

  if (setjmp(jmp_tomenu)!=0)
    {
    ackdivert=0;

    switch(bbs_currenttask)
      {
      case 1:
        if (logon_routine()==0) longjmp(jmp_carrier,1);
        bbs_currenttask=2;
        break;
      }
    }

  /*** Parse menus */
  do
    {
    /*** Do a menu ***********************************************************/
    int optioncount=0,keypress=0,loop,level; char lastmenu[20],*p;
    
    if (bbs_currenttask!=2) break;

    /* Poll so we can break infinite loops */
    timepoll();

    /* Keep a copy of current menu file open */
    strcpy(lastmenu,currentmenu);

    /*** Display menu text whilst we parse away merrily */
    if (user.f_user & USER_NOVICE)
      {
      /* Show the menu text */
      sprintf(tempbuffer,"<ARCbbs$menutext>.%s.%s",menupath,currentmenu);
      bbs_sendfile(tempbuffer,menu_options,SENDFILE_NONSTOP);
      }

    /* Open menu data file */
    sprintf(tempbuffer,"<ARCbbs$menudata>.%s.%s",menupath,currentmenu);
    if ((bbs_file[1]=fopen(tempbuffer,"rb"))==NULL)
      {
      port_txstring("Can't find menu\n",1);
      bbs_currenttask=3;
      break;
      }

    /* Parse file */
    do
      {
      /* Get a line */
      mygets(tempbuffer,bbs_file[1]);

      /* Comment line? */
      if (tempbuffer[0]!='|' && tempbuffer[0] && tempbuffer[0]!=255)
        {        
        /* Is line an immediate command or menu option? */
        if(tempbuffer[1]!=',')
          {
          /* Immediate command */
          do_command(tempbuffer,1);
          }
        else
          {
          /* Menu option (key,reply,command,elsecommand,level,flags) */
          menu_options[optioncount]=tempbuffer[0];

          p=tempbuffer+2;
          get_item(&p,menu_replies[optioncount]);
          get_item(&p,menu_commands[optioncount]);
          get_item(&p,menu_elsecommands[optioncount]);
          get_item(&p,tempbuffer2);

          menu_available[optioncount]=1;
          if (*tempbuffer2) sscanf(tempbuffer2+1,"%x",&level);
          switch(tempbuffer2[0])
            {
            case '<': if (user.userlevel>=level) menu_available[optioncount]=0; break;
            case '=': if (user.userlevel!=level) menu_available[optioncount]=0; break;
            case '>': if (user.userlevel< level) menu_available[optioncount]=0; break;
            }

          get_item(&p,tempbuffer2);
          switch(tolower(tempbuffer2[0]))
            {
            case 'f': if (flag_process(tempbuffer2+1)==0) menu_available[optioncount]=0; break;
            case 'u': if (user.usernumber!=atoi(tempbuffer2+1)) menu_available[optioncount]=0; break;
            }
          optioncount++;
          }
        }
      }
    while(!feof(bbs_file[1]));
    _fclose(&bbs_file[1]);

    /* Terminate the options string so we can use in sendfile2 */
    menu_options[optioncount]=0;

    if (strcmp(currentmenu,lastmenu) || optioncount==0) continue;

    /* Send the prompt */
    port_txstring(menu_prompt,1); 

    /* Get reply */
    getmenureply:
    do
      {
      keypress=toupper(port_get());
  
      /* '?' always redisplays menu (unless redefined) */
      if (keypress=='?' && strchr(menu_options,'?')==NULL) 
        {
        port_txstring("?\n\n",0);

        /* Show the menu text */
        sprintf(tempbuffer,"<ARCbbs$menutext>.%s.%s",menupath,currentmenu);
        bbs_sendfile(tempbuffer,menu_options,SENDFILE_NONSTOP);

        /* Send the prompt */
        port_txstring(menu_prompt,1); 
        }
      }
    while((loop=(int)strchr(menu_options,keypress))==NULL);
    loop-=(int)menu_options;

    /* Clear buffer to halt menu */
    port_txclear();

    /* Check user stats */
    if (menu_available[loop]==0)
      {
      if (*noaccess)
        {
        if (menu_elsecommands[loop][0]) do_command(menu_elsecommands[loop],1);
        else
          {
          /* Can't access that option */
          port_txstring(noaccess,1); port_crlf(); port_crlf();
          }
        }
      else goto getmenureply;
      }
    else
      {
      /* Print menu reply */
      port_txstring(menu_replies[loop],0); port_crlf();

      /* Do the command */
      do_command(menu_commands[loop],1);
      }
    }
  while(bbs_currenttask==2);

  if (bbs_currenttask==3)
    {
    /*** 'Nice' logoff system */
    nicelogoff=1;
    bbs_sendfile("<ARCbbs$text>.Logoff",0,0);

    /* Set user 'new' read time */
    get_user(&user);
    if (readnew) 
      {
      user.m_message=read_msgnumber()+1;
      if (user.m_message<0) user.m_message=0;
      }

    if (readnewfiles)
      {
      user.m_file=read_msgnumber()+1;
      if (user.m_file<0) user.m_file=0;
      }

    put_user(&user);
    }

  longjmp(jmp_carrier,1);
  }

void wimp_waitfor(int t)
  {
  clock_t start=clock();
  while((clock()-start)<=t) window_poll();
  }


void do_set(int thing,char *buffer,int save,int sysop,int usern)
  {
  if (buffer==NULL) buffer="";
  if (save) get_user(&user);

  switch(thing)
    {
    case 0:       /* SET PASSWORD */
      {
      int tmp1,tmp2;

      do
        {
        bbs_sendfile("<ARCbbs$text>.Password",0,0);

        port_txstring("Enter old password\n: ",1);
        port_readline(tempbuffer2,255,HIDE|NOBOX); port_crlf();
        if (user.passcrc!=calcrc(tempbuffer2,strlen(tempbuffer2)))
          {
          bbs_sendfile("<ARCbbs$text>.Badoldpw",0,0);
          goto endpw;
          }

        port_txstring("Enter new password\n: ",1);
        port_readline(tempbuffer2,255,HIDE|NOBOX); port_crlf();
        tmp1=calcrc(tempbuffer2,strlen(tempbuffer2));
        if (!*tempbuffer2) goto endpw;

        port_txstring("Reenter for confirmation\n: ",1);
        port_readline(tempbuffer2,255,HIDE|NOBOX); port_crlf();
        tmp2=calcrc(tempbuffer2,strlen(tempbuffer2));
        if (!*tempbuffer2) goto endpw;

        if (tmp1!=tmp2)
          {
          port_txstring("Passwords do not match. Try again.\n",1);
          }
        }
      while(tmp1!=tmp2);
      user.passcrc=tmp1;

      endpw:
      break;
      }
    case 1:       /* SET TERMINAL */
      {
      port_txstring("Your terminal emulation:\n 1. TTY\n 2. VT52\n 3. VT100\n 4. ANSI\nSelect: ",1);
      do
        {
        user.termtype=port_readchar(NOECHO)-49;
        }
      while(user.termtype<0 || user.termtype>3);
      port_txw(user.termtype+49); port_crlf();

      break;
      }
    case 2:       /* SET RATIO */
      {
      if (sysop) user.ratio=atoi(buffer);
      break;
      }
    case 3:       /* SET TIME */
      {
      if (sysop) user.timeallowed=atoi(buffer);
      break;
      }
    case 4:       /* UPLOADS */
      {
      if (sysop)
        {
        switch(buffer[0])
          {
          case '+':
            {
            user.uploads+=atoi(buffer+1); break;
            }
          case '-':
            {
            user.uploads-=atoi(buffer+1);
            if (user.uploads<0) user.uploads=0;
            break;
            }
          default:
            {
            user.uploads=atoi(buffer); break;
            }
          }
        }
      break;
      }
    case 5:       /* DOWNLOADS */
      {
      if (sysop)
        {
        switch(buffer[0])
          {
          case '+':
            {
            user.downloads+=atoi(buffer+1); break;
            }
          case '-':
            {
            user.downloads-=atoi(buffer+1);
            if (user.downloads<0) user.downloads=0;
            break;
            }
          default:
            {
            user.downloads=atoi(buffer); break;
            }
          }
        }
      break;
      }
    case 6:       /* USERLEVEL */
      {
      if (sysop) 
        {
        int a;
        
        if (sscanf(buffer,"%x",&a)==1) user.userlevel=a;
        }

      break;
      }  
    case 7:       /* MORE */
      {              
      user.f_user&=~USER_NOMORE;

      if (buffer[0])
        {
        if (tolower(buffer[1])=='f') user.f_user|=USER_NOMORE;
        }
      else
        {
        if (port_yesno("Do you want 'More?' prompts every screenful? ")==0)
          user.f_user|=USER_NOMORE;
        }
      break;
      }    
    case 8:       /* PAGELENGTH */
      {     
      if (buffer[0]) user.pagelen=atoi(buffer);
      else
        {
        char inbuf[4];
        int a;
              
        do
          {
          port_txstring("Page length [default=24] ",1);
          port_readline(inbuf,3,NUMONLY);
          if (inbuf[0]==0) 
            {
            port_txstring("{bfgbg wb}24{std}\n",1);
            strcpy(inbuf,"24");
            }
          else port_crlf();
          a=atoi(inbuf)-1;
          if (a<4 || a>255) port_txstring("Illegal page length. Please try again.\n",1);
          }
        while(a<4 || a>255);

        user.pagelen=a;
        }
      break;
      }
    case 9:       /* CLEARSCREEN */
      {              
      user.f_user&=~USER_NOCLEAR;

      if (buffer[0])
        {
        if (tolower(buffer[1])=='f') user.f_user|=USER_NOCLEAR;
        }
      else
        {
        if (port_yesno("Do you want screen clearing codes to be sent? ")==0)
          user.f_user|=USER_NOCLEAR;
        }
      break;
      }
    case 10:       /* SET PASSWORD */
      {
      int tmp1,tmp2;

      do
        {
        port_txstring("Enter new password\n: ",1);
        port_readline(tempbuffer2,255,HIDE|NOBOX); port_crlf();
        tmp1=calcrc(tempbuffer2,strlen(tempbuffer2));
        if (!*tempbuffer2) goto endpw2;

        port_txstring("Reenter for confirmation\n: ",1);
        port_readline(tempbuffer2,255,HIDE|NOBOX); port_crlf();
        tmp2=calcrc(tempbuffer2,strlen(tempbuffer2));
        if (!*tempbuffer2) goto endpw2;

        if (tmp1!=tmp2)
          {
          port_txstring("Passwords do not match. Try again.\n",1);
          }
        }
      while(tmp1!=tmp2);
      user.passcrc=tmp1;

      endpw2:
      break;
      }
    }
  if (save) put_user(&user);
  }

int do_filesearch(int arean,char *searchstr,int *searched,int *matches)
  {
  mail_block fileheader; mail_area area;
  int current,lastmsg=clock(); char *ptr;
       
  read_area(arean,&area);        

  /* Trap for unused areas to stop annoying pause! */
  if (area.name[0]==0) return(0);
        
  /* Poll wimp to stop CPU-hogging */
  timepoll();

  if (area.name[0]!=0 && can_read(&area) && (area.areaflags&FLAG_FILEAREA)!=0)
    {                     
    current=area.last_message;

    while(current!=-1)
      {
      timepoll();
      if ((clock()-lastmsg)>100)
        {
        mprintf(1,"{bfgbg cn}Searched {fg y}%5d {fg c}files\r",*searched);
        lastmsg=clock();
        }

      fileheader.message_number=current;
      get_messageh(&fileheader); (*searched)++;
      current=fileheader.message_backward;

      if (fileheader.status==1)
        {
        ptr=fileheader.to;      while(*ptr) *ptr++=tolower(*ptr);
        ptr=fileheader.subject; while(*ptr) *ptr++=tolower(*ptr);

        if (strstr(fileheader.to,searchstr)!=NULL ||
            strstr(fileheader.subject,searchstr)!=NULL)
          {
          if ((fileheader.flags&FILE_SYSOP)==0 ||
              (user.f_user&USER_SYSOP)!=0)
            {
            int trash=1;
            get_message(&fileheader,message_buffer);
            mprintf(1,"\n"); (*matches)++;
            if (longoptions(&fileheader,&area,&trash)) return(1);

            mprintf(1,"{bfgbg cn}Searched {fg y}%5d {fg c}files\r",*searched);
            lastmsg=clock();
            }
          }
        }
      } 
    }                 

  /* Poll some more */
  timepoll();
  return(0);
  }

void do_command(char *command,int sysop)
  {
  int c1,third,posn=0;
  char br_used,was_quoted,commandbuffer[40],buffer[128],buffer2[128],
       buffer3[128],*elcommand=0,*el;

  *buffer=*buffer2=*buffer3=0;

  /* Look for ELSE, just in case */
  if ((el=strstr(command," ELSE "))!=NULL)
    {
    *el=0;
    elcommand=el+6;
    }

  /* Get command out */
  if (!parser(1,commandbuffer,39,command,whitesp,breakch,quote,0,&br_used,&posn,&was_quoted))
    {
    /* Get second parameter */
    if (!parser(0,buffer,127,command,whitesp,breakch,quote,0,&br_used,&posn,&was_quoted))
      {
      third=posn;

      /* Get 3rd parameter */
      if (!parser(0,buffer2,127,command,whitesp,breakch,quote,0,&br_used,&posn,&was_quoted))
        {
        strcpy(buffer3,command+posn);
        }
      }
    }

  /* Find what command it is */
  for(c1=0;(stricmp(commands[c1],commandbuffer) && commands[c1][0]!=' ');c1++);

  /* Process command */
  switch(c1)
    {
    case 0:  /* GOTO <menuname> */
      {
      if (sysop) strcpy(currentmenu,buffer);
      break;
      }
    case 1:  /* LOGOFF */
      {
      bbs_currenttask=3; break;
      }
    case 2:  /* DROP */
      {
      bbs_currenttask=4; break;
      }
    case 3:  /* RELOGON */
      {
      bbs_currenttask=1; longjmp(jmp_tomenu,1); break;
      }
    case 4:  /* TYPE <filename> [NOPROMPT] */
      {
      if (bbs_sendfile(buffer,"\003 \015",(tolower(buffer2[0])=='n')?SENDFILE_NONSTOP:0)) port_rx();
      break;
      }
    case 5:  /* TYPE_F <filename> [NOPROMPT] */
      {
      if (bbs_sendfile(buffer,"\003 \015",(tolower(buffer2[0])=='n')?SENDFILE_NONSTOP:0)) port_rx();
      break;
      }
    case 6:  /* ONLINE */
      {
      send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                  0,"Snooping");
      strcpy(online_users[portnumber].doing,"Snooping");

      showonline();
      break;
      }
    case 7:  /* READ <messagearea> */
      {
      int areanumber=atoi(buffer);
      mail_area info;
             
      if (tolower(buffer[0])=='c')
        {
        if ((areanumber=conference)==-1)
          {
          port_txstring("{bfg r}No conference selected{std}\n",1);
          break;
          }
        }

      /* Get area info */
      read_area(areanumber,&info);
      if ((info.areaflags&FLAG_FILEAREA)!=0)
        {
        mprintf(1,"{bfg r}Area %d (%s) is not a message area{std}\n",areanumber,info.name);
        break;
        }
      if (can_read(&info)==0)
        {
        mprintf(1,"{bfg r}You are not authorized to read this area.{std}\n");
        break;
        }
      message_read(areanumber,&info,0);
      break;
      }
    case 8:  /* WRITE <messagearea> [<userid>] */
      {
      int areanumber=atoi(buffer),to=-1;
      mail_area info;
                
      if (tolower(buffer[0])=='f')
        {
        _address_arch2 to;

        /* Fidonet write, check for pre-addressed mail */
        if (buffer2[0]!=0)
          {
          if (sscanf(buffer2,"%hd:%hd/%hd.%hd",&to.zone,&to.net,&to.node,&to.point)==4)
            {
            if (buffer3[0]!=0)
              {
              message_fidowrite(&to,buffer3,"",-1);
              break;
              }
            else
              {
              message_fidowrite(&to,NULL,"",-1);
              break;
              }
            }
          }

        message_fidowrite(NULL,NULL,"",-1);
        break;
        }

      if (tolower(buffer[0])=='c')
        {
        if ((areanumber=conference)==-1)
          {
          port_txstring("{bfg r}No conference selected{std}\n",1);
          break;
          }
        }

      /* Preaddressed? */
      if (*buffer2) to=atoi(buffer2);

      /* Get area info */
      if (areanumber>=0)
        {
        read_area(areanumber,&info);
        if ((info.areaflags&FLAG_FILEAREA)!=0)
          {
          mprintf(1,"{bfg r}Area %d (%s) is not a message area{std}\n",areanumber,info.name);
          break;
          }

        if (can_write(&info)==0)
          {
          mprintf(1,"{bfg r}You are not authorized to write to this area.{std}\n");
          break;
          }
        message_write(areanumber,&info,to);
        }
      break;
      }
    case 9:  /* UPLOAD <filearea> */
      {
      char filename[21],realname[81];
      int  error,option,areanumber=atoi(buffer);
      mail_area info; clock_t startupload;
      char tbuffer[40];

      if (tolower(buffer[0])=='c')
        {
        if ((areanumber=filebase)==-1)
          {
          port_txstring("{bfg r}No filebase selected{std}\n",1);
          break;
          }
        }
             
      if (areanumber)
        {
        read_area(areanumber,&info);
        if (can_write(&info)==0)
          {
          mprintf(1,"{bfg r}You are not authorized to write to this area.{std}\n");
          break;
          }
        }

      /* Batch filename */
      sprintf(tbuffer,"<ARCbbs$temp>.Batch_%d",portnumber);
           
      showm:
      bbs_sendfile("<ARCbbs$prompts>.Upload",0,0);
          
      getc:                              
      startupload=clock();
      switch(option=toupper(port_get()))
        {
        case 'Z':
          {              
          port_txstring("Zmodem\n",1);
          error=zmodem_error(zmodem(0),0,1);
          goto batchfill;
          }
        case 'S':
          {              
          port_txstring("SEAlink\n",1);
          error=zmodem_error(sealink(0),0,1);
          goto batchfill;
          break;
          }
        case 'K':
          {              
          port_txstring("Kermit\n",1);
          error=zmodem_error(kermit(0),0,1);
          goto batchfill;
          break;
          }
        case 'Y':
          {              
          port_txstring("Ymodem Batch\n",1);
          error=zmodem_error(ymodem(0),0,1);
          goto batchfill;
          break;
          }
        case 'X':
        case '1':
          {
          port_txstring(option=='X'?"Xmodem\n":"Xmodem-1k\n",1);
          port_txstring("\nFilename: ",1);
          port_readline(filename,20,0);
          port_crlf();

          sprintf(realname,"<ARCbbs$Upload>.%02d/0000",portnumber);
          error=xmodem_error(xmodem(realname,(option=='X')?0:1,0),0,1);

          if (error==DONE)
            {
            int filelen;

            /* Find length of file */
            if ((bbs_file[4]=fopen(realname,"rb"))!=NULL)
              {
              fseek(bbs_file[4],0,SEEK_END);
              filelen=(int)ftell(bbs_file[4]);
              _fclose(&bbs_file[4]);
              }
            else
              {
              filelen=0;
              }

            thislogon+=((clock()-startupload)/6000)*2; /* Twice time */
            file_upload(areanumber,filelen,realname,filename);
            }
          else
            {
            /* Delete the temporary file */
            remove(realname);
            }
          break;
          }
        case 'C':
          {
          port_txstring("Cancelled\n",1); goto endupload;
          }         
        case '?':
          {
          goto showm;
          }
        default:
          {
          goto getc;
          }
        }
          
      endupload:
      break;

      batchfill:
      if (error==DONE)
        {
        int filelen,filenumber=0;

        if ((bbs_file[4]=fopen(tbuffer,"rb"))!=NULL)
          {
          do
            {
            if (!feof(bbs_file[4]))
              {
              sprintf(tbuffer,"<ARCbbs$Upload>.%02d/%04d",portnumber,
                                                             filenumber++);
              /* Find length of file */
              if ((bbs_file[5]=fopen(tbuffer,"rb"))!=NULL)
                {
                fseek(bbs_file[5],0,SEEK_END);
                filelen=(int)ftell(bbs_file[5]);
                _fclose(&bbs_file[5]);
                }
              else
                {
                filelen=0;
                }
              
              /* Get filename from batch */
              mygets(realname,bbs_file[4]);
              if (realname[0])
                {
                realname[20]=0; /* Trim to 20 characters */

                file_upload(areanumber,filelen,tbuffer,realname);
                }
              }
            }
          while(!feof(bbs_file[4]));
          _fclose(&bbs_file[4]);
          }
        thislogon+=(clock()-startupload)/3000; /* Twice time */
        }
      break;
      }
    case 10: /* DOWNLOAD <filearea> */
      {
      char filename[21];
      int filearea=atoi(buffer),current,fn;
      mail_area area; mail_block fileheader;
                                        
      if (tolower(buffer[0])=='c')
        {
        if ((filearea=filebase)==-1)
          {
          port_txstring("{bfg r}No filebase selected{std}\n",1);
          break;
          }
        }

      read_area(filearea,&area);

      if ((area.areaflags&FLAG_FILEAREA)==0)
        {
        mprintf(1,"{bfg r}Area %d (%s) is not a file area{std}\n",filearea,area.name);
        break;
        }

      if (can_read(&area)==0)
        {
        mprintf(1,"{bfg r}You are not authorized to read this area.{std}\n");
        break;
        }

      do
        {
        port_txstring("\nFilename/#filenumber: ",1);
        port_readline(filename,20,0);
        port_crlf();
                           
        if(filename[0]==0) break;

        if(filename[0]=='#')
          {
          fn=atoi(filename+1);

          /* Filenumber */
          current=area.last_message;
        
          do
            {
            fileheader.message_number=current;
            get_messageh(&fileheader);
            current=fileheader.message_backward;
            }
          while(fileheader.file_location!=fn && current!=-1);
        
          if (fileheader.file_location!=fn)
            {
            port_txstring("File not found.\n",1);
            goto enddl;
            }
          }
        else
          {
          char *a;

          /* Filename */
          current=area.last_message;      

          a=filename; while(*a) *a++=toupper(*a);

          do
            {
            fileheader.message_number=current;
            get_messageh(&fileheader);
            a=fileheader.to; while(*a) *a++=toupper(*a);
            current=fileheader.message_backward;

            timepoll();
            }
          while(strcmp(fileheader.to,filename) && current!=-1);

          if (strcmp(fileheader.to,filename))
            {
            port_txstring("File not found.\n",1);
            goto enddl;
            }

          fn=fileheader.file_location;
          }
        }
      while(0);

      if (filename[0]) file_download(fn,fileheader.file_length,fileheader.to,1,1);

      enddl:
      break;
      }
    case 11: /* DOING <string> */
      {
      send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                      0,buffer);
      strcpy(online_users[portnumber].doing,buffer);
      break;
      }
    case 12: /* FILELIST <filebase> <SHORT|LONG> */
      {                      
      int filearea=atoi(buffer); mail_area info;

      if (tolower(buffer[0])=='c')
        {
        if ((filearea=filebase)==-1)
          {
          port_txstring("{bfg r}No filebase selected{std}\n",1);
          break;
          }
        }

      read_area(filearea,&info);
      if (can_read(&info)==0)
        {
        mprintf(1,"{bfg r}You are not authorized to read this area.{std}\n");
        break;
        }

      do_filelist(filearea,(strcmp(buffer2,"SHORT")==0),0);
      break;
      }
    case 13: /* PAGE */
      {
      send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,0,
                          "Requesting chat");
      /* Ask for chat */
      port_txstring("\nRequesting chat...",1);
      send_message(servertask,BBS_CHATREQUEST,portnumber,PAGING,0,0);

      /* Set flag */
      got_data=0;

      /* Reply will arrive soon */
      while(!got_data) window_poll();

      /* Set flag */
      got_data=0;

      switch(got_code)
        {
        case NO_CHATTING:
          {
          port_txstring("sorry, paging is disabled.\n",1);
          goto endchat;
          }
        case ALREADY_CHATTING:
          {
          port_txstring("sorry, Sysop is already chatting.\n",1);
          goto endchat;
          }
        case PAGING:
          {
          port_txstring("paging...",1);
          }
        }

      /* Reply will arrive soon */
      while(!got_data) window_poll();

      switch(got_code)
        {
        case NO_CHATTING:
          {
          port_txstring("no reply.\n",1);
          break;
          }
        case EXCUSE:
          {
          port_txstring("excuse reads:\n",1);
          port_txstring(got_string,1);
          port_crlf();
          break;
          }
        case ACCEPT:
          {
          port_txstring("chat accepted\n",1);
          port_txstring("\n+++ Chat started\n",1);
          sysop_chat();
          break;
          }
        }
      endchat:
      break;
      }
    case 14: /* CHAT <channel> */
      {
      int channel=atoi(buffer),tp=0,ch,xl;
      char typebuffer[80],firstname[31];
                             
      /* Find first name */
      sscanf(user.username,"%s ",&firstname);
      if (sscanf(user.username,"%s ",&firstname)==0)
        strcpy(firstname,user.username);

      /* Send message so we know who is in chat or not */

      send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                      0,"Multiuser chat");
      strcpy(online_users[portnumber].doing,"Multiuser chat");

      muchatting=1;
      port_txstring("\n{bfgbg wb} MultiUser Chat v1.02 {bg n} Started (use /h for help)\n",1);
      wimp_waitfor(25);

      /* Ask server to chain us into chat channel */
      send_message(servertask,BBS_MUCONTROL,portnumber,MU_JOIN,channel,0);

      do
        {     
        clock_t t=clock();
        tp=xl=0; 
          
        mumsg[0]=0; typebuffer[0]=0;
                                  
        do
          {
          while(port_rxbuffer()==0 && (clock()-t)<12000)
            {
            window_poll();
            if (mumsg[0])
              {
              char *mm2=mumsg;
              int how=16;
              
              if (mumsg[0]==1)
                {
                mm2++; how=0;
                }

              if (strlen(mm2)>tp)
                {
                mprintf(how+1,"\015%s\n",mm2);
                mprintf(16+1,typebuffer);
                }
              else
                {
                int a=tp-strlen(mm2),b;
                mprintf(how+1,"\015%s",mm2);
                for(b=0;b<a;b++) port_txw(' ');
                mprintf(16+1,"\n%s",typebuffer);
                }
              mumsg[0]=0;
              }
            }

          if (port_rxbuffer()==0) goto endmuchat;

          switch(ch=port_get())
            {
            case 10:
            case 13:
              {
              port_crlf();
              xl=1;
              break;
              }
            case 8:
            case 127:
              {
              if (tp)
                {
                typebuffer[--tp]=0;
                port_txw(8); port_txw(32); port_txw(8);
                }
              break;
              }
            default:
              {
              if (ch>31 && tp!=80)
                {
                port_txw(ch);
                typebuffer[tp++]=ch;
                typebuffer[tp]=0;
                }
              break;
              }
            }
          if (xl!=0 && typebuffer[0]==0) xl=0;
          }        
        while(xl==0);

        if (typebuffer[0]!='/')
          {
          if (typebuffer[0]=='-')
            {
            sprintf(tempbuffer,"%s %s",firstname,typebuffer+1);
            }
          else
            {
            sprintf(tempbuffer,"%s says: %s",firstname,typebuffer);
            }
          send_strmessage(servertask,BBS_MUDATA,portnumber,MU_PUBLIC,0,
                          tempbuffer);
          }
        else
          {
          switch(toupper(typebuffer[1]))
            {
            case 'X':
              {
              muchatting=0;
              break;
              }
            case 'H':
              {
              bbs_sendfile("<ARCbbs$prompts>.MUchat",0,0);
              break;
              }
            case '!':
              {
              int prt=atoi(typebuffer+2);

              if (prt>=0 && prt<=16)
                {
                sprintf(tempbuffer,"[%d] %s whispers %s",portnumber,
                        firstname,typebuffer+3);
                send_strmessage(servertask,BBS_MUDATA,portnumber,MU_PRIVATE,
                                prt,tempbuffer);
                }
              break;
              }
            case 'S':
              {
              showonline();
              break;
              }
            case 'P':
              {
              int prt=atoi(typebuffer+2);

              if (prt>=0 && prt<=16)
                {
                mprintf(1,"{bfgbg wb} MultiUser Chat {bg n} Paging port %d...\n",prt);
                sprintf(tempbuffer,"\001\007%s requests your presence in MultiUser chat\015",user.username);
                send_strmessage(servertask,BBS_MESSAGE,portnumber,
                                prt,0,tempbuffer);
                }
              break;
              }
            }
          }
        }
      while(muchatting);

      endmuchat:

      send_message(servertask,BBS_MUCONTROL,portnumber,MU_LEAVE,channel,0);

      /* Send message so we know who is in chat or not */
      send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                      0,"Limbo");
      strcpy(online_users[portnumber].doing,"Limbo");

      port_txstring("\n{bfgbg wb} MultiUser Chat {bg n} Finished\n",1);
      
      break;
      }
    case 15: /* CHECKMAIL [<userid>] */
      {
      if (sysop && buffer[0]!=0)
        {
        do_checkmail(atoi(buffer),0);
        }
      else
        {
        do_checkmail(user.usernumber,0);
        }
      break;
      }
    case 16: /* SETFLAGS <flags> */
    case 17: /* CLRFLAGS <flags> */
    case 18: /* TOGFLAGS <flags> */
      {
      int a,u=0,f=0,m=0;

      if (sysop)
        {            
        get_user(&user);
        for(a=0;a<strlen(buffer);a+=2)
          {
          int number=toupper(buffer[a+1]);
          if (number>='0' && number<='9') number-='0';
          else number-=('A'-10);

          /* Check flag type */
          switch(toupper(buffer[a]))
            {
            case 'U': /* User flag */
              {
              u|=1<<number; break;
              }
            case 'F': /* File flag */
              {
              f|=1<<number; break;
              }
            case 'M': /* Message flag */
              {
              m|=1<<number; break;
              }
            }
          }

        switch(c1)
          {
          case 16: /* SETFLAGS */
            {
            user.f_user|=u; user.f_file|=f; user.f_message|=m;
            break;
            }
          case 17: /* CLRFLAGS */
            {
            user.f_user&=~u; user.f_file&=~f; user.f_message&=~m;
            break;
            }
          case 18: /* TOGFLAGS */
            {
            user.f_user^=u; user.f_file^=f; user.f_message^=m;
            break;
            }
          }
        put_user(&user);
        }
      break;
      }
    case 19: /* GOTO_P <password> <menuname> */
      {
      char password[40];

      if (sysop)
        {
        port_txstring("Password :",1);
        port_readline(password,39,HIDE|TOUPPER|NOBOX);
   
        if (!strcmp(password,buffer))
          {
          port_txstring(": Accepted\n",1);
          strcpy(currentmenu,buffer2);
          }
        else
          {
          if (*noaccess)
            {
            mprintf(1,"\n%s\n",noaccess);
            }
          }
        }
      break;
      }
    case 20: /* PROMPT <string> */
      {
      strcpy(menu_prompt,buffer);
      break;
      }
    case 21: /* SHOWQUEUE */
      { 
      queue_show();
      break;
      }
    case 22: /* SET */
      {
      int thing=0;

      /* Find what command it is */
      for(;(strcmp(buffer,sets[thing]) && sets[thing][0]!=' ');thing++);                                   
      do_set(thing,buffer2,1,sysop,user.usernumber);
      break;
      }
    case 23: /* PATH <menupath> */
      {
      if (sysop) strcpy(menupath,buffer);
      break;
      }
    case 24: /* ECHO <string> */
      {
      if (*buffer)
        {
        port_txstring(buffer,1);
        }
      else
        {
        port_crlf();
        }
      break;
      }
    case 25: /* NOACCESS <string> */
      {
      if (sysop)
        {
        if (*buffer)
          {
          strcpy(noaccess,buffer);
          }
        else
          {
          *noaccess=0;
          }
        }
      break;
      }
    case 26: /* IF <condition> <command> ELSE <command> */
      {
      int level=0,gol=-10,can=1,usernr=-1;

      switch(tolower(buffer[0]))
        {
        case 'f': /* Flags */
          {
          can=flag_process(buffer+1);
          break;
          }
        case 'l': /* Userlevel */
          {
          sscanf(buffer+2,"%x",&level);
          switch(buffer[1])
            {
            case '<': gol=-1; break;
            case '>': gol=1;  break;
            case '=': gol=0;  break;
            }
          break;
          }
        case 'u': /* User */
          {
          sscanf(buffer+1,"%d",&usernr);
          break;
          }
        }

      if (((gol==  1) && (user.userlevel<level)) ||
          ((gol== -1) && (user.userlevel>level)) ||
          ((gol==  0) && (user.userlevel!=level)) ||
          ((usernr!=-1) && (user.usernumber!=usernr)) || can==0)
        {
        if (elcommand!=NULL)
          {
          do_command(elcommand,sysop);
          }
        else
          {
          if (*noaccess)
            {
            /* Can't access that option */
            mprintf(1,"%s\n",noaccess);
            }
          }
        }
      else
        {
        char buffera[256];
        
        sprintf(buffera,"%s %s",buffer2,buffer3);
        do_command(buffera,sysop);
        }

      break;
      }
    case 27: /* CLI */
      {
      do_cli((user.f_user&USER_SYSOP)!=0);
      break;
      }
    case 28: /* UTILS */
      {
      if (sysop) do_utils();
      break;
      }
    case 29: /* HELP */
    case 30: /* ? */
      {
      bbs_sendfile("<ARCbbs$text>.CLI_help",0,0);
      break;
      }
    case 31: /* CALLRATE <char> */
      {
      get_user(&user);
      user.callrate=buffer[0];
      put_user(&user);
      break;
      }
    case 32: /* CALLCOST */
      {
      int unit=494,units,cost,tpu=0,rate=1,onfor=(clock()-answer_time)/100;
      struct tm now;

      gtime(&now,time(NULL));

      switch(now.tm_wday)
        {
        case 6:
        case 0:
          {
          break;
          }
        default:
          {
          if ((now.tm_hour>=8  && now.tm_hour<9 ) ||
              (now.tm_hour>=13 && now.tm_hour<18)) rate=2;
          if (now.tm_hour>=9 && now.tm_hour<13) rate=3;
          break;
          }
        }

      switch(toupper(user.callrate))
        {
        case 'L':
          {
          switch(rate)
            {
            case 1: tpu=220; break;
            case 2: tpu= 80; break;
            case 3: tpu= 58; break;
            }
          break;
          }
        case 'A':
          {
          switch(rate)
            {
            case 1: tpu= 81; break;
            case 2: tpu= 36; break;
            case 3: tpu= 27; break;
            }
          break;
          }
        case '1':
          {
          switch(rate)
            {
            case 1: tpu= 50; break;
            case 2: tpu= 32; break;
            case 3: tpu= 24; break;
            }
          break;
          }
        case 'B':
          {
          switch(rate)
            {
            case 1: tpu= 38; break;
            case 2: tpu= 26; break;
            case 3: tpu= 19; break;
            }
          break;
          }
        }

      mprintf(1,"{bfg c}You've been online for %d:%02d, ",onfor/60,onfor%60);

      if (tpu!=0 && rate!=0)
        {
        units=(((clock()-answer_time)/100)/tpu)+1;
        cost=unit*units;
        mprintf(1,"cost of call: %c%d.%02d{std}\n",
                   (user.termtype==TER_ANSI)?0x9c:163,cost/10000,(cost/100)%10000);
        }
      else
        {
        mprintf(1,"{bfg r}Unknown call rate{std}\n");
        }
      break;
      }
    case 33: /* DOING_ALERT */
      {
      send_strmessage(servertask,BBS_USERTASK,portnumber,msginterrupt=0,
                      0,buffer);
      strcpy(online_users[portnumber].doing,buffer);
      }
    case 34: /* ALERT */
      {
      send_message(servertask,BBS_ALERT,portnumber,0,0,0);
      break;
      }
    case 35: /* TIME */
      {
      switch(buffer[0])
        {
        case '+':
          {
          thislogon+=atoi(buffer+1);
          break;
          }
        case '-':
          {
          thislogon-=atoi(buffer+1);
          if (thislogon<0) thislogon=0;
          break;
          }
        default:
          {
          thislogon=atoi(buffer);
          break;
          }
        }
      break;
      }
    case 36: /* GOTO_IP <password> <menuname> */
      {
      char password[40];

      if (sysop)
        {
        port_readline(password,39,NOECHO|TOUPPER|NOBOX);
   
        if (!strcmp(password,buffer))
          {
          strcpy(currentmenu,buffer2);
          }
        }
      break;
      }
    case 37: /* NEWSCAN <FILE|MESSAGE> */
      {
      int count,abort=0,max=areamax(); mail_area area;

      if (strcmp(buffer,"FILE"))
        {
        /* Message scan */
        for(count=1;count<max && abort==0;count++)
          {
          if (conf_member(user.conferences,count))
            {
            read_area(count,&area);
            if ((area.areaflags&FLAG_FILEAREA)==0 && area.name[0]!=0)
              {
              if (can_read(&area))
                {
                if (message_read(count,&area,'N')!=0) /* Read all new msgs */
                  {
                  /* User wants to abort */
                  if (port_yesno("\n{bfg r}Do you wish to abort whole newscan? {std}")) break;
                  }
                }  
              }
            }
          }
        }
      else
        {
        /* File scan */
        for(count=1;count<max && abort==0;count++)
          {  
          if (conf_member(user.conferences,count))
            {
            read_area(count,&area);
            if ((area.areaflags&FLAG_FILEAREA)!=0 && area.name[0]!=0)
              {
              if (can_read(&area))
                {
                if (do_filelist(count,0,'N')!=0) /* Read all new files */
                  {
                  /* User wants to abort */
                  if (port_yesno("\n{bfg r}Do you wish to abort whole newscan? {std}")) break;
                  }
                }
              }
            }
          }
        }
      port_crlf();
      break;
      }
    case 38: /* FILESEARCH_G */
      {
      int count,max=areamax(),searched=0,matches=0; char searchstr[21];
                            
      port_txstring("\n{bfg g}Enter (partial) filename/description to search for: ",1);
      port_readline(searchstr,20,TOLOWER); port_crlf();
      if (*searchstr==0) break;                       

      /* Global search */
      for(count=1;count<max;count++)
        {   
        if (conf_member(user.conferences,count))
          {              
          if (do_filesearch(count,searchstr,&searched,&matches))
            {
            /* User wants to abort */
            if (port_yesno("\n{bfg r}Do you wish to abort whole search? {std}")) break;
            } 
          }
        }

      mprintf(1,"{bfgbg cn}Searched {fg y}%5d {fg c}files, got {fg y}%d {fg c}matches\n",searched,matches);

      break;
      }
    case 39: /* FILESEARCH <area> */
      {
      char searchstr[21]; int filearea=atoi(buffer),searched=0,matches=0;

      if (tolower(buffer[0])=='c')
        {
        if ((filearea=filebase)==-1)
          {
          port_txstring("{bfg r}No filebase selected{std}\n",1);
          break;
          }
        }
                            
      port_txstring("\n{bfg g}Enter (partial) filename/description to search for: ",1);
      port_readline(searchstr,20,TOLOWER); port_crlf();
      if (*searchstr)
        {
        do_filesearch(filearea,searchstr,&searched,&matches);
        mprintf(1,"{bfgbg cn}Searched {fg y}%5d {fg c}files, got {fg y}%d {fg c}matches\n",searched,matches);
        }
      break;
      }
    case 40: /* S_MESSAGE */
      {
      int count,max=areamax(),wantit=(stricmp(getenv("ARCbbs$theworks"),"yes")==0 && (user.f_user&USER_SYSOP)!=0);
      mail_area area;
            
      remove(file_pathname(FILE_SCRATCHPAD));
      if ((bbs_file[6]=fopen(file_pathname(FILE_SCRATCHPAD),"w+"))==NULL)
        {
        mprintf(1,"{bfg r}Cannot open scratchpad file!{std}\n");
        break;
        }
      
      /* More buffering, maestro! */
      setvbuf(bbs_file[6],NULL,_IOFBF,16384);

      mprintf(1,"{bfg g}Filing messages in your scratchpad...\n");
                
      noof_filed=0;

      /* Global 'file' */
      for(count=(tolower(buffer[0])=='p')?0:1;count<max;count++)
        {        
        if (conf_member(user.conferences,count) || count==0)
          {
          /* Read the area */
          read_area(count,&area);
                           
          if (((area.areaflags&FLAG_FILEAREA)==0 && area.name[0]!=0 &&
              can_read(&area)) || count==0)
            {                                      
            /* Remember to use CRLF's with PC format */
            message_file(count,&area,bbs_file[6],wantit);
            }
          }
        }     
      scratchpad=(int)ftell(bbs_file[6]);

      /* Close the batch file */
      _fclose(&bbs_file[6]);

      mprintf(1,"\015Filed %4d messages (%d bytes)\n",noof_filed,scratchpad);
      port_rxclear();          

      break;
      }    
    case 41: /* S_ARC [c16|q|v] */ 
      {
      char tname[30],sname[30],command[128]; int newlen=0;
                                            
      if (scratchpad==0)
        {
        mprintf(1,"{bfg r}Your scratchpad is empty{std}\n");
        break;
        }     

      mprintf(1,"{bfg g}ARC'ing scratchpad...");
         
      sprintf(tname,"<ARCbbs$temp>.Temp_%d",portnumber);
      sprintf(sname,"<ARCbbs$temp>.Scratch_%d",portnumber);
              
      /* Make sure no old file exists */
      remove(tname);

      /* Makeup spark command and send it */
      sprintf(command,"%s -G -H -a",buffer);
      do_arc(tname,"/",command,sname);

      /* Erase scratchpad & rename ARC as normal scratchpad */
      while(remove(sname)) window_poll();
      rename(tname,sname);

      if ((bbs_file[4]=fopen(sname,"rb"))!=NULL)
        {
        fseek(bbs_file[4],0,SEEK_END);
        newlen=(int)ftell(bbs_file[4]);
        _fclose(&bbs_file[4]);
        }

      mprintf(1,"done ({bfg c}old length={fg w}%d{fg c}, new length={fg w}%d{fg g}){std}\n",scratchpad,newlen);
      scratchpad=newlen;
      port_rxclear();

      break;
      }
    case 42: /* S_DOWNLOAD */
      {
      if (scratchpad)
        {                                             
        /* Download the file to the user */
        file_download(FILE_SCRATCHPAD,scratchpad,"scratchpad",1,0);
        }
      else
        {
        mprintf(1,"{bfg r}Your scratchpad is empty{std}\n");
        }
      break;
      }
    case 43: /* ANYKEY */
      {
      port_get();
      break;
      }
    case 44: /* CONF_MEMBER <FILE|MESSAGE> */
      {  
      int message=(strcmp(buffer,"FILE")!=0),max=areamax(),a,abort=0,
          line=0,nonstop=(user.f_user&USER_NOMORE);
      mail_area area;    

      if (message)
        {
        mprintf(1,"{bfgbg wb}  # {fg y}#Msg {fg c}Conference name               {fg w}  # {fg y}#Msg {fg c}Conference name{eol std}\n");
        }
      else
        {
        mprintf(1,"{bfgbg wb}  # {fg y}#Fil {fg c}Filebase name                 {fg w}  # {fg y}#Fil {fg c}Filebase name{eol std}\n");
        }

      for(a=1;a<max && abort==0;a++)
        {
        if (conf_member(user.conferences,a))
          {                  
          /* Read the area thingy! */
          read_area(a,&area);        

          if ((message!=0 && (area.areaflags&FLAG_FILEAREA)==0) ||
              (message==0 && (area.areaflags&FLAG_FILEAREA)!=0))
            {
            if ((line%2)==0)
              {
              abort=mprintf(0,"{bfg w}%3d {fg y}%4d {fg c}%-30s",a,
                                            area.count,area.name);
              }
            else
              {
              abort=mprintf(0,"{bfg w}%3d {fg y}%4d {fg c}%s\n",a,
                                            area.count,area.name);
              }

            line++;
            if (line==(user.pagelen*2) && nonstop==0)
              {
              int ch;

              line=0;
              mprintf(1,"{bfg w}More: {fg c}[{fg y}Space{fg c}]{fg w}/{fg c}[{fg y}Return{fg c}]{fg w} continues, {fg c}[{fg y}N{fg c}]{fg w}onstop, {fg c}[{fg y}Ctrl-C{fg c}]{fg w} aborts{fg y}\015");
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
            }
          }
        }
      port_crlf();

      break;
      }
    case 45: /* CONF_LIST <FILE|MESSAGE> */
      {  
      int message=(strcmp(buffer,"FILE")!=0),max=areamax(),a,abort=0,
          line=0,nonstop=(user.f_user&USER_NOMORE);
      mail_area area;    

      if (message)
        {
        mprintf(1,"{bfgbg wb}  # {fg y}#Msg {fg c}Conference name               {fg w}  # {fg y}#Msg {fg c}Conference name{eol std}\n");
        }
      else
        {
        mprintf(1,"{bfgbg wb}  # {fg y}#Fil {fg c}Filebase name                 {fg w}  # {fg y}#Fil {fg c}Filebase name{eol std}\n");
        }

      for(a=1;a<max && abort==0;a++)
        {                  
        /* Read the area thingy! */
        read_area(a,&area);        

        if ((can_read(&area) || can_write(&area)) && area.name[0]!=0)
          {
          if ((message!=0 && (area.areaflags&FLAG_FILEAREA)==0) ||
              (message==0 && (area.areaflags&FLAG_FILEAREA)!=0))
            {
            if ((line%2)==0)
              {
              abort=mprintf(0,"{bfg w}%3d {fg y}%4d {fg c}%-30s",a,
                                            area.count,area.name);
              }
            else
              {
              abort=mprintf(0,"{bfg w}%3d {fg y}%4d {fg c}%s\n",a,
                                            area.count,area.name);
              }
            line++;
            if (line==(user.pagelen*2) && nonstop==0)
              {
              int ch;

              line=0;
              mprintf(1,"{bfg w}More: {fg c}[{fg y}Space{fg c}]{fg w}/{fg c}[{fg y}Return{fg c}]{fg w} continues, {fg c}[{fg y}N{fg c}]{fg w}onstop, {fg c}[{fg y}Ctrl-C{fg c}]{fg w} aborts{fg y}\015");
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
            }
          }
        }
      if (abort) port_txclear();
      port_crlf();

      break;
      }
    case 46: /* CONF_JOIN <confnr> */
      {  
      mail_area area; char confnr[5]; int conf;
                    
      if (buffer[0])
        {
        if (tolower(buffer[0])!='q') conf=atoi(buffer); else conf=atoi(buffer+1);
        }
      else
        {
        port_txstring("\n{bfg g}Conference/filebase to join: ",0);
        port_readline(confnr,4,NUMONLY); port_crlf();
        if (confnr[0]==0) break;                       
        conf=atoi(confnr);
        }

      if (conf<areamax())
        {                 
        read_area(conf,&area);
              
        if (can_read(&area) || can_write(&area))
          {
          get_user(&user);
          conf_join(user.conferences,conf);
          if (tolower(buffer[0])!='q')
            mprintf(1,"{bfg c}Joined area {fg w}%s{std}\n",area.name);
          put_user(&user);
          }
        else
          {
          if (tolower(buffer[0])!='q')
            mprintf(1,"{bfg r}Can't join that area{std}\n");
          }
        }

      break;
      }
    case 47: /* CONF_RESIGN <confnr> */
      {    
      char confnr[5]; int conf;

      if (buffer[0])
        {
        conf=atoi(buffer);
        }
      else
        {
        port_txstring("\n{bfg g}Conference/filebase to resign from: ",0);
        port_readline(confnr,4,NUMONLY); port_crlf();
        if (confnr[0]==0) break;                       
        conf=atoi(confnr);
        }     

      if (conf<areamax())
        {
        get_user(&user);
        conf_resign(user.conferences,conf);
        put_user(&user);
        }

      break;
      }
    case 48: /* CONF_SELECT <FILE|MESSAGE> [<confnr>] */
      {  
      int message=(strcmp(buffer,"FILE")!=0),max=areamax(),a,abort=0,
          line=0,nonstop=(user.f_user&USER_NOMORE);
      mail_area area;    
                   
      if (buffer2[0])
        {
        if (message)
          {
          read_area(conference=atoi(buffer2),&area);
          strcpy(conferencename,area.name);
          }
        else
          {
          read_area(filebase=atoi(buffer2),&area);
          strcpy(filebasename,area.name);
          }
        }
      else
        {
        char selbuff[5]; int te;

        if (message)
          {
          mprintf(1,"{bfgbg wb}  # {fg y}#Msg {fg c}Conference name               {fg w}  # {fg y}#Msg {fg c}Conference name{eol std}\n");
          }
        else
          {
          mprintf(1,"{bfgbg wb}  # {fg y}#Fil {fg c}Filebase name                 {fg w}  # {fg y}#Fil {fg c}Filebase name{eol std}\n");
          }

        for(a=1;a<max && abort==0;a++)
          {
          if (conf_member(user.conferences,a))
            {                                  
            /* Read the area thingy! */
            read_area(a,&area);        

            if ((message!=0 && (area.areaflags&FLAG_FILEAREA)==0) ||
                (message==0 && (area.areaflags&FLAG_FILEAREA)!=0))
              {
              if ((line%2)==0)
                {
                abort=mprintf(0,"{bfg w}%3d {fg y}%4d {fg c}%-30s",a,
                                            area.count,area.name);
                }
              else
                {
                abort=mprintf(0,"{bfg w}%3d {fg y}%4d {fg c}%s\n",a,
                                            area.count,area.name);
                }
              line++;
              if (line==(user.pagelen*2) && nonstop==0)
                {
                int ch;

                line=0;
                mprintf(1,"{bfg w}More: {fg c}[{fg y}Space{fg c}]{fg w}/{fg c}[{fg y}Return{fg c}]{fg w} continues, {fg c}[{fg y}N{fg c}]{fg w}onstop, {fg c}[{fg y}Ctrl-C{fg c}]{fg w} aborts{fg y}\015");
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
              }
            }
          }

        if (abort) port_txclear();
        port_crlf();
        port_txstring(message?"{bfg g}Conference : ":"{bfg g}Filebase : ",1);
        port_readline(selbuff,4,NUMONLY); port_crlf();
        if (selbuff[0]==0) break;
        te=atoi(selbuff);
        if (te!=0 && conf_member(user.conferences,te))
          {                                          
          read_area(te,&area);
          if ((message!=0 && (area.areaflags&FLAG_FILEAREA)==0) ||
              (message==0 && (area.areaflags&FLAG_FILEAREA)!=0))
            {
            if (message)
              {
              conference=te;
              strcpy(conferencename,area.name);
              }
            else
              {
              filebase=te;
              strcpy(filebasename,area.name);
              }
            }
          else
            {                  
            if (message)
              {
              port_txstring("\n{bfg r}Not a conference!{std}\n",1);
              }
            else
              {
              port_txstring("\n{bfg r}Not a filebase!{std}\n",1);
              }    
            }
          }
        }

      break;
      }
    case 49: /* USERLIST */
      {
      int abort=0,a; char search[31],searchtry[120],line[120],*lower; user_block temp;
      
      port_txstring("\n{bfg g}Username/partial username to search for (key [Return] to see all)\n: ",1);
      port_readline(search,30,TOLOWER); port_crlf();
                                           
      port_txstring("{bfgbg wb}#    {fg c}Username                       {fg g}Last logon           {fg w}Call Mail Upld Dnld Rg{eol std}\n",1);

      for(a=1;a<mod_extentuser() && abort==0;a++)
        {
        timepoll();
        mod_readuserdata(a,&temp);
        if (temp.status==1)
          {
          sprintf(line,"{bfg w}%04d {fg c}%-30s {fg g}%-20s {fg w}%4d %4d %4d %4d%s\n",a,temp.username,
                                                 cctime(&temp.t_lastlogon)+4,temp.logons,
                                                 temp.mailcount,temp.uploads,
                                                 temp.downloads,
                                                 (temp.f_user&USER_REGISTERED)?" Y":"");
          strcpy(lower=searchtry,line);

          /* Into lowercase */
          while(*lower) *lower++=tolower(*lower);

          /* Does it match? */
          if (strstr(searchtry,search)!=NULL) abort=mprintf(0,"%s",line);
          else abort=port_rxbuffer();
          }
        else
          {
          abort=port_rxbuffer();
          }
        }

      port_rxclear();
      port_txstring("{std}",1);
      break;
      }
    case 50: /* CLEARQUEUE */
      {
      queue_length=0;
      mprintf(1,"{bfg r}Your download queue has been cleared{std}\n");
      break;
      }
    case 51: /* DOWNQUEUE */
      {
      queue_download(1);
      break;
      }
    case 52: /* QUESTION <filename> */
      {
      questionnaire(buffer);
      break;
      }
    case 53: /* ORDERDISK */
      {
      order_disk();      
      break;
      }
    case 54: /* temp */
      {
      screen_edit();
      break;
      }
    case 55: /* CALL <port> */
      {
      int cport=atoi(buffer);
      clock_t w; os_regset r;

      /* Check to see if port is in use */
      port_select(cport);

      /* Check DTR level of port */
      r.r[0]=0; r.r[1]=0; r.r[2]=0xffffffff;
      os_swi(0x57,&r);

      if ((r.r[1]&8)==0)
        {
        port_select(portnumber);
        mprintf(1,"{bfg r}Sorry, the pass-through port is in use. Please try again later.{std}\n");
        break;
        }

      port_select(portnumber);
      mprintf(1,"{bfg r}Connecting...");
            
      /* Put DTR high on that port */
      gateway=1;
      port_select(gatewayport=cport);
      port_dtr(1); port_rts(1);

      /* Wait for DCD on port to go high */
      w=clock();
      while((clock()-w)<500)
        {
        port_select(cport);
        if (port_dcd()) break;
        window_poll();
        }
                    
      if (port_dcd()==0)
        {
        port_select(portnumber);
        mprintf(1,"connection could not be made{std}\n");
        break;
        }

      port_select(portnumber);
      mprintf(1,"connected{std}\n");

      /* Loop, transferring data, until DCD on port goes low */
      do
        {
        int data;

        port_select(cport);
        if (port_rxbuffer())
          {
          do
            {
            data=port_rx();
            port_select(portnumber);
            port_txw(data);
            port_select(cport);
            }
          while(port_rxbuffer());
          }

        port_select(portnumber);
        if (port_rxbuffer())
          {
          data=port_rx();
          port_select(cport);
          if (port_txbuffer()==0)
            {
            do
              {
              window_poll();
              port_select(cport);
              }
            while(port_txbuffer()==0);
            }
          port_txw(data);
          }

        window_poll();
        port_select(cport);
        }
      while(port_dcd());
      
      /* Drop DTR/RTS */
      port_rts(0); port_dtr(0);
      gateway=0;

      port_select(portnumber);
      mprintf(1,"{bfg r}Disconnected{std}\n");
      break;
      }
    case 56: /* SETMAIL */
      {
      int days,a,quiet=0,msgnr=read_msgnumber(); time_t t=time(NULL);
      mail_block header; struct tm *extime; char buff[80];
                                      
      if (tolower(buffer[0])=='q') { days=atoi(buffer+1); quiet=1; } else days=atoi(buffer);

      days=days*(24*60*60);
      t-=days;

      if (!quiet)
        {
        mprintf(1,"{bfg g}We're going...");
        wimp_waitfor(60);

        for(a=0;a<3;a++)
          {
          wimp_waitfor(40);     
          mprintf(1,"back..."); 
          }                     
        }

      while(msgnr>=0)
        {
        timepoll();

        header.message_number=msgnr; get_messageh(&header);
        if (header.date_entered<t && header.status!=0) break;
        msgnr-=1;
        }        
                                         
      get_user(&user);
      if (tolower(buffer2[0])=='m') user.m_message=msgnr;
      if (tolower(buffer2[0])=='f') user.m_file   =msgnr;
      put_user(&user);

      extime=localtime(&t);
      datetxt(buff,1900+extime->tm_year,1+extime->tm_mon,extime->tm_mday);
      if (!quiet) mprintf(1,"\nback to %s!{std}\n",buff);

      break;
      }
    case 57: /* SETSPEED */
      {
      int p=atoi(buffer),speed=atoi(buffer2);

      port_select(p);
      port_speed(speed,speed);
      port_parity(5);
      port_select(portnumber);

      break;
      }
    case 58: /* NEWMENU */
      {
      menu_initialise();
      do_menu(buffer);
      break;
      }
    case 59: /* SCANFORME */
      {
      do_forme();
      break;
      }
    case 60: /* DOOR <doornr> */
      {
      if (doors_connect(atoi(buffer)))
        {
        mprintf(1,"{bfg r}Unable to connect to door{std}\n");
        }
      break;
      }
    case 61: /* OSCLI <command> */
      {
      os_cli(buffer);
      break;
      }
    case 62: /* SCRIPT name */
      {
      script_run(buffer);
      break;
      }
    default:
      {
      mprintf(1,"Unknown command: %s %s\n",commandbuffer,buffer);
      break;
      }
    }
  }
