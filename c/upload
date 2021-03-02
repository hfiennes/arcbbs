/*                  _____________________________________________
                  [>                                             <]
Project           [> PortNet communications network              <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Local upload module                         <]
Current version   [> 00.22                                       <]
Version date      [> 04-November-1992                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"
#include "xferrecv.h"

/* Other headers */
#include "userlog.h"
#include "mail.h"
#include "servcomm.h"
#include "servmess.h"

#define IC event->data.but.m.i

/* PreDeclarations */
int  main(int,char**),get_iconstate(int),filesize(char*);
void errorlog(os_error*,char*),window_init(void),load_file(void),
     write_icon(int,char*,int),read_icon(int,char*),open_window(void),
     from_icon(void),set_icon(int,int),open_window2(wimp_w,int,int),
     edit_msgproc(wimp_eventstr*,void*),window_poll(void),file_upload(void);
char *file_pathname(int);

/* Global variables */
int     got_data,got_code,areanumber=-1,blank=0,portnumber,size,filetype;
wimp_t  servertask; wimp_w  area_handle;
mail_block fileheader; mail_area area;
char    buffer[320],bigname[161],name[21],*message_buffer,*_message_buffer;

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
  int maxarea;

  _message_buffer=buffer;
  message_buffer=(_message_buffer+9);

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

  maxarea=areamax();
  do
    {
    read_area(++areanumber,&area);
    }
  while(areanumber<maxarea && (area.areaflags&FLAG_FILEAREA)==0);
  if (areanumber>=maxarea) read_area(areanumber=0,&area);
  write_icon(9,area.name,1);

  name[0]=bigname[0]=0;

  /* Open window */
  open_window(); open_window2(area_handle,0,0);

  while(1) event_process();

  return(0);
  }

void window_init()
  {
  wimp_wind *window;

  /* Initialise WIMP library modules */
  wimpt_init("LOCALupload");      /* Setup task */
  res_init("LOCupload");          /* Resource init */
  template_init();

  window=template_syshandle("upload");
  wimp_create_wind(window,&area_handle);
  win_activeinc();

  /* Attach event handler */
  win_register_event_handler(area_handle,edit_msgproc,0);
  win_claim_unknown_events(area_handle);
  win_claim_idle_events(area_handle);
  event_setmask(0);
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
        case 7:
          {
          int oldareanumber=areanumber;

          do
            {
            read_area(--areanumber,&area);
            }
          while(areanumber>=0 && (area.areaflags&FLAG_FILEAREA)==0);

          if (areanumber<0) read_area(areanumber=oldareanumber,&area);
          else write_icon(9,area.name,1);

          break;
          }
        case 8:
          {
          int oldareanumber=areanumber,maxarea=areamax();

          do
            {
            read_area(++areanumber,&area);
            }
          while(areanumber<maxarea && (area.areaflags&FLAG_FILEAREA)==0);

          if (areanumber>=maxarea) read_area(areanumber=oldareanumber,&area);
          else write_icon(9,area.name,1);
          break;
          }
        /* Write */
        case 15:
          {
          file_upload();    

          /* Blank out filename, etc to show it's been uploaded */
          write_icon(17,"",1);
          write_icon(18,"",1);
          break;
          }
        }
      break;
      }
    case wimp_EKEY:
      {
      switch(event->data.key.chcode)
        {
        case 13:      /* CR */
        case 0x18e:   /* Cursor down */
          {
          wimp_caretstr car;

          if (event->data.key.c.i>=10 && event->data.key.c.i<14)
            {
            car.w=area_handle;
            car.i=event->data.key.c.i+1;
            car.x=car.y=0;
            car.height=-1;
            car.index=0;
            wimp_set_caret_pos(&car);
            }

          break;
          }
        case 0x18f:   /* Cursor up */
          {
          wimp_caretstr car;

          if (event->data.key.c.i>=11 && event->data.key.c.i<15)
            {
            car.w=area_handle;
            car.i=event->data.key.c.i-1;
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
        case wimp_MDATALOAD:   /* insert data */
          {
          load_file();
          break;
          }
        default:
          {
          if (event->data.msg.hdr.action>BBS_CHUNK && event->data.msg.hdr.action<(BBS_CHUNK+64))
            {  
            got_code=event->data.msg.data.words[0];
            got_data=1;
            }
          break;
          }
        }
      break;
      }
    }
  }
 
void write_icon(int icon,char *string,int refresh)
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
  if (refresh) wimp_set_icon_state(area_handle,icon,0,0);
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
  wimp_redrawstr red; wimp_caretstr car;

  write_icon(10,"",0);
  write_icon(11,"",0);
  write_icon(12,"",0);
  write_icon(13,"",0);
  write_icon(14,"",0);
  write_icon(16,"",0);
  write_icon(17,"",0);
  write_icon(18,"",0);

  red.w=area_handle;
  red.box.x0=0;    red.box.y0=-324;
  red.box.x1=1234; red.box.y1=0;
  wimp_force_redraw(&red);

  car.w=area_handle;
  car.i=6;
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

void set_icon(int icon,int onoff)
  {
  wimp_icreate temp;
  wimp_i trash;

  /* Get icon info */
  wimp_get_icon_info(area_handle,icon,&temp.i);
  temp.w=area_handle;

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
  wimp_delete_icon(area_handle,icon);
  wimp_create_icon(&temp,&trash);
  /* wimp_set_icon_state(area_handle,icon,onoff?0:1<<21,1<<21); */
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

void load_file()
  {
  char *filename;

  /* Fetch the type and name of the file */
  filetype=xferrecv_checkinsert(&filename);

  /* Check the name and make a permanent copy */
  if (strlen(filename)>160)
    {
    werr(0, "File name is too long");
    }
  else
    {
    strcpy(bigname,filename);

    /* Get size of file (and check it is a file) */
    size=filesize(bigname);

    /* Check file type */
    if (size>0)
      {
      os_regset in; char temp[20];

      char *ptr=bigname+strlen(bigname)-1;
      while(ptr>=bigname && *ptr!='.' && *ptr!=':') ptr--;
      if (*ptr=='.' || *ptr==':') ptr++;
      strcpy(name,ptr);

      in.r[0]=18; in.r[2]=filetype;
      os_swi(0x29,&in); /* OS_FSControl */
      in.r[4]=0; /* Check zero-terminated */

      sprintf(temp,"&%03x (%s)",filetype,(char*)&in.r[2]);
                                     write_icon(17,temp,1);
      sprintf(temp,"%d",size);       write_icon(18,temp,1);
      write_icon(16,name,1);
      }
    }

  /* Indicate load is completed */
  xferrecv_insertfileok();
  }

int filesize(char *filename)
  {
  os_filestr file;

  /* Read the size of the file */
  file.action=5;    /* Get catalogue info */
  file.name  =filename;
  if (wimpt_complain(os_file(&file))!=0) return(0);

  switch (file.action)
    {
    case 0: werr(0,"File not found");              return(0);
    case 2: werr(0,"You cannot load a directory"); return(0);
    }

  if ((file.loadaddr & 0xFFF00000)==0xFFF00000)
    {
    return(file.start);
    }
  else
    {
    werr(0,"File cannot be loaded");
    return(0);
    }
  }

void file_upload()
  {
  mail_block header; user_block uploader;
  int flags=0,thisblock;
  char *ptr=buffer,temp[80],fbuffer[10240];
  FILE *inf,*outf;
  os_filestr attr; int attr_load,attr_exec;
                  
  read_icon(20,temp); uploader.usernumber=atoi(temp); get_user(&uploader);
  memset(&header,0,255);

  header.reply_from=-1;
  header.message_area=areanumber;
  header.from_id=uploader.usernumber;
  strcpy(header.from,uploader.username);
  read_icon(16,header.to);
  header.to_id=0; /* Number of downloads */
  header.file_length=size;
  header.file_location=get_filenumber();

  /* Set date sent + expire 1 year afterwards */
  header.date_entered=header.date_sent=time(NULL);

  read_icon(10,header.subject);

  buffer[0]=0;

  read_icon(11,temp); if (*temp) { strcpy(ptr,temp); ptr+=strlen(temp)+1; }
  read_icon(12,temp); if (*temp) { strcpy(ptr,temp); ptr+=strlen(temp)+1; }
  read_icon(13,temp); if (*temp) { strcpy(ptr,temp); ptr+=strlen(temp)+1; }
  read_icon(14,temp); if (*temp) { strcpy(ptr,temp); ptr+=strlen(temp)+1; }

  header.message_length=(ptr-buffer);

  if (filetype==0xddc) flags|=FILE_ARC;
  if (filetype!=0xfff) flags|=FILE_BINARY;

  /* Set binary or whatever */
  header.flags=flags;

  attr.action=17;
  attr.name=bigname;
  os_file(&attr);
  attr_load=attr.loadaddr;
  attr_exec=attr.execaddr;

  /* Copy upload file to filepath file */
  if ((inf=fopen(bigname,"rb"))==NULL)
    {
    werr(0,"Can't open source file");
    return;
    }

  if ((outf=fopen(file_pathname(header.file_location),"wb"))==NULL)
    {
    werr(0,"Can't open destination file");
    fclose(inf);
    return;
    }

  /* Copy file, poll every 10k */
  do
    {
    thisblock=fread(fbuffer,1,10240,inf);
    fwrite(fbuffer,1,thisblock,outf);
    window_poll();
    }
  while(thisblock==10240);
  fclose(inf); fclose(outf);

  attr.action=2;
  attr.name=file_pathname(header.file_location);
  attr.loadaddr=attr_load;
  os_file(&attr);

  attr.action=3;
  attr.name=file_pathname(header.file_location);
  attr.execaddr=attr_exec;
  os_file(&attr);

  if (put_message(&header,buffer)!=OK)
    {
    werr(0,"Could not upload file");
    return;
    }

  send_message(servertask,BBS_FILECOUNTS,0,1,0,0);
  
  sprintf(fbuffer,"#%06d",header.file_location);
  write_icon(16,fbuffer,1);
  }

char *file_pathname(int filenumber)
  {
  static char pathname[80];
  sprintf(pathname,"<ARCbbs$filedata>.%06d.%06d.%06d",((filenumber/2500)*2500),
                   ((filenumber/50)*50),filenumber);
  return(pathname);
  }
