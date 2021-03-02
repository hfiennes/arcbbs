/*                  _____________________________________________
                  [>                                             <]
Project           [> PortNet communications network              <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Server module                               <]
Current version   [> 01.64                                       <]
Version date      [> 08-July-1993                                <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1989-1993 by    <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"

#define ONLINE_COL      11
#define OFFLINE_COL     10

#define VERSION        "1.64"
#define VERSION_STRING "1.64 (08-July-1993)"

#include "config.h"
#include "servmess.h"
#include "userlog.h"
#include "modcomm.h"
#include "mail.h"
#include "fido.h"
#include "chat.h"               
#include "stdarg.h"
#include "dboxquery.h"

/* PreDeclarations */
int  main(void),users_online(void);
void errorlog(os_error*,char*),window_init(void),window_closedown(void),
     start_slaves(void),start_slavetask(int),do_trim(void),trim_area(int,int,int),
     kill_slaves(void),kill_slavetask(int),write_icon(wimp_w,wimp_i,char*),
     send_message(int,int,int,int,int),start_edittask(int,int),
     read_icon(wimp_w,wimp_i,char*),mu_send(int,int,char*),
     open_swindow(wimp_w,int,int),set_icon(int,int),set_porticon(int,int,int),
     start_localtask(int),send_strmessage(int,int,int,int,int,char*),
     server_msgproc(wimp_eventstr*,void*),do_iconmenu(void*,char*),
     do_portmenu(void*,char*),start_areatask(int),start_uploadtask(int,int);
static menu do_makeport(void*);

/* Global variables */
wimp_w server_handle,verbose_handle,chat_handle,page_handle;
int  portstatus[64],portuser[64],count_online=0,count_other=0,noofcalls,chat,
     portspeed[64],porttask[64],noofcallstoday=0,mail_tx=0,mail_rx=0,file_tx=0,
     file_rx=0,paging=0,pageuser=0,muchat=0,portchannel[48],got_type,
     last_menu,chatting=0,typepos=0,linesloaded=0;
static int quit=0;
wimp_t slave_tasks[64],mail_tasks[64],mail_ports[64],ourtask;
char tempbuffer[256],sysopmessage[82],divertmenu[21],edit_usernr[9],
     typebuffer[21],__message_buffer[16384],*_message_buffer,*message_buffer,
     mailer_taskname[30];
FILE *calllog;
clock_t lastevent=0,lastkey=0,lastpoll=0,lasteventcheck=0;
menu portmenu=(menu)-1,message,divert;
mail_block *lastmsg;        

void window_poll()
  {
  event_process();
  lastpoll=clock();
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

void do_event()
  {
  /* Close BBS files */
  mod_closeall();
                                        
  /* Run the event */
  system("WimpTask <ARCbbs$event>"); 

  /* Reopen BBS files */
  mod_openall();

  /* Trim messageareas */
  do_trim();
  }

int main()
  {
  char buffer[12],*t; int a; user_list *ul;
  struct tm event;
  char eventtime[20]; int hr,min;

  /* Get the time of the event */
  strcpy(eventtime,getenv("ARCbbs$EventTime"));
  if (sscanf(eventtime,"%d:%d",&hr,&min)!=2)
    {
    hr=2; min=30;
    }

  _message_buffer=__message_buffer;
  message_buffer=(_message_buffer+9);

  /* Init the window stuff */
  window_init();

  for(a=0;a<48;a++)
    {                 
    ul=(user_list*)(mod_getstatuspointer()+(64*a));
    ul->baudrate=ul->usernumber=-1;
    ul->username[0]=ul->doing[0]=0;
    }

  /* Open files */
  noofcalls=mod_readcallcount();

  /* Setup flags */
  if (!strcmp(getenv("ARCbbs$Chat"),"Y")) chat=-1; else chat=0;
  write_icon(server_handle,12,"");
  write_icon(server_handle,20,VERSION);
  sprintf(buffer,"%d",noofcalls);
  write_icon(server_handle,11,buffer);
  set_icon(9,chat);

  /* Start processes if needed */
  if (!strcmp(getenv("ARCbbs$Usage"),"Y")) wimp_starttask("<ARCserver$Dir>.!Usage");
  if (!strcmp(getenv("ARCbbs$Spark"),"Y")) wimp_starttask("<ARCserver$Dir>.!Spark");
                    
  /* Initialise fido */
  fido_setup();

  /* Get name of mailer task, default to BinkleyTerm_%d */
  if ((t=getenv("ARCbbs$MailerTaskName"))!=NULL) strcpy(mailer_taskname,t);
  else strcpy(mailer_taskname,"BinkleyTerm_%d");

  /* Start specified slave tasks */
  start_slaves();

  /* Poll loop */
  while(quit==0)
    {
    window_poll();

    if (chatting)
      {
      /* Send new data every 1/5th second */
      if ((clock()-lastkey)>20 || typepos==19)
        {
        typebuffer[typepos]=0;
        send_strmessage(pageuser,BBS_CHATDATA,0,0,0,typebuffer);
        typepos=0; lastkey=clock();
        }
      }

    if ((clock()-lasteventcheck)>2000)
      {
      gtime(&event,time(NULL));
      if (event.tm_hour==hr && event.tm_min==min && (clock()-lastevent)>6000)
        {
        lastevent=clock();                   
        do_event();
        }
      lasteventcheck=clock();
      }
    }

  /* Kill slave tasks */
  if (quit!=2) kill_slaves();
       
  /* Closedown */
  exit(0);
  }

static void nullproc(wimp_i junk)
  {
  int task=9;
  junk=junk;

  /* Edit tasks are 9-12 */
  while(slave_tasks[task]!=-1) task++;
  if (task<=12) start_localtask(task);
  }

void window_init()
  {
  wimp_wind *window;
  static menu iconmenu,editnr,upload,fidonet,maint;

  /* Initialise WIMP library modules */
  wimpt_init("ARCserver");        /* Setup task */
  ourtask=wimpt_task();           /* Get taskid for messages */
  res_init("ARCserver");          /* Resource init */
  resspr_init();                  /* Sprite init */
  template_init();
  dbox_init();

  /* Create the windows */
  window=template_syshandle("verbose");
  wimp_create_wind(window,&verbose_handle);
  window=template_syshandle("server");
  wimp_create_wind(window,&server_handle);
  window=template_syshandle("chat");
  wimp_create_wind(window,&chat_handle);

  window=template_syshandle("page");
  wimp_create_wind(window,&page_handle);
  win_activeinc();

  /* Get on icon bar */
  baricon("!arcserver",(int)resspr_area(),nullproc);

  /* Attach event handlers */
  win_register_event_handler(server_handle,server_msgproc,0);
  win_register_event_handler(verbose_handle,server_msgproc,0);

  /* Route unknown/null events to that window */
  win_claim_unknown_events(server_handle);       
  win_claim_idle_events(server_handle);
  event_setmask(0); /*wimp_EMPTRENTER|wimp_EMPTRLEAVE);*/

  /* Create icon bar menu */
  iconmenu=menu_new("BBS Server",">Info,>Database status,Edit user,Brief status,Verbose status,Local logon,Upload,Maintenance,Ensure files,Quit");
  editnr=menu_new("User number:","blah");
  upload=menu_new("Upload","Batch upload,Local upload");
  maint=menu_new("Maintenance","Area edit,Trim messagebases,Run event,Fidonet");
  fidonet=menu_new("Fidonet","Import,Export");
  menu_submenu(maint,4,fidonet);
  menu_submenu(iconmenu,3,editnr);
  menu_submenu(iconmenu,7,upload);
  menu_submenu(iconmenu,8,maint);
  menu_make_writeable(editnr,1,edit_usernr,8,0);
  event_attachmenu(win_ICONBAR,iconmenu,do_iconmenu,0);
                          
  /* Attach other menus */
  event_attachmenumaker(verbose_handle,do_makeport,do_portmenu,(void*)0);
  event_attachmenumaker(server_handle,do_makeport,do_portmenu,(void*)1);

  /* Clear chat window */
  chat_clearmap();
  }

void open_swindow(wimp_w wind,int x,int y)
  {
  wimp_wstate state;

  wimp_get_wind_state(wind,&state);

  /* Open window to full size, ontop of everything else */
  state.o.x=x;
  state.o.y=y;
  state.o.behind=-1;

  wimp_open_wind(&state.o);
  }

void server_msgproc(wimp_eventstr *e,void *handle)
  {                          
  static clock_t lastbeep;

  handle=handle;

  if (paging)
    {
    if ((clock()-lastbeep)>99)
      {
      bbc_vdu(7); lastbeep=clock();
          
      if (!--paging)
        {                           
        /* End of paging, close window */
        wimp_close_wind(page_handle);

        send_message(pageuser,BBS_CHATREQUEST,NO_CHATTING,0,0);
        }
      }
    }

  switch(e->e)
    {
    case wimp_EOPEN:    /* Open_Window_Request */
      {
      wimp_open_wind(&e->data.o);
      break;
      }
    case wimp_ECLOSE:   /* Close_Window_Request */
      {
      wimp_close_wind(e->data.o.w);
      if (e->data.o.w==chat_handle)
        {     
        if (chatting)
          {
          send_message(pageuser,BBS_CHATREQUEST,NO_CHATTING,0,0);
          chatting=0;
          }
        }
      break;
      }
    case wimp_EREDRAW:  /* Redraw window */
      {
      if (e->data.o.w==chat_handle) chat_redraw();
      break;
      }
    case wimp_EBUT:     /* Mouse click */
      {
      int window;
      if (e->data.but.m.w==server_handle)  window=1;
      if (e->data.but.m.w==page_handle)    window=2;
      if (e->data.but.m.w==chat_handle)    window=3;

      if(e->data.but.m.bbits==1 || e->data.but.m.bbits==4)   /* Left/right button */
        {
        switch(window)
          {
          case 1: /* Server window */
            {
            switch(e->data.but.m.i)
              {
              case 0: case 1: case 2:
              case 3: case 4: case 5:
              case 6: case 7: case 8:
              /* Toggle line indicators */
                {
                int port=e->data.but.m.i;
                char buffer[10];

                portstatus[port]=portstatus[port]?0:1;

                if (portstatus[port])
                  {
                  start_slavetask(port);
                  write_icon(verbose_handle,(4*port)+3,"Offline");
                  }
                else
                  {
                  kill_slavetask(port);
                  write_icon(server_handle,15,buffer);
                  write_icon(verbose_handle,(4*port)+3,"Not loaded");
                  }

                write_icon(verbose_handle,(4*port)+1,"Off");
                set_porticon(port,portstatus[port],OFFLINE_COL);
              
                break;
                }
              case 9:
              /* Chat dis/enabled */
                {
                chat=!chat;
                break;
                }
              }
            break;
            }
          case 2: /* Page window */
            {
            switch(e->data.but.m.i)
              {
              case 1: /* Accept chat */
                {
                /* Accepted chat request */
                wimp_close_wind(page_handle);
                send_message(pageuser,BBS_CHATREQUEST,ACCEPT,0,0);
                chat_clearmap();
                open_swindow(chat_handle,0,0);
                paging=0; chatting=1; typepos=0;
                break;
                }
              case 2: /* Ignore chat */
                {
                /* Send back no chatting msg to slave */
                wimp_close_wind(page_handle);
                send_message(pageuser,BBS_CHATREQUEST,NO_CHATTING,0,0);
                paging=0;
                break;
                }
              case 3: /* Excuse */
                {                                        
                char excuse[80]; int a;
                                             
                wimp_close_wind(page_handle);
                /* Read excuse */

                read_icon(page_handle,4,excuse); 

                /* Look for funnies as terminators */               
                for(a=0;a<80;a++)
                  {
                  if (excuse[a]<32) excuse[a]=0;
                  }
                window_poll();
                /* Send back excuse to slave */
                send_strmessage(pageuser,BBS_CHATREQUEST,EXCUSE,0,0,excuse);
                paging=0;
                break;
                }
              }
            break;
            } 
          case 3: /* Chat window, gain caret */
            {
            wimp_caretstr temp;

            temp.w=chat_handle; temp.i=-1;
            temp.x=chat_cursorx<<4;
            temp.y=-(chat_cursory<<5)-32;
            temp.height=32;
            temp.index=0;

            /* Stick caret in that window */
            wimp_set_caret_pos(&temp);
              
            break;
            }
          }
        }   
      break;
      }                  
    case wimp_EGAINCARET:
      {
      wimp_caretstr temp;

      temp.w=chat_handle; temp.i=-1;
      temp.x=chat_cursorx<<4;
      temp.y=-(chat_cursory<<5)-32;
      temp.height=32;
      temp.index=0;

      /* Stick caret in that window */
      wimp_set_caret_pos(&temp);
  
      break;
      }
    case wimp_EKEY:     /* Key pressed */
      {
      if (chatting)
        {       
        int key=e->data.key.chcode;

        if (key==13) { key=10; chat_data(13); }
        if (key>31 && key!=127) chat_data(key+256); else chat_data(key);
        typebuffer[typepos++]=key;
        }
      break;
      }
    case 17:            /* Incoming message */
    case 18:
      {
      int word0=e->data.msg.data.words[0],
          word1=e->data.msg.data.words[1],
          word2=e->data.msg.data.words[2],
          mtask=e->data.msg.hdr.task;

      char buffer[40];

      switch(got_type=e->data.msg.hdr.action)
        {
        /* Kill task? */
        case 0:
          {
          quit=2;
          break;
          }
        /* Task starting up */
        case wimp_MINITTASK:
          {          
          int port; char *taskname=e->data.msg.data.chars+8,buffer[20];
                       
          /* Online/Local task startup */
          if (sscanf(taskname,"ARCbbs_%d",&port)==1)
            {
            slave_tasks[port]=mtask;

            /* Mail BBS slave? */
            if (taskname[strlen(taskname)-1]=='M') mail_tasks[port]=mtask;

            linesloaded++; sprintf(buffer,"%d",++count_online); write_icon(server_handle,15,buffer);

            if (strcmp(getenv("ARCbbs$system"),"cryton")!=0)
              {
              if (linesloaded>MAXIMUM_LINES) kill_slavetask(port);
              else send_message(port,BBS_SERVERIS,0,0,0);
              }
            else send_message(port,BBS_SERVERIS,0,0,0);
            
            break;
            }                            
                               
          /* Mailer startup */
          if (sscanf(taskname,mailer_taskname,&port)==1)
            {
            mail_tasks[port]=mtask;
            break;
            }

          /* User editor startup */
          if (sscanf(taskname,"ARCedit_%d",&port)==1)
            {
            slave_tasks[port]=mtask;
            send_message(port,BBS_SERVERIS,0,0,0);
            break;
            }
          break;
          }
        case wimp_MCLOSETASK: 
          {
          int a; char buffer[20];

          /* BBS slave dying? */
          for(a=0;a<64;a++)
            {
            if (slave_tasks[a]==mtask)
              {  
              if (a<48)
                {
                if (mail_tasks[a]==mtask)
                  {
                  /* It's a mail slave */
                  mail_tasks[a]=-1;
                
                  /* Restart mailer for that port */
                  sprintf(tempbuffer,"<ARCbbs$Mailer> %d",a);
                  wimp_starttask(tempbuffer);
                  }
                }
              slave_tasks[a]=-1;

              linesloaded--;
              sprintf(buffer,"%d",--count_online); write_icon(server_handle,15,buffer);

              break;
              }
            } 

          /* Mailer dying? */
          for(a=0;a<48;a++)
            {
            if (mail_tasks[a]==mtask)
              {
              int retcode;
              mail_tasks[a]=-1;

              /* Check the returncode */
              sprintf(tempbuffer,"Binkley$Return%d",a);
              switch(retcode=atoi(getenv(tempbuffer)))
                {
                case 0:
                case 1:    /* Ctrl-X exit */
                  {
                  break;
                  }
                case 3:    /* Spawn the BBS */
                case 12:
                case 24:
                case 48:
                case 96:
                case 144:
                case 168:
                case 192:
                  {
                  char runbbs[60];

                  sprintf(runbbs,"<ARCserver$Dir>.!ARCbbs m %d",a);
                  wimp_starttask(runbbs);
                  window_poll(); window_poll();
                  break;
                  }
                case 5: /* Inbound ARCmail received */
                case 6: /* Inbound mail */
                  {
                  fido_dearc();
                  fido_doinbound("*PT");
                  fido_doinbound("*_P");
                  fido_dooutbound();
                  wimp_starttask("<ARCbbs$Mailer>.ArcMail");

                  /* Restart mailer for that port */
                  sprintf(tempbuffer,"<ARCbbs$Mailer> %d",a);
                  wimp_starttask(tempbuffer);
                  break;
                  }
                case 7: /* Process outbound mail */
                  {
                  fido_dooutbound();
                  wimp_starttask("<ARCbbs$Mailer>.ArcMail");

                  /* Restart mailer for that port */
                  sprintf(tempbuffer,"<ARCbbs$Mailer> %d",a);
                  wimp_starttask(tempbuffer);
                  break;
                  }
                default: /* Unknown other */
                  { 
                  /* Pass code to batchfile */
                  sprintf(tempbuffer,"Run <ARCbbs$Mailer>.Other_Ret %d",a);
                  system(tempbuffer);

                  /* Restart mailer for that port */
                  sprintf(tempbuffer,"<ARCbbs$Mailer> %d",a);
                  wimp_starttask(tempbuffer);
                  break;
                  }
                }
              break;
              }
            }
          break;
          }
        /* Slave sending username ? */
        case BBS_USERID:
          {
          if (word0<48)
            {
            /* Put username into window */
            write_icon(verbose_handle,4*word0,e->data.msg.data.chars+12);
            }
          break;
          }        
        /* Sysop mail */
        case BBS_SYSOPMAIL:
          {
          if (chat)
            {
            dbox sysopmail=dbox_new("sysopmail");
            clock_t now=clock();
                           
            dbox_setfield(sysopmail,1,e->data.msg.data.chars+12);
            dbox_showstatic(sysopmail);
            while((clock()-now)<400) window_poll();
            dbox_dispose(&sysopmail);
            }

          break;
          }
        /* Slave sending new taskinfo */
        case BBS_USERTASK:
          {
          if (word0<48)
            {
            strcpy(buffer,e->data.msg.data.chars+12);
            buffer[16]=0;

            /* Put task into window */
            write_icon(verbose_handle,(4*word0)+3,buffer);
            wimp_set_icon_state(verbose_handle,(4*word0)+3,2<<28,0xf0000000);
            porttask[word0]=word1;
            }
          break;
          }
        /* Slave to alert sysop? */
        case BBS_ALERT:
          {
          wimp_set_icon_state(verbose_handle,(4*word0)+3,11<<28,0xf0000000);
          bbc_vdu(7);
          break;
          }
        /* Slave sending new icon info? */
        case BBS_ICONINFO:
          {
          portuser[word0]=word1;
          portspeed[word0]=word2;
          if (word0<48)
            {
            if (portspeed[word0])
              {
              set_porticon(word0,1,ONLINE_COL);
              sprintf(buffer,"%d",portspeed[word0]);
              write_icon(verbose_handle,(4*word0)+1,buffer);
              write_icon(verbose_handle,(4*word0)+3,"Online");
              }
            else
              {
              set_porticon(word0,1,OFFLINE_COL);
              write_icon(verbose_handle,(4*word0)+1,"Off");
              write_icon(verbose_handle,(4*word0)+3,"Offline");
              }
            }
          break;
          }
        /* Change online count? */
        case BBS_COUNTONLINE:
          {
          if (e->data.msg.data.words[0]==1)
            {
            noofcalls++; noofcallstoday++;
            mod_inccallcount();
            sprintf(buffer,"%d",noofcalls);
            write_icon(server_handle,11,buffer);
            sprintf(buffer,"%d",noofcallstoday);
            write_icon(server_handle,10,buffer);
            }
          break;
          }
        /* Change 'other' count? */
        case BBS_COUNTOTHER:
          {
          break;
          }
        case BBS_STATUS:
          {
          /* Send information message */
          send_message(word0,BBS_STATUS,count_online,0,0);

          break;
          }
        case BBS_CHATDATA:
          {
          char *pointer=e->data.msg.data.chars+12;

          /* Display data */
          while(*pointer)
            {                                   
            if (*pointer==10) chat_data(13);
            chat_data(*pointer++);
            }
          break;
          }
        case BBS_CHATREQUEST:
          {
          if (word1==PAGING)
            {
            if (!chat)
              {
              /* Send 'no chatting' message */
              send_message(word0,BBS_CHATREQUEST,NO_CHATTING,0,0);
              }
            else
              {
              if (paging || chatting)
                {
                send_message(word0,BBS_CHATREQUEST,ALREADY_CHATTING,0,0);
                }
              else
                {
                user_list *ul;

                /* Send 'paging' message */
                paging=10; lastbeep=0; pageuser=word0;
                send_message(word0,BBS_CHATREQUEST,PAGING,0,0);
                
                /* Copy name into paging window */
                ul=(user_list*)(mod_getstatuspointer()+(64*word0));
                write_icon(page_handle,0,ul->username);
                  
                /* Open page window */
                open_swindow(page_handle,350,400);
                }
              }
            }
          else
            {
            wimp_close_wind(chat_handle);
            chatting=paging=0;
            }
          break;
          }
        case BBS_MUCONTROL:
          {
          user_list *ul=(user_list*)(mod_getstatuspointer()+(64*word0));

          if (word1==MU_JOIN)
            {
            /* Setup chat user in table */
            portchannel[word0]=word2;

            /* Send joined message */
            sprintf(tempbuffer,"\001{bfgbg wb} MultiUser Chat {bg n} %s has arrived\012",ul->username);
            mu_send(word2,-1,tempbuffer);
            }
          else
            {
            /* Clear chat user in table */
            portchannel[word0]=0;

            /* Send left message */
            sprintf(tempbuffer,"\001{bfgbg wb} MultiUser Chat {bg n} %s has left\012",ul->username);
            mu_send(word2,-1,tempbuffer);
            }
          break;
          }
        case BBS_MUDATA:
          {
          if (word1==MU_PUBLIC)
            {
            mu_send(portchannel[word0],word0,e->data.msg.data.chars+12);
            }
          else
            {
            send_strmessage(word2,BBS_MUDATA,0,0,0,
                                        e->data.msg.data.chars+12);
            }
          break;
          }
        case BBS_PUTLOG:
          {
          if ((calllog=fopen("<ARCbbs$miscdata>.CallLog","a"))==NULL)
            {
            werr(1,"Could not write new activity log file");
            }
          fprintf(calllog,"%s\n",e->data.msg.data.chars+12);
          fclose(calllog);

          break;
          }
        case BBS_MESSAGE:
          {
          if (portspeed[word1]!=0 && porttask[word1]==0)
            {
            send_strmessage(word1,BBS_MESSAGE,0,0,0,e->data.msg.data.chars+12);
            }
          break;
          }
        case BBS_READUSER:
          {
          if (portspeed[word0]!=0)
            {
            send_message(word0,BBS_READUSER,0,0,0);
            }
          break;
          }
        case BBS_FILECOUNTS:
          {
          char buffer[10];

          file_tx+=word0;
          file_rx+=word1;
          
          sprintf(buffer,"%d",file_tx);
          write_icon(server_handle,18,buffer);
          sprintf(buffer,"%d",file_rx);
          write_icon(server_handle,19,buffer);
          break;
          }
        }
      break;
      }
    }
  }

void mu_send(int channel,int from,char *message)
  {
  int a=0;
  for (;a<48;a++)
    {
    if (portchannel[a]==channel && a!=from)
      {
      send_strmessage(a,BBS_MUDATA,0,0,0,message);
      }
    }
  }

void start_slaves()
  {
  int a; char *b,prtcpy[20];

  /* Set everything -closed- */
  for(a=0;a<48;a++)
    { portstatus[a]=0; portuser[a]=0; portspeed[a]=0; portchannel[a]=0; }

  /* Set all taskhandles =-1 */
  for(a=0;a<64;a++) slave_tasks[a]=-1;

  /* Set all mail ports off */
  for(a=0;a<48;a++) { mail_ports[a]=0; mail_tasks[a]=-1; }

  /* Read port string so we can see what ports should be up */
  b=getenv("ARCbbs$mailports");
  while(*b) mail_ports[(*b++)-'0']=1;

  strcpy(b=prtcpy,getenv("ARCbbs$ports"));
  while(*b)
    {      
    int port=(*b++)-'0';
                   
    if (port>=0 && port<=9)
      {
      portstatus[port]=1;
      set_porticon(port,1,OFFLINE_COL);
      start_slavetask(port); 
      }
    }
  }

void kill_slaves()
  {
  int a;
  for(a=0;a<64;a++)
    {
    if (slave_tasks[a]!=-1 || mail_tasks[a]!=-1)
      {
      kill_slavetask(a);
      }
    }
  }

void start_slavetask(int port)
  {
  char buffer[40];

  if (mail_ports[port])
    {
    sprintf(buffer,"<ARCbbs$Mailer> %d",port);
    }
  else
    {
    sprintf(buffer,"<ARCserver$Dir>.!ARCbbs n %d",port);
    }
  wimp_starttask(buffer);
  window_poll(); window_poll();
  }

void start_localtask(int tasknumber)
  {
  char buffer[40];
  sprintf(buffer,"<ARCserver$Dir>.!ARCbbs l %d",tasknumber);
  errorlog(wimp_starttask(buffer),"StrtTask"); 
  window_poll(); window_poll();
  }

void kill_slavetask(int task)
  {
  wimp_msgstr killmessage;

  if (slave_tasks[task]!=-1)
    {
    killmessage.hdr.size=20;
    killmessage.hdr.your_ref=0;
    killmessage.hdr.action=BBS_QUIT; /* Kill task */
    wimp_sendmessage(wimp_ESEND,&killmessage,slave_tasks[task]);
    window_poll(); window_poll();
    }

  if (task<48)
    {
    if (mail_tasks[task]!=-1)
      {
      killmessage.hdr.size=20;
      killmessage.hdr.your_ref=0;
      killmessage.hdr.action=0; /* Kill task */
      wimp_sendmessage(wimp_ESEND,&killmessage,mail_tasks[task]);
      }
    }
  }

void start_edittask(int user,int tasknumber)
  {
  char buffer[50];
  sprintf(buffer,"<ARCserver$Dir>.!ARCedit %d %d",user,tasknumber);
  errorlog(wimp_starttask(buffer),"StrtTask");
  }

void start_areatask(int tasknumber)
  {
  char buffer[50];
  sprintf(buffer,"<ARCserver$Dir>.!AREAedit %d %d",ourtask,tasknumber);
  errorlog(wimp_starttask(buffer),"StrtTask");
  }

void start_uploadtask(int tasknumber,int batch)
  {
  char buffer[50];
  sprintf(buffer,"<ARCserver$Dir>.!%supload %d %d",batch?"BAT":"LOC",ourtask,
                                                                tasknumber);
  errorlog(wimp_starttask(buffer),"StrtTask");
  }

void send_message(int task,int action,int word1,int word2,int word3)
  {
  wimp_msgstr message;
  message.hdr.size=32;
  message.hdr.your_ref=0;
  message.hdr.action=action;
  message.data.words[0]=word1;
  message.data.words[1]=word2;
  message.data.words[2]=word3;
  errorlog(wimp_sendmessage(wimp_ESEND,&message,slave_tasks[task]),
                                                            "SendMsg1");
  }

void send_strmessage(int task,int action,int word1,int word2,int word3,char *string)
  {
  wimp_msgstr message;

  message.hdr.size=4*((32+strlen(string))/4+1);
  message.hdr.your_ref=0;
  message.hdr.action=action;
  message.data.words[0]=word1;
  message.data.words[1]=word2;
  message.data.words[2]=word3;
  strcpy(message.data.chars+12,string);
  wimp_sendmessage(wimp_ESEND,&message,slave_tasks[task]);
  }

void window_closedown()
  {         
  exit(0);
  }

void write_icon(wimp_w window,wimp_i icon,char *string)
  {
  wimp_icon isblock;
  wimp_icreate iblock;

  /* Get icon info */
  wimp_get_icon_info(window,icon,&isblock);

  if (isblock.flags & 0x100)
    {
    strcpy(isblock.data.indirecttext.buffer,string);
    }
  else
    {
    strcpy(isblock.data.text,string);
    wimp_delete_icon(window,icon);
    iblock.w=window;
    iblock.i=isblock;
    wimp_create_icon(&iblock,&icon);
    }
  wimp_set_icon_state(window,icon,0,0);
  }
 
void errorlog(os_error *err,char *part)
  {
  os_error *err2;
  if (err==NULL) return;
  err2=wimp_reporterror(err,3,part);
  err2=wimp_closedown();
  }

void read_icon(wimp_w window,wimp_i icon,char *string)
  {
  wimp_icon isblock;

  /* Get icon info */
  wimp_get_icon_info(window,icon,&isblock);

  if (isblock.flags & 0x100)
    {
    strcpy(string,isblock.data.indirecttext.buffer);
    }
  else
    {
    strcpy(string,isblock.data.text);
    }
  }

void set_icon(int icon,int onoff)
  {
  wimp_set_icon_state(server_handle,icon,onoff?0:1<<21,1<<21);
  }

void set_porticon(int line,int loaded,int online)
  {
  wimp_icon upd; wimp_icreate cr; wimp_i junk;

  wimp_get_icon_info(server_handle,line,&upd);
  upd.flags&=0xf0ffffff;
  upd.flags|=online<<24; 
  wimp_delete_icon(server_handle,line);
  cr.w=server_handle; 
  cr.i=upd;
  wimp_create_icon(&cr,&junk);   
  set_icon(line,loaded);
  }

void do_iconmenu(void *handle,char *hit)
  {
  handle=handle;

  switch(hit[0])
    {
    case 1:
    /* Info */
      {
      dbox d=dbox_new("info");
                         
      dbox_setfield(d,3,VERSION_STRING);
      dbox_showstatic(d);
      dbox_fillin(d);
      dbox_dispose(&d);

      break;
      }
    case 2:
    /* Database status */
      {
      dbox d=dbox_new("mailfree");
      char *lookupmap=mod_lookupmap(),*datamap=mod_datamap();
      int lookupmaplen=mod_lookuplength(),datamaplen=mod_datalength(),
          lookuplen=mod_lookupext(),datalen=mod_dataext(),
          lookup_mapusage,data_mapusage,lookup_fileusage,data_fileusage,
          lookup_activecount=0,lookup_highest=mod_getlastlookup()+1;
      char temp[20];
      int a,b,c;
 
      /*** Count active bits in lookupmap file */
      for(a=0;a<lookupmaplen;a++)
        {
        b=lookupmap[a];
        for(c=0;c<8;c++)
          {
          if (b&1) lookup_activecount++;
          b>>=1;
          }
        }

      /*** Usage of map files (map size vs data size) */
      lookup_mapusage=(lookuplen*100)/(lookupmaplen*8);
      data_mapusage=(datalen*100)/datamaplen;

      sprintf(temp,"%d%%",lookup_mapusage);
      dbox_setfield(d,0,temp);
      sprintf(temp,"%d%%",data_mapusage);
      dbox_setfield(d,5,temp);

      /*** Usage of data files (blocks used vs data size) */
      lookup_fileusage=(lookup_activecount*100)/lookuplen;
      for(a=0,b=0;a<datamaplen;a++) if (datamap[a]) b++;
      data_fileusage=(b*100)/datalen;

      sprintf(temp,"%d%%",lookup_fileusage);
      dbox_setfield(d,1,temp);
      sprintf(temp,"%d%%",data_fileusage);
      dbox_setfield(d,6,temp);

      /*** Highest msg number */
      dbox_setnumeric(d,2,lookup_highest);

      /*** Number of active messages */
      dbox_setnumeric(d,3,lookup_activecount);

      /*** Number of dead (cannot be used for public msgs/files) messages */
      dbox_setnumeric(d,4,lookup_highest-lookup_activecount);

      dbox_showstatic(d);
      dbox_fillin(d);
      dbox_dispose(&d);

      break;
      }
    case 3:
    /* Edit a user */
      {
      int task=48;
    
      /* Edit tasks are 48-63 */
      while(slave_tasks[task]!=-1) task++;
      if (task<=63)
        {
        switch(hit[1])
          {
          case 0:
            {
            start_edittask(0,task);
            break;
            }
          case 1:
            {
            start_edittask(atoi(edit_usernr),task);
            break;
            }
          }
        }
      break;
      }
    case 4:
    /* Brief list */
      {
      open_swindow(server_handle,500,0);
      break;
      }
    case 5:
    /* Verbose list */
      {
      open_swindow(verbose_handle,500,0);
      break;
      }
    case 6:
    /* Local logon */
      {
      int task=32;
    
      /* Edit tasks are 32-47 */
      while(slave_tasks[task]!=-1) task++;
      if (task<=47) start_localtask(task);
      break;
      }
    case 7:
    /* Upload file(s) */
      { 
      switch(hit[1])
        {
        case 1:
        /* Batch upload */
          {
          int task=48;
    
          /* Edit tasks are 48-63 */
          while(slave_tasks[task]!=-1) task++;
          if (task<=63) start_uploadtask(task,1);
          break;
          }
        case 2:
          {
          /* Local upload */
          int task=48;
    
          /* Edit tasks are 48-63 */
          while(slave_tasks[task]!=-1) task++;
          if (task<=63) start_uploadtask(task,0);
          break;
          }
        }
      break;
      }
    case 8:
    /* Maintenance */
      {
      switch(hit[1])
        {
        case 1:
        /* Area edit */
          {
          int task=48;
    
          /* Edit tasks are 48-63 */
          while(slave_tasks[task]!=-1) task++;
          if (task<=63) start_areatask(task);
          break;
          }
        case 2:
        /* Trim */
          {
          if (dboxquery("Are you sure you want to trim the messageareas?")==1) do_trim();
          break;
          }
        case 3:
        /* Run event */
          {
          if (dboxquery("Are you sure you want to run the event?")==1) do_event();
          break;
          }
        case 4:
        /* Fido */
          {
          switch(hit[2])
            {
            case 1:
            /* Import */
              {
              fido_dearc();
              fido_doinbound("*PT");
              fido_doinbound("*_P");
              break;
              }
            case 2:
            /* Export */
              {
              fido_dooutbound();
              wimp_starttask("<ARCbbs$Mailer>.ArcMail");
              break;
              }
            }
          break;
          }
        }
      break;
      }
    case 9:
    /* Ensure files */
      {
      mod_saveall();
      break;
      } 
    case 10:
    /* Quit */
      {
      if (users_online())
        {
        if (dboxquery("There are users online, sure you want to quit?")==1) quit=1;
        else quit=0;
        }
      else quit=1;

      if (quit)
        {
        kill_slaves();
        exit(0);
        }
      break;
      }
    }
  }

static menu do_makeport(void *handle)
  {
  wimp_mousestr where;
  char body[80],title[20];

  handle=handle; last_menu=-1;
  wimp_get_point_info(&where);

  if (where.w==server_handle)
    {
    if (where.i< 9) last_menu=where.i;
    }
  if (where.w==verbose_handle)
    {
    if (where.i<128) last_menu=where.i/4;
    }
                              
  if (last_menu==-1) return(NULL); 

  if (portmenu!=(menu)-1) menu_dispose(&portmenu,1);

  /* Port-specific menu */
  sprintf(title,"Port %d",last_menu);
  sprintf(body,"Edit user,Message,Divert,Add 10 mins,Add upload,Chat with user,Snoop on user,%s",portspeed[last_menu]==0?"Answer call":"Drop line");
  portmenu=menu_new(title,body);
  message=menu_new("Message:","msg");
  menu_make_writeable(message,1,sysopmessage,81,0);
  menu_submenu(portmenu,2,message);
  divert=menu_new("Divert to:","msg");
  menu_make_writeable(divert,1,divertmenu,20,0);
  menu_submenu(portmenu,3,divert);

  menu_setflags(portmenu,1,0,portspeed[last_menu]==0);
  menu_setflags(portmenu,2,0,portspeed[last_menu]==0);
  menu_setflags(portmenu,3,0,portspeed[last_menu]==0);
  menu_setflags(portmenu,4,0,portspeed[last_menu]==0);
  menu_setflags(portmenu,5,0,portspeed[last_menu]==0);
  menu_setflags(portmenu,6,0,(portspeed[last_menu]==0 || porttask[last_menu]!=0));
  menu_setflags(portmenu,7,0,portspeed[last_menu]==0);
  menu_setflags(portmenu,8,0,slave_tasks[last_menu]==-1);

  return(portmenu);
  }

void do_portmenu(void *handle,char *hit)
  {
  handle=handle;

  if (last_menu!=-1)
    {
    switch(hit[0])
      {
      case 1:
      /* Edit online user */
        {
        int task=10;
   
        /* Edit tasks are 48-63 */
        while(slave_tasks[task]!=-1) task++;
        if (task<=63) start_edittask(portuser[last_menu],task);
        break;
        }
      case 2:
      /* Message to user */
        {
        switch(hit[1])
          {
          case 1:
            {
            char temp[80];

            sprintf(temp," Message from Sysop: %s",sysopmessage);
            send_strmessage(last_menu,BBS_MESSAGE,0,0,0,temp);
            break;
            }
          }
        break;
        }
      case 3:
      /* Divert to menu */
        {
        switch(hit[1])
          {
          case 1:
            {
            send_strmessage(last_menu,BBS_DIVERT,0,0,0,divertmenu);
            break;
            }
          }
        break;
        }
      case 4:
      /* Add 10 mins */
        {
        send_message(last_menu,BBS_TIME,10,0,0);
        break;
        }
      case 5:
      /* Add upload */
        {
        send_message(last_menu,BBS_RATIO,0,0,0);
        break;
        }
      case 6:
      /* Chat */
        {
        send_message(last_menu,BBS_FORCECHAT,0,0,0);
        open_swindow(chat_handle,0,0);
        pageuser=last_menu; paging=0; chatting=1;
        break;
        }
      case 7:
      /* Snoop */
        {
        send_message(last_menu,BBS_SNOOP,0,0,0);
        break;
        }
      case 8:
      /* Answer/drop */
        {
        send_message(last_menu,BBS_DROPUSER,0,0,0);
        break;
        }
      }
    }
  }

int users_online()
  {
  char buffer[16];
  int a,ol=0;

  for(a=0;a<48;a++)
    {
    if (a<9)
      {
      read_icon(verbose_handle,(4*a)+1,buffer);
      if (atoi(buffer)>0) ol++;
      }
    }
  
  return(ol);
  }

void do_trim()
  {
  FILE *trimfile; char line[80];
  int area,maxnr;

  if ((trimfile=fopen("<ARCbbs$MiscData>.Trim","r"))==NULL) return;

  while(fgets(line,80,trimfile)!=NULL)
    {
    if (line[0]!=';')
      {
      if (sscanf(line,"%d %d",&area,&maxnr)==2) trim_area(area,maxnr,0);
      }
    }

  fclose(trimfile);
  }
