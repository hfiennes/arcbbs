/*                  _____________________________________________
                  [>                                             <]
Project           [> PortNet communications network              <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Area edit module                            <]
Current version   [> 00.17                                       <]
Version date      [> 27-October-1992                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"

/* Other headers */
#include "userlog.h"
#include "mail.h"
#include "servcomm.h"
#include "servmess.h"

#define IC event->data.but.m.i

/* PreDeclarations */
int  main(int,char**),get_iconstate(int);
void errorlog(os_error*,char*),window_init(void),
     write_icon(int,char*),read_icon(int,char*),open_window(void),
     from_icon(void),set_icon(int,int),open_window2(wimp_w,int,int),
     edit_msgproc(wimp_eventstr*,void*),window_poll(void),
     do_editmenu(void*,char*);
menu do_makeedit(void*);

/* Global variables */
int     got_data,got_code,areanumber=0,blank=0,portnumber;
wimp_t  servertask;
wimp_w  area_handle;
mail_area area,area2;

char msgmenu[1024],filemenu[1024],*message_buffer,*_message_buffer;
char msgflag[32],fileflag[32];
menu menu_msg,menu_file;  
int  noof_msgflags,noof_fileflags;

void send_message(int task,int action,int word1,int word2,int word3,int word4)
  {
  wimp_msgstr message;

  message.hdr.size=36;
  message.hdr.your_ref=0;
  message.hdr.action=action;
  message.data.words[0]=word1;
  message.data.words[1]=word2;
  message.data.words[2]=word3;
  message.data.words[3]=word4;
  errorlog(wimp_sendmessage(wimp_ESEND,&message,task),"SendMsg1");
  }

int main(int argc,char *argv[])
  {
  if (argc!=3)
    {
    return(0);
    }
  /* Read server task handle from environment string */
  servertask=atoi(argv[1]);

  /* Read task number from environment string */
  portnumber=atoi(argv[2]);

  /* Initialise wimp */
  window_init();

  /* Send message to server with OUR taskid */
  send_message(servertask,BBS_TASKID,portnumber,0,0,0);

  read_area(areanumber,&area);

  /* Open window */
  open_window(); open_window2(area_handle,0,0);

  while(1) event_process();

  return(0);
  }

void window_init()
  {
  wimp_wind *window;
  FILE *f; char *p,line[80]; int a,b;

  /* Initialise WIMP library modules */
  wimpt_init("AREAedit");         /* Setup task */
  res_init("AREAedit");           /* Resource init */
  template_init();

  window=template_syshandle("area");
  wimp_create_wind(window,&area_handle);
  win_activeinc();

  /* Attach event handler */
  win_register_event_handler(area_handle,edit_msgproc,0);
  win_claim_unknown_events(area_handle);

  /* Read flag files */
  if ((f=fopen("<ARCbbs$MiscData>.Flags.Message","r"))==NULL) exit(0);
  p=msgmenu; b=0;
  do
    {
    if (!fgets(line,79,f)) break;
    if (sscanf(line,"%d %s",&a,p)==2)
      {
      msgflag[b++]=a; p+=strlen(p);
      *p++=',';
      }
    }
  while(!feof(f));
  fclose(f); *--p=0; noof_msgflags=b;
  menu_msg=menu_new("Msg flags",msgmenu);

  if ((f=fopen("<ARCbbs$MiscData>.Flags.File","r"))==NULL) exit(0);
  p=filemenu; b=0;
  do
    {
    if (!fgets(line,79,f)) break;
    if (sscanf(line,"%d %s\n",&a,p)==2)
      {
      fileflag[b++]=a; p+=strlen(p);
      *p++=',';
      }
    }
  while(!feof(f));
  fclose(f); *--p=0; noof_fileflags=b;
  menu_file=menu_new("File flags",filemenu);
               
  event_attachmenumaker(area_handle,do_makeedit,do_editmenu,0);
  }

menu do_makeedit(void *h)
  {
  int a;
  h=h;

  if (get_iconstate(4))
    {
    for(a=0;a<noof_fileflags;a++) menu_setflags(menu_file,a+1,(area.flags&(1<<fileflag[a]))!=0,0);
    return(menu_file);
    }
  for(a=0;a<noof_msgflags;a++) menu_setflags(menu_msg,a+1,(area.flags&(1<<msgflag[a]))!=0,0);
  return(menu_msg);
  }

void do_editmenu(void *h,char *hit)
  {
  h=h;
                            
  if (get_iconstate(4))
    {
    if (hit[1]==0) area.flags^=(1<<fileflag[hit[0]-1]);
    }
  else
    {
    if (hit[1]==0) area.flags^=(1<<msgflag[hit[0]-1]);
    }
  }

void edit_msgproc(wimp_eventstr *event,void *handle)           
  {
  handle=handle;

  switch(event->e)
    {
    case wimp_EOPEN:    /* Open_Window_Request */
      {
      errorlog(wimp_open_wind(&event->data.o),"OpenWind");
      break;
      }
    case wimp_ECLOSE:   /* Close_Window_Request */
      {
      errorlog(wimp_close_wind(area_handle),"CloseWind");
      send_message(servertask,BBS_TASKID,portnumber,1,0,0);
      exit(0);
      break;
      }
    case wimp_EBUT:      /* Mouse click */
      {
      switch(IC)      /* Switch on icon number */
        {
        /* Up/down areanumber */
        case 11:
        case 12:
          {
          char temp[10];
          read_icon(0,temp); sscanf(temp,"%d",&areanumber);
          if (IC==11) areanumber+=1; else areanumber-=1;
          if (areanumber<0) areanumber=0;
          read_area(areanumber,&area);
          open_window();
          break;
          }
        /* Write */
        case 14:
          {
          char temp[10];

          read_area(areanumber,&area2);
          from_icon();
          if (blank)
            {
            blank=0;
            area.first_message=-1;
            area.last_message=-1;
            area.newest=area.count=area.user_id=0;
            area.user[0]=0;
            }
          else
            {
            area.first_message=area2.first_message;
            area.last_message=area2.last_message;
            area.newest=area2.newest;
            area.count=area2.count;
            area.user_id=area2.user_id;
            }
          read_icon(0,temp); sscanf(temp,"%d",&areanumber);
          area.areaflags&=0x7fffffff; /* To get rid of obsolete fileflags */
          write_area(areanumber,&area);
          mod_saveall();
          break;
          }
        /* Blank */
        case 13:
          {
          blank=1;
          break;
          }
        }
      break;
      }
    case wimp_EKEY:
      {
      switch(event->data.key.chcode)
        {
        case 13:
          {
          /* Has user entered areanumber & pressed cr? */
          if (event->data.key.c.i==0) 
            {
            char temp[10];
            read_icon(0,temp); sscanf(temp,"%d",&areanumber);
            if (areanumber<0) areanumber=0;
            read_area(areanumber,&area);
            open_window();
            }
          break;
          }
        default:
          {
          wimp_processkey(event->data.key.chcode);
          break;
          }
        }
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
          exit(0);
          break;
          }
        }
      break;
      }
    }
  }
 
void write_icon(int icon,char *string)
  {
  wimp_icon isblock;
  wimp_icreate iblock;

  /* Get icon info */
  wimp_get_icon_info(area_handle,icon,&isblock);

  if (isblock.flags & 0x100)
    {
    strcpy(isblock.data.indirecttext.buffer,string);
    }
  else
    {
    strcpy(isblock.data.text,string);
    wimp_delete_icon(area_handle,icon);
    iblock.w=area_handle;
    iblock.i=isblock;
    wimp_create_icon(&iblock,&icon);
    }
  wimp_set_icon_state(area_handle,icon,0,0);
  }

void read_icon(int icon,char *string)
  {
  wimp_icon isblock;

  /* Get icon info */
  wimp_get_icon_info(area_handle,icon,&isblock);

  if (isblock.flags & 0x100)
    {
    strcpy(string,isblock.data.indirecttext.buffer);
    }
  else
    {
    strcpy(string,isblock.data.text);
    }
  }

int get_iconstate(int icon)
  {
  wimp_icon isblock;

  /* Get icon info */
  wimp_get_icon_info(area_handle,icon,&isblock);

  if (isblock.flags & 0x00200000) return(0);
  else return(1);
  }

void open_window()
  {
  char temp[10];
  wimp_redrawstr red;
  wimp_caretstr car;

  /* Set window data fom userlog data */
  write_icon(1,area.name);
  sprintf(temp,"%x",area.level);     write_icon(2,temp);
  sprintf(temp,"%d",area.editor_id); write_icon(3,temp);
  sprintf(temp,"%d",areanumber);     write_icon(0,temp);
  set_icon(4,area.areaflags & FLAG_FILEAREA);
  set_icon(5,area.areaflags & FLAG_ECHOMAIL);
  set_icon(6,area.areaflags & FLAG_WRITEABLE);
  set_icon(7,area.areaflags & FLAG_READABLE);
  set_icon(15,area.areaflags & FLAG_AUTOINVISIBLE);
  set_icon(16,area.areaflags & FLAG_FREEDOWNLOAD);
  set_icon(17,area.areaflags & FLAG_FREEUPLOAD);

  red.w=area_handle;
  red.box.x0=0;   red.box.y0=-224;
  red.box.x1=654; red.box.y1=-184;
  /*wimp_force_redraw(&red);*/

  car.w=area_handle;
  car.i=1;
  car.x=car.y=0;
  car.height=-1;
  car.index=0;
  wimp_set_caret_pos(&car);
  }

void open_window2(wimp_w wind,int x,int y)
  {
  wimp_wstate state;

  wimp_get_wind_state(wind,&state);

  /* Open window to full size, ontop of everything else */
  state.o.x=x;
  state.o.y=y;
  state.o.behind=-1;

  errorlog(wimp_open_wind(&state.o),"OpenWindI");
  }

void from_icon()
  {
  char temp[40];

  read_icon(1,area.name);
  read_icon(2,temp); sscanf(temp,"%x",&area.level);
  read_icon(3,temp); sscanf(temp,"%d",&area.editor_id);
  area.areaflags=0;
  if (get_iconstate(4)) area.areaflags|=FLAG_FILEAREA;
  if (get_iconstate(5)) area.areaflags|=FLAG_ECHOMAIL;
  if (get_iconstate(6)) area.areaflags|=FLAG_WRITEABLE;
  if (get_iconstate(7)) area.areaflags|=FLAG_READABLE;
  if (get_iconstate(15)) area.areaflags|=FLAG_AUTOINVISIBLE;
  if (get_iconstate(16)) area.areaflags|=FLAG_FREEDOWNLOAD;
  if (get_iconstate(17)) area.areaflags|=FLAG_FREEUPLOAD;
  }

void set_icon(int icon,int onoff)
  {
  wimp_set_icon_state(area_handle,icon,onoff?0:1<<21,1<<21);
  }

void errorlog(os_error *err,char *part)
  {
  os_error *err2;
  if (err==NULL) return;
  err2=wimp_reporterror(err,3,part);
  err2=wimp_closedown();
  exit(0);
  }

void window_poll() /* Fudge for servcomm.c */
  {
  event_process();
  }
