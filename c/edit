/*                  _____________________________________________
                  [>                                             <]
Project           [> PortNet communications network              <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Edit module                                 <]
Current version   [> 00.48                                       <]
Version date      [> 19-November-1992                            <]
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
#include "servmess.h"  
#include "servcomm.h"
#include "modcomm.h"
#include "crc.h"
#include "msgs.h"

#define IC event->data.but.m.i

/* Maximum number of users in userlog/portnumber fix (unused) */
int maxusers,portnumber;
user_list *online_users;

/* PreDeclarations */
int  main(int,char**),get_user2(void),put_user2(void),get_iconstate(int);
void errorlog(os_error*,char*),window_init(void),do_editmenu(void*,char*),
     write_icon(int,char*),read_icon(int,char*),open_window(void),
     from_icon(void),set_icon(int,int),open_window2(wimp_w,int,int),
     edit_msgproc(wimp_eventstr*,void*),do_editmenu(void*,char*);
static menu do_makeedit(void*);

/* Global variables */
int     tasknumber,got_data,got_code,oldstatus,edit_menutype=-1,items;
wimp_t  servertask=0;
wimp_w  edit_handle;
char    usern[31],findbuffer[32],rembuffer[32];
menu    edit_menu,confirmnew,find_user,rem_user,add_user;
user_block ub;

char msgmenu[1024],filemenu[1024],usermenu[1024];
char msgflag[32],fileflag[32],userflag[32],
     *_message_buffer,*message_buffer;
menu menu_msg,menu_file,menu_user;  
int  noof_msgflags,noof_fileflags,noof_userflags;

int main(int argc,char *argv[])
  {
  if (argc!=3)
    {
    return(0);
    } 

  /* Read user number from environment string */
  ub.usernumber=atoi(argv[1]);

  /* Read task number from environment string */
  tasknumber=atoi(argv[2]);

  maxusers=mod_maxuser();
  online_users=(void*)mod_getstatuspointer();           

  /* Initialise wimp */
  window_init();
              
  /* Wait for server id */
  while(servertask==0) event_process();

  /* Get userlog data for requested user */
  get_user2();

  /* Open window */
  open_window(); open_window2(edit_handle,0,0);

  while(1) event_process();

  return(0);
  }
             
void window_poll()
  {
  event_process();
  }

void window_init()
  {
  static char buffer[20];
  wimp_wind *window;
  FILE *f; char line[80],*p; int a,b;

  /* Start our task up */
  sprintf(buffer,"ARCedit_%d",tasknumber);

  /* Initialise WIMP library modules */
  wimpt_init(buffer);             /* Setup task */
  res_init("ARCedit");            /* Resource init */
  template_init();                       
  dbox_init();

  window=template_syshandle("edit");
  wimp_create_wind(window,&edit_handle);
  win_activeinc();

  /* Attach event handler */
  win_register_event_handler(edit_handle,edit_msgproc,0);
  win_claim_unknown_events(edit_handle);
  event_setmask(wimp_EMPTRENTER|wimp_EMPTRLEAVE|wimp_EMNULL);

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

  if ((f=fopen("<ARCbbs$MiscData>.Flags.User","r"))==NULL) exit(0);
  p=usermenu; b=0;
  do
    {
    if (!fgets(line,79,f)) break;
    if (sscanf(line,"%d %s\n",&a,p)==2)
      {
      userflag[b++]=a; p+=strlen(p);
      *p++=',';
      }
    }
  while(!feof(f));
  fclose(f); *--p=0; noof_userflags=b;
  menu_user=menu_new("User flags",usermenu);

  /* Attach menu maker */
  event_attachmenumaker(edit_handle,do_makeedit,do_editmenu,(void*)0);
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
      errorlog(wimp_close_wind(edit_handle),"CloseWind");
      exit(0);
      break;
      }
    case wimp_EBUT:      /* Mouse click */
      {
      switch(IC)      /* Switch on icon number */
        {
        /* Password calculate */
        case 10:
          {
          char temp[256];
          read_icon(9,temp); ub.passcrc=calcrc(temp,strlen(temp));
          write_icon(9,"");
          break;
          }
        /* Up/down usernumber */
        case 11:
        case 13:
          {
          char temp[10];
          read_icon(12,temp); sscanf(temp,"%d",&ub.usernumber);
          ub.usernumber+=(12-IC);
          if (ub.usernumber>=maxusers || ub.usernumber<0)
            ub.usernumber-=(12-IC);
          get_user2();
          open_window();
          break;
          }
        /* Read user */
        case 20:
          {
          char temp[10];
          read_icon(74,temp); sscanf(temp,"%d",&ub.usernumber);
          if (ub.usernumber>(maxusers-1))  ub.usernumber=maxusers-1;
          if (ub.usernumber<0)             ub.usernumber=0;
          get_user2();
          open_window();
          break;
          }
        /* Reset time today */
        case 37:
          {
          ub.timetoday=0;
          break;
          }
        /* Save user */
        case 25:
          {
          int a;

          /* Write userlog record */
          from_icon();
          put_user2();

          /* Check to see if user is online */
          for(a=0;a<16;a++)
            {
            if (online_users[a].username[0]!=0)
              {
              if (online_users[a].usernumber==ub.usernumber)
                {
                send_message(servertask,BBS_READUSER,a,0,0,0);
                }
              }
            }
          break;
          }
        }
      break;
      }
    case wimp_EKEY:
      {
      wimp_caretstr car;

      switch(event->data.key.chcode)
        {
        case 13:
          {
          /* Has user entered usernumber & pressed cr? */
          if (event->data.key.c.i==12) 
            {
            char temp[10];
            read_icon(12,temp); sscanf(temp,"%d",&ub.usernumber);
            if (ub.usernumber>(maxusers-1))  ub.usernumber=maxusers-1;
            if (ub.usernumber<0)             ub.usernumber=0;
            get_user2();
            open_window();
            }
          else
            {
            if (event->data.key.c.i>=0 && event->data.key.c.i<=7)
              {
              /* Shift caret down one */
              car.w=edit_handle;
              car.i=event->data.c.i+1;
              car.x=car.y=0;
              car.height=-1;
              car.index=0;
              wimp_set_caret_pos(&car);
              }
            }
          break;
          }
        case 0x18e:   /* Cursor down */
          {
          if (event->data.key.c.i>=0 && event->data.key.c.i<=7)
            {
            /* Shift caret down one */
            car.w=edit_handle;
            car.i=event->data.c.i+1;
            car.x=car.y=0;
            car.height=-1;
            car.index=0;
            wimp_set_caret_pos(&car);
            }
          break;
          }
        case 0x18f:   /* Cursor up */
          {
          if (event->data.key.c.i>=1 && event->data.key.c.i<=8)
            {
            /* Shift caret up one */
            car.w=edit_handle;
            car.i=event->data.c.i-1;
            car.x=car.y=0;
            car.height=-1;
            car.index=0;
            wimp_set_caret_pos(&car);
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
        case BBS_SERVERIS:
          {
          servertask=event->data.msg.hdr.task;
          break;
          }
        }
      break;
      }
    }
  }

int get_user2()
  {
  int retcode;

  retcode=get_user(&ub);

  /* Copy username into temp buffer */
  strcpy(usern,ub.username);
  oldstatus=ub.status;       

  /* Return result code */
  return(retcode);
  }

int put_user2()
  {
  put_user(&ub);

  if (ub.status!=0 && oldstatus!=0)
    {                             
    /* Are old/new usernames different? */
    if (strcmp(usern,ub.username)!=0)
      {
      /* Yes they are different. Was old username null? If not, delete hash
         entry */
      if (usern[0]) hash_kill(usern);

      /* Is new entry not null? If so, put it in hash table */
      if (ub.username[0]) hash_write(ub.username,ub.usernumber);
      }
    }

  if (ub.status!=0 && oldstatus==0)
    {
    /* Add hash entry */
    if (ub.username[0]) hash_write(ub.username,ub.usernumber);
    }

  if (ub.status==0 && oldstatus!=0)
    {
    /* User is no longer active, kill any hash reference */
    if (usern[0]) hash_kill(usern);
    }

  /* Copy username into temp buffer */
  strcpy(usern,ub.username);
  oldstatus=ub.status;                    

  /* Return result code */
  return(got_code);
  }
 
void write_icon(int icon,char *string)
  {
  wimp_icon isblock;
  wimp_icreate iblock;

  /* Get icon info */
  wimp_get_icon_info(edit_handle,icon,&isblock);

  if (isblock.flags & 0x100)
    {
    strcpy(isblock.data.indirecttext.buffer,string);
    }
  else
    {
    strcpy(isblock.data.text,string);
    wimp_delete_icon(edit_handle,icon);
    iblock.w=edit_handle;
    iblock.i=isblock;
    wimp_create_icon(&iblock,&icon);
    }
 /* wimp_set_icon_state(edit_handle,icon,0,0); */
  }

void read_icon(int icon,char *string)
  {
  wimp_icon isblock;

  /* Get icon info */
  wimp_get_icon_info(edit_handle,icon,&isblock);

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
  wimp_get_icon_info(edit_handle,icon,&isblock);

  if (isblock.flags & 0x00200000)
    {
    return(0);
    }
  else
    {
    return(1);
    }
  }

void open_window()
  {
  char temp[10];
  wimp_redrawstr red;

  /* Set window data fom userlog data */
  write_icon(0,ub.username);
  write_icon(1,ub.realname);
  write_icon(2,ub.address[0]);
  write_icon(3,ub.address[1]);
  write_icon(4,ub.address[2]);
  write_icon(5,ub.address[3]);
  write_icon(6,ub.postcode);
  write_icon(8,ub.telephone);
  write_icon(9,""); /* Blank password */
  set_icon(22,ub.status);
  sprintf(temp,"%d",ub.padsize);     write_icon(19,temp);
  sprintf(temp,"%d",ub.ratio);       write_icon(24,temp);
  sprintf(temp,"%d",ub.usernumber);  write_icon(12,temp);
  sprintf(temp,"%x",ub.userlevel);   write_icon(18,temp);
  sprintf(temp,"%d",ub.timeallowed); write_icon(14,temp);
  sprintf(temp,"%d",ub.uploads);     write_icon(23,temp);
  sprintf(temp,"%d",ub.downloads);   write_icon(21,temp);
  temp[0]=ub.callrate; temp[1]=0;    write_icon(7,temp);
  red.w=edit_handle;
  red.box.x0=162; red.box.y0=-644;
  red.box.x1=652; red.box.y1=-10;
  wimp_force_redraw(&red);
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

  read_icon(0,ub.username);
  read_icon(1,ub.realname);
  read_icon(2,ub.address[0]);
  read_icon(3,ub.address[1]);
  read_icon(4,ub.address[2]);
  read_icon(5,ub.address[3]);
  read_icon(6,ub.postcode);
  read_icon(8,ub.telephone);
  read_icon(19,temp); sscanf(temp,"%d",&ub.padsize);
  read_icon(24,temp); sscanf(temp,"%d",&ub.ratio);
  read_icon(18,temp); sscanf(temp,"%x",&ub.userlevel);
  read_icon(14,temp); sscanf(temp,"%d",&ub.timeallowed);
  read_icon(23,temp); sscanf(temp,"%d",&ub.uploads);
  read_icon(21,temp); sscanf(temp,"%d",&ub.downloads);
  read_icon(7,temp); ub.callrate=temp[0];
  ub.status=get_iconstate(22)?1:0;
  }

void set_icon(int icon,int onoff)
  {
  wimp_icreate temp;
  wimp_i trash;

  /* Get icon info */
  wimp_get_icon_info(edit_handle,icon,&temp.i);
  temp.w=edit_handle;

  /* Reset its flags */
  if (onoff==0)
    {
    temp.i.flags|=0x00200000;
    }
  else
    {
    temp.i.flags&=0xffdfffff;
    }

  /* Re-create the icon */
  wimp_delete_icon(edit_handle,icon);
  wimp_create_icon(&temp,&trash);
  /* wimp_set_icon_state(edit_handle,icon,onoff?0:1<<21,1<<21); */
  }

void errorlog(os_error *err,char *part)
  {
  os_error *err2;
  if (err==NULL) return;
  err2=wimp_reporterror(err,3,part);
  err2=wimp_closedown();
  exit(0);
  }

menu do_makeedit(void *handle)
  {
  wimp_mousestr where;
  menu edit_menu1,edit_menu2,edit_menu3,edit_menu4;
  int a;

  handle=handle;
 
  if (event_is_menu_being_recreated()==FALSE)
    {
    edit_menutype=items=0;
    wimp_get_point_info(&where);

    if (where.i==15) edit_menutype=1;
    if (where.i==16) edit_menutype=2;
    if (where.i==17) edit_menutype=3;

    switch(edit_menutype)
      {
      case 0:
        {
        /* Usual menu */
        edit_menu=menu_new("BBS Useredit",">Info,Reset mail flags,Find user,Add lookup,Remove lookup");
        edit_menu1=menu_new("","Sure?");    
        menu_submenu(edit_menu,2,edit_menu1);
        edit_menu2=menu_new("Username","enterhere");
        menu_make_writeable(edit_menu2,1,findbuffer,31,0);
        menu_submenu(edit_menu,3,edit_menu2);
        findbuffer[0]=0;
        edit_menu3=menu_new("Username","enterhere");
        menu_make_writeable(edit_menu3,1,rembuffer,31,0);
        menu_submenu(edit_menu,4,edit_menu3);
        edit_menu4=menu_new("Username","enterhere");
        menu_make_writeable(edit_menu4,1,rembuffer,31,0);
        menu_submenu(edit_menu,5,edit_menu4);
        rembuffer[0]=0; 
        break;
        }
      case 1:
        {               
        /* User flags */  
        edit_menu=menu_user;
        items=noof_userflags;
        break;
        }
      case 2:
        {               
        /* Message flags */  
        edit_menu=menu_msg;
        items=noof_msgflags;
        break;
        }
      case 3:
        {               
        /* File flags */  
        edit_menu=menu_file;
        items=noof_fileflags;
        break;
        }
      }
    }

  if (items)
    {
    switch(edit_menutype)
      {
      case 1: /* User */
        {
        for(a=0;a<items;a++) menu_setflags(edit_menu,a+1,
                                           (ub.f_user&(1<<userflag[a]))!=0,0);
        break;
        }
      case 2: /* Message */
        {
        for(a=0;a<items;a++) menu_setflags(edit_menu,a+1,
                                           (ub.f_message&(1<<msgflag[a]))!=0,0);
        break;
        }
      case 3: /* File */
        {
        for(a=0;a<items;a++) menu_setflags(edit_menu,a+1,
                                           (ub.f_file&(1<<fileflag[a]))!=0,0);
        break;
        }
      }                 
    }
  return(edit_menu);  
  }

void do_editmenu(void *handle,char *hit)
  {
  handle=handle;

  switch(edit_menutype)
    {
    case 0: /* Main menu */
      {
      switch(hit[0])
        {
        case 1:
        /* Info */
          {
          dbox d=dbox_new("info");

          dbox_showstatic(d);
          dbox_fillin(d);
          dbox_dispose(&d);

          break;
          }
        case 2:
        /* Reset mail flags */
          {
          switch(hit[1])
            {
            case 1:
              {
              /* Blank out info */
              ub.t_firstlogon=ub.t_lastlogon=time(NULL);
              ub.m_message=ub.m_file=0;
      
              /* Setup other system flags */
              ub.mailpointer_start=ub.mailpointer_end=ub.morepointer=-1;
              ub.mailcount=0; ub.logons=0;
              break;
              }
            }
          break;
          }
        case 3:
        /* Find a user */
          {
          switch(hit[1])
            {
            case 1:
              {
              int result=hash_find(findbuffer);
    
              if (result!=NODATA)
                {
                ub.usernumber=result;
                get_user2();
                open_window();
                }
              break;
              }
            }
          break;
          }
        case 4:
        /* Add a username */
          {
          switch(hit[1])
            {
            case 1:
              {
              hash_write(rembuffer,ub.usernumber);
              break;
              }
            }
          break;
          }
        case 5:
        /* Remove a username */
          {
          switch(hit[1])
            {
            case 1:
              {
              hash_kill(rembuffer);
              break;
              }
            }
          break;
          }
        }
      break;
      }
    case 1: /* User */
      {
      ub.f_user^=(1<<userflag[hit[0]-1]);
      break;
      }
    case 2: /* Messages */
      {     
      ub.f_message^=(1<<msgflag[hit[0]-1]);
      break;
      }
    case 3: /* File */
      {     
      ub.f_file^=(1<<fileflag[hit[0]-1]);
      break;
      }
    }
  }

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
