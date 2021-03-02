/*                  _____________________________________________
                  [>                                             <]
Project           [> Local downloader                            <]
Author            [> Hugo Fiennes                                <]
Date started      [> 27-March-1991                               <]
                  [>                                             <]
Module name       [> Main                                        <]
Current version   [> 01.00                                       <]
Version date      [> 07-April-1991                               <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>    This source is COPYRIGHT (c) 1991 by     <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/
                          
#define VERSION "1.00 (07-April-1991)"

#include "include.h"      /* All the includes */
#include "xfersend.h"

extern int  main(int,char**);
extern void window_init(void),do_iconmenu(void*,char*),nullproc(wimp_i),
            read_icon(wimp_w,wimp_i,char*),write_icon(wimp_w,wimp_i,char*,int),
            set_icon(wimp_w,wimp_i,int),open_swindow(wimp_w,int,int),
            poll_proc(wimp_eventstr*,void*);

menu   iconmenu;
wimp_w enternr;
int filetype,filenumber,filesize;
char dbuffer[10240];

int main(int argc,char *argv[])
  {
  window_init();

  while(1) event_process();
  }

void window_init()
  {
  wimp_wind *window;

  /* Start our task up */
  wimpt_init("LocalDL");          /* Setup task */
  res_init("LocalDL");            /* Resource init */
  resspr_init();
  template_init();
  dbox_init();

  /* Get on iconbar */
  baricon("!localdl",(int)resspr_area(),nullproc);

  /* Make a menu */
  iconmenu=menu_new("LocalDL",">Info,Quit");
  event_attachmenu(win_ICONBAR,iconmenu,do_iconmenu,0);

  /* Make window */
  window=template_syshandle("filenr");
  wimpt_noerr(wimp_create_wind(window,&enternr));
                   
  win_register_event_handler(enternr,poll_proc,0);
  win_activeinc();
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

      dbox_setfield(d,3,VERSION);
      dbox_showstatic(d);
      dbox_fillin(d);
      dbox_dispose(&d);

      break;
      }
    case 2:
    /* Quit */
      {
      exit(0);
      break;
      }
    }
  }

char *file_pathname(int filenumber)
  {
  static char pathname[80];
  sprintf(pathname,"<ARCbbs$filedata>.%06d.%06d.%06d",((filenumber/2500)*2500),
                   ((filenumber/50)*50),filenumber);
  return(pathname);
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

void write_icon(wimp_w window,wimp_i icon,char *string,int refresh)
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
  if (refresh) wimp_set_icon_state(window,icon,0,0);
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

void nullproc(wimp_i a)
  {
  wimp_caretstr c;

  a=a;

  /* Open the window! */
  open_swindow(enternr,0,0);

  c.w=enternr; c.i=c.x=c.y=0; c.height=-1; c.index=0;
  wimp_set_caret_pos(&c);
  }

int getfiletype(char *filename)
  {
  os_filestr file;

  /* Read the type of the file */
  file.action=5;    /* Get catalogue info */
  file.name  =filename;
  if (os_file(&file)!=0) return(-1);

  if ((file.loadaddr & 0xfff00000)!=0xfff00000) return(0);
  return((file.loadaddr>>8)&0xfff);
  }

int getfilesize(char *filename)
  {
  os_filestr file;

  /* Read the size of the file */
  file.action=5;    /* Get catalogue info */
  file.name  =filename;
  if (os_file(&file)!=0) return(0);

  if (file.action==0 || file.action==2) return(0);
  return(file.start);
  }

BOOL saveit(char *leaf,void *h)
  {
  FILE *in,*out;
  int blocksize;

  h=h;

  if ((in=fopen(file_pathname(filenumber),"rb"))==NULL) return(FALSE);

  sprintf(dbuffer,"Create %s %x",leaf,filesize);
  system(dbuffer);
  sprintf(dbuffer,"Settype %s %x",leaf,filetype);
  system(dbuffer);

  if ((out=fopen(leaf,"rb+"))==NULL)
    {
    fclose(in); remove(leaf); return(FALSE);
    }

  do
    {             
    blocksize=(filesize>10240)?10240:filesize;
    fread(dbuffer,blocksize,1,in);
    fwrite(dbuffer,blocksize,1,out);
    filesize-=blocksize;
    }
  while(filesize>0);

  fclose(in); fclose(out);
  return(TRUE);
  }

void poll_proc(wimp_eventstr *e,void *handle)
  {
  handle=handle;

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
      break;
      }
    case wimp_EKEY:
      {
      if (e->data.key.chcode==13)
        {
        char buffer[128];
        char *path;

        /* User has hit cr in window on filenumber */
        read_icon(enternr,0,buffer);
        if (sscanf(buffer,"%d",&filenumber)!=1) break;
        sprintf(buffer,"%06d",filenumber);
        
        path=file_pathname(filenumber);

        if ((filetype=getfiletype(path))>=0)
          {
          if ((filesize=getfilesize(path))>0)
            {
            saveas(filetype,buffer,filesize,saveit,0,0,0);
            }
          }
        }
      else
        {
        wimp_processkey(e->data.key.chcode);
        }

      break;
      }
    case 17:            /* Incoming message */
    case 18:
      {
      switch(e->data.msg.hdr.action)
        {
        case 0:              /* Quit */
          {
          exit(0); break;
          }
        }
      break;
      }
    }
  }
