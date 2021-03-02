/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 17-August-1990                              <]
                  [>                                             <]
Module name       [> Scrolling terminal code                     <]
Current version   [> 00.25                                       <]
Version date      [> 05-April-1993                               <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1990-1993 by    <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"
#include "scr.h"
#include "port.h"
#include "colourtran.h"
#include "alarm.h"

extern wimp_w local_handle;
extern void   open_swindow(wimp_w,int,int),
              scr_sendesc(char*),close_spool(void);
extern int    inkey(int),getmsgnr(void);
extern BOOL   message_save(char*,void*);

static clock_t lastredraw,lastflash,lastcflash;
static int     scr_careton=1,scr_carethere=0,*ansiset,cursorphase,is16;
static sprite_pixtrans trans[256];
        
int scr_spoolfile=0,scr_spoolstrip=1;
int scr_available=0;
               
struct_scr scr_screen;
static int reqcol[]={ 0x00000000,0x0000BB00,0x00BB0000,0x00BBBB00,
                      0xAA000000,0xBB00BB00,0xBBBB0000,0xBBBBBB00,
                      0x00000000,0x0000FF00,0x00FF0000,0x00FFFF00,
                      0xFF000000,0xFF00FF00,0xFFFF0000,0xFFFFFF00 };
extern int  scr_colours[16];

void settitle(wimp_w w, char *newtitle)
  {
  wimp_redrawstr r;
  wimp_winfo *settitle_winfo=malloc(sizeof(wimp_wind) + 30*sizeof(wimp_icon));

  /* --- get the window's details --- */
  settitle_winfo->w = w;
  wimpt_noerr(wimp_get_wind_info(settitle_winfo));

  /* --- put the new title string in the title icon's buffer --- */
  strcpy(settitle_winfo->info.title.indirecttext.buffer, newtitle);

  free(settitle_winfo);

  /* --- invalidate the title bar in absolute coords --- */
  r.w = (wimp_w) -1;    /* absolute screen coords */
  r.box = settitle_winfo->info.box;
  r.box.y1 += 36;            /* yuk - tweaky */
  r.box.y0 = r.box.y1 - 36;
  wimpt_noerr(wimp_force_redraw(&r));
  }

void scr_tofileproblem(os_error *err)
  {
  err=err;
  }

void scr_init()
  {
  int a,*b;
  FILE *tf;         

  atexit(close_spool);

  if ((scr_screen.spritearea=malloc(0x10ebc))==NULL)
    {
    werr(1,"Can't get memory for sprite");
    }

  scr_screen.width=80;
        
  sprite_area_initialise(scr_screen.spritearea,(864*80)+0xbc);
  sprite_create_rp(scr_screen.spritearea,"ScrollSprite",sprite_haspalette,80<<3,216,12,(sprite_ptr*)&scr_screen.spriteptr);

  if ((scr_screen.wordmap=(int*)malloc(7680))==NULL)
    {
    werr(1,"Can't allocate memory for wordmap (scr)");
    }

  if ((scr_screen.charset=(int*)malloc(17280))==NULL)
    {
    werr(1,"Can't allocate memory for charset (scr)");
    } 

  if ((tf=fopen("<ARCbbs$Dir>.ANSIset","rb"))==NULL)
    {
    werr(1,"Can't open characterset file (scr)");
    }
  fread(scr_screen.charset,17280,1,tf);
  fclose(tf);
  ansiset=scr_screen.charset;

  scr_screen.currentattrib=0x70000000;
  scr_screen.x=scr_screen.y=0;                   
                       
  /* Setup palette */
  b=scr_screen.spriteptr+11;
  for(a=0;a<16;a++) b[a*2]=b[(a*2)+1]=reqcol[a];
  scr_colourmap();

  scr_screen.sprite=scr_screen.spriteptr+(scr_screen.spriteptr[8]/4);
  scr_screen.wordmaptop=scr_screen.wordmap+1920;
  scr_screen.spritetop=scr_screen.sprite+17280;

  scr_screen.cursortype=1; /* 0 for block */
  scr_screen.cursorstatus=0;
     
  scr_process(-1);          
  scr_process(12); /* Clear screen */

  scr_refresh();        
  scr_setcaret(-1);
  scrcursor_setflash(1);

  scr_available=1;
  }
                      
void scr_start()
  {
  scr_screen.minx=scr_screen.width; scr_screen.maxx=-1;
  scr_screen.miny=24;               scr_screen.maxy=-1;
  }

void scr_end()
  {
  wimp_redrawstr r;
  BOOL out;

  if (scr_screen.maxx==-1) return;

  r.w=local_handle;
  r.box.x0=scr_screen.minx<<4;
  r.box.x1=(scr_screen.maxx+1)<<4;
  r.box.y0=-(scr_screen.maxy+1)*36;
  r.box.y1=-scr_screen.miny*36;
  wimp_update_wind(&r,&out);
  scr_redraw(&r,&out);
  }

void scr_refresh()
  {
  wimp_redrawstr r;
  BOOL out;

  r.w=local_handle;
  r.box.x0=0;
  r.box.x1=scr_screen.width<<4;
  r.box.y0=-24*36;
  r.box.y1=0;
  wimp_update_wind(&r,&out);
  scr_redraw(&r,&out);
  scr_start();
  }

void scr_direct2(int b,int a)
  {
  scr_direct(b,a%scr_screen.width,a/scr_screen.width);
  }

void scr_flash(int calledat,void *handle)
  {
  static int phase;

  handle=handle;

  if (calledat==-1)
    {
    scr_flashprocess(phase=1);
    scr_end(); scr_start();
    return;
    }

  if ((phase=1-phase)==0) scr_screen.currentattrib|=FLASHHIDE;
  else scr_screen.currentattrib&=~FLASHHIDE;
          
  if (scr_flashprocess(phase))
    {
    scr_end(); scr_start();
    }

  alarm_set(lastflash=(alarm_timenow()+50),scr_flash,0);
  }

void scrcursor_flash(int calledat,void *handle)
  {
  wimp_redrawstr r;
  BOOL out;
  int a=scr_screen.wordmap[scr_screen.y*scr_screen.width]&DUB;

  handle=handle;

  if (calledat==-1) scr_docursor(cursorphase=0);
  else
    {
    if (scr_careton) scr_docursor((cursorphase=1-cursorphase));
    alarm_set(lastcflash=(alarm_timenow()+CURSORTIME),scrcursor_flash,0);
    }

  /* Redraw the cursor space */
  r.w=local_handle;
  r.box.x0=scr_screen.x<<(a?5:4);
  r.box.x1=r.box.x0+(a?32:16);
  r.box.y1=-scr_screen.y*36;
  r.box.y0=r.box.y1-36;
  wimp_update_wind(&r,&out);
  scr_redraw(&r,&out);
  }

void scr_toscrollback(int pos)
  {
  int x,y; char line[134],*t;

  if (1) return;

  if (pos==-1)
    {
    for(y=0;y<24;y++)
      {
      line[0]=10;
      for(x=0;x<scr_screen.width;x++)
        {
        if ((line[x+1]=((scr_screen.wordmap[x+(scr_screen.width*y)]>>8)&0xff))<32)
          line[x+1]=32;
        }
      t=line+scr_screen.width-1;
      while(*t==' ' && t>=line) t--;
      *++t=0;
      /*scrollback_prints(line);*/
      }
    }
  else
    {
    line[0]=10;
    for(x=0;x<scr_screen.width;x++)
      {
      if ((line[x+1]=((scr_screen.wordmap[x+(scr_screen.width*pos)]>>8)&0xff))<32)
        line[x+1]=32;
      }
    t=line+scr_screen.width-1;
    while(*t==' ' && t>=line) t--;
    *++t=0;
    /*scrollback_prints(line);*/
    }
  }

void scr_setflash(int a)
  {
  if (a)
    {
    scr_flash(-1,0);
    if (lastflash) alarm_remove(lastflash,0);
    alarm_set(lastflash=(alarm_timenow()+50),scr_flash,0);
    }
  else
    {
    scr_flash(-1,0);
    if (lastflash) alarm_remove(lastflash,0);
    }
  }

void scrcursor_setflash(int a)
  {
  if (a)
    {
    scrcursor_flash(-1,0);
    if (lastcflash) alarm_remove(lastcflash,0);
    alarm_set(lastcflash=(alarm_timenow()+CURSORTIME),scrcursor_flash,0);
    }
  else
    {
    scrcursor_flash(-1,0);
    if (lastcflash) alarm_remove(lastcflash,0);
    }
  }

BOOL scr_screensave(char *fname,void *handle)
  {
  FILE *sel;
  char line[136],*t;
  int x,y,top=0,bottom=23,h=(int)handle;
                    
  if (h!=NULL)
    {
    top=h&0xff;
    bottom=h>>8;
    }

  if ((sel=fopen(fname,"a"))==NULL)
    {
    /*werr(0,"Can't save screen",fname);*/
    return(FALSE);
    }

  for(y=top;y<=bottom;y++)
    {
    for(x=0;x<scr_screen.width;x++)
      {
      line[x]=((scr_screen.wordmap[x+(scr_screen.width*y)]>>8)&0xff);
      }
    t=line+scr_screen.width-1;
    while(*t==' ' && t>=line) t--;
    *++t=0;
    fprintf(sel,"%s\012",line);
    }

  fclose(sel);
  return(TRUE);
  }

BOOL scr_spritesave(char *fname,void *handle)
  {
  os_error *b; int a;

  handle=handle;
         
  scr_caret(0);
  for(a=0;a<16;a++) scr_colours[a]=a+(a<<4)+(a<<8)+(a<<12)+(a<<16)+(a<<20)+(a<<24)+(a<<28);
  if (scr_screen.width==132)
    for(a=0;a<3168;a++) scr_direct(scr_screen.wordmap[a],a%132,a/132);
  else
    for(a=0;a<1920;a++) scr_direct(scr_screen.wordmap[a],a%80,a/80);
  scr_flashprocess(1);

  b=sprite_area_save((sprite_area*)scr_screen.spritearea,fname);

  scr_colourmap();
  if (scr_screen.width==132)
    for(a=0;a<3168;a++) scr_direct(scr_screen.wordmap[a],a%132,a/132);
  else
    for(a=0;a<1920;a++) scr_direct(scr_screen.wordmap[a],a%80,a/80);
  scr_caret(1);

  if (b!=0)
    {
    werr(0,"Error! %s",b->errmess);
    return(FALSE);
    }
  return(TRUE);
  }

/* Redraw whole sprite incase of colour change */
void scr_totalredraw()
  {
  int a;

  for(a=0;a<1920;a++) scr_direct(scr_screen.wordmap[a],a%80,a/80);
  scr_refresh();
  }

int scr_process(int ch)
  { 
  scr_torawfile(ch);         /* non 0 return indicates problem */
  scr_caret(0);
  return(ansi_process(ch));
  }

int ansi_process(int ch)
  {
  static int state,param[8],paramc,sregion_top,sregion_bottom;
  int a,b; os_error *err;

  scr_caret(0);
  if (state==0 && ch>31)
    {
    if ((err=scr_tofile(ch))!=NULL) scr_tofileproblem(err);
    scr_plotchar(ch|(ch<<8),scr_screen.x++,scr_screen.y);
    if (scr_screen.x==80)
      {
      if (1/*current.ansi&S_WRAP*/)
        {
        scr_screen.x=0; ansi_process(10);
        }
      else scr_screen.x--;
      }
    goto thetail;
    }

  if (ch==-1)
    {
    state=0;
    lastredraw=0; sregion_top=0; sregion_bottom=23;
    scr_screen.charset=ansiset;
    scr_screen.currentattrib=0x70000000;
    scr_setflash(1/*current.ansi&S_FLASH*/);
    scr_screen.width=80;
    return(0);
    }

  switch(state)
    {
    case 0: /* Default */
      {
      switch(ch)
        {
        case 7:
          {
          os_swi0(256+7);
          break;
          }
        case 8:
          {
          if (--scr_screen.x==-1)
            {
            scr_screen.x=79;
            ansi_process(11);
            return(0);
            }
          scr_careton=0;
          break;
          }
        case 9: /* Tab */
          {
          int newx=((((scr_screen.x)/8)+1)*8-1);
          if (++newx>79) newx=79;

          while(scr_screen.x<newx) ansi_process(32);
          scr_careton=0;
          break;
          }
        case 10:
          {
          if ((err=scr_tofile(ch))!=NULL) scr_tofileproblem(err);
          }
        case -10:
          {
          if (++scr_screen.y==(sregion_bottom+1))
            {
            wimp_box block2copy;
            wimp_redrawstr r;
            BOOL out;
            int oldattrib=scr_screen.currentattrib;

            scr_screen.currentattrib&=~REVERSE; /* No background */
            scr_screen.y=sregion_bottom;

            /* Send to scrollback buffer */
            scr_toscrollback(0);

            if (sregion_top==0 && sregion_bottom==23)
              {
              /* No point updating what will disappear! */
              if (scr_screen.miny==0) scr_screen.miny++;
              scr_end();

              scr_scrollmapup();

              block2copy.x0=0;    block2copy.y0=-864;
              block2copy.x1=(scr_screen.width<<4)-1; block2copy.y1=-36;
              wimp_blockcopy(local_handle,&block2copy,0,-828);

              r.box.y0=-24*36;
              r.box.y1=-23*36;
              }
            else
              {
              /* Partial scroll */

              /* Send to scrollback buffer */
              scr_toscrollback(sregion_top);

              /* No point updating what will disappear! */
              if (scr_screen.miny==sregion_top) scr_screen.miny++;
              scr_end();

              scr_partscrollmapup(sregion_top,(sregion_bottom-sregion_top));
              scr_clearline(sregion_bottom);

              block2copy.x0=0;    block2copy.y0=-(36*(sregion_bottom+1));
              block2copy.x1=(scr_screen.width<<4)-1; block2copy.y1=-(36*(sregion_top+1));
              wimp_blockcopy(local_handle,&block2copy,0,-(36*sregion_bottom));

              r.box.y0=-((sregion_bottom+1)*36);
              r.box.y1=-sregion_bottom*36;
              }
            scr_screen.currentattrib=oldattrib;

            r.w=local_handle;
            r.box.x0=0;
            r.box.x1=scr_screen.width<<4;
            wimp_update_wind(&r,&out);
            scr_redraw(&r,&out);
            scr_start();
            return(0);
            }
          else
            {
            if (scr_screen.y>23) scr_screen.y=23;
            }
          break;
          }
        case 11:
        case -11:
          {
          if (--scr_screen.y==(sregion_top-1))
            {
            wimp_box block2copy;
            wimp_redrawstr r;
            BOOL out;
            int oldattrib=scr_screen.currentattrib;

            scr_screen.currentattrib&=~REVERSE; /* No background */
            scr_screen.y=sregion_top;

            if (sregion_top==0 && sregion_bottom==23)
              {
              /* No point updating what will disappear! */
              if (scr_screen.maxy==23) scr_screen.miny--;
              scr_end();

              scr_scrollmapdown();

              block2copy.x0=0;    block2copy.y0=-828;
              block2copy.x1=(scr_screen.width<<4)-1; block2copy.y1=0;
              wimp_blockcopy(local_handle,&block2copy,0,-864);

              r.box.y0=-36;
              r.box.y1=0;
              }
            else
              {
              /* Partial scroll */
              /* No point updating what will disappear! */
              if (scr_screen.maxy==sregion_bottom) scr_screen.maxy--;
              scr_end();

              scr_partscrollmapdown(sregion_bottom,(sregion_bottom-sregion_top));
              scr_clearline(sregion_top);

              block2copy.x0=0;    block2copy.y0=-(36*sregion_bottom);
              block2copy.x1=(scr_screen.width<<4)-1; block2copy.y1=-(36*sregion_top);
              wimp_blockcopy(local_handle,&block2copy,0,-36*(sregion_bottom+1));

              r.box.y0=-36*(sregion_top+1);
              r.box.y1=-36*sregion_top;
              }
            scr_screen.currentattrib=oldattrib;

            r.w=local_handle;
            r.box.x0=0;
            r.box.x1=scr_screen.width<<4;

            wimp_update_wind(&r,&out);
            scr_redraw(&r,&out);
            scr_start();
            return(0);
            }
          else
            {
            if (scr_screen.y<0) scr_screen.y=0;
            }
          break;
          }
        case 12:
        case -2:
          {
          if (ch==12) scr_toscrollback(-1);
          for(a=0;a<24;a++) scr_clearline(a);
          scr_end();
          scr_screen.x=scr_screen.y=scr_careton=0;
          break;
          }
        case 13:
          {
          scr_screen.x=0;
          scr_careton=0;
          break;
          }
        case 27:
          {
          state=1; /* Go into 'esc' state */
          break;
          }
        case (27+128):
          {
          state=2; /* Go into 'CSI rx' state */
          break;
          }
        default:
          {
          if (ch==0) break;
          scr_plotchar(ch|(ch<<8),scr_screen.x++,scr_screen.y);
          if (scr_screen.x==80)
            {
            if (1/*current.ansi&S_WRAP*/)
              {
              scr_screen.x=0; ansi_process(10);
              }
            else scr_screen.x--;
            }
          break;
          }
        }
      return(0);
      }
    case 1: /* Escape received */
      {
      if (ch==0x1b) break;
      state=(ch=='[')?2:0;
      return(0);
      }
    case 2: /* CSI received, get parameters */
      {
      for(a=0;a<8;a++) param[a]=0;
      paramc=0; state=3;
      ansi_process(ch);
      return(0);
      }
    case 3: /* Get data */
      {
      if (isalpha(ch))
        {
        paramc++;
        state=4;
        ansi_process(ch);
        }
      if (ch==';')
        {
        if (paramc<8) paramc++; else state=0;
        }
      if (isdigit(ch)) param[paramc]=(param[paramc]*10)+(ch-'0');
      return(0);
      }
    case 4: /* Got command */
      {
      switch(ch)
        {
        case 'A': /* Move cursor up */
          {
          if (param[0]==0) param[0]++;
          scr_screen.y-=param[0];
          if (scr_screen.y<sregion_top) scr_screen.y=sregion_top;
          scr_careton=0;
          break;
          }
        case 'B': /* Move cursor down */
          {
          if (param[0]==0) param[0]++;
          scr_screen.y+=param[0];
          if (scr_screen.y>sregion_bottom) scr_screen.y=sregion_bottom;
          scr_careton=0;
          break;
          }
        case 'C': /* Move cursor right */
          {
          if (param[0]==0) param[0]++;
          scr_screen.x+=param[0];
          if (scr_screen.x>79) scr_screen.x=79;
          scr_careton=0;
          break;
          }
        case 'D': /* Move cursor left */
          {
          if (param[0]==0) param[0]++;
          scr_screen.x-=param[0];
          if (scr_screen.x<0) scr_screen.x=0;
          scr_careton=0;
          break;
          }
        case 'H':
        case 'f': /* Position cursor */
          {
          if (param[0]==0) param[0]=1;
          if (param[1]==0) param[1]=1;
          if (param[0]>24) param[0]=24;
          if (param[1]>80) param[1]=80;
          scr_screen.y=param[0]-1;
          scr_screen.x=param[1]-1;
          scr_careton=0;
          break;
          }
        case 'J': /* Erase screen */
          {
          int oldattrib=scr_screen.currentattrib;

          scr_screen.currentattrib&=~(REVERSE|0x0f000000); /* No background */
          scr_toscrollback(-1);

          switch(param[0])
            {
            case 0: /* Erase from cursor to end of screen (inc) */
              {
              if (scr_screen.x<79)
                {
                for(a=scr_screen.x;a<80;a++) scr_plotchar(32<<8,a,scr_screen.y);
                }
              if (scr_screen.y<23)
                {
                for(a=scr_screen.y+1;a<24;a++) scr_clearline(a);
                }
              break;
              }
            case 1: /* Erase from start of screen to cursor (inc) */
              {
              if (scr_screen.y>0)
                {
                for(a=0;a<scr_screen.y;a++) scr_clearline(a);
                }
              for(a=0;a<=scr_screen.x;a++) scr_plotchar(32<<8,a,scr_screen.y);
              break;
              }
            case 2: /* Erase whole screen */
              {
              for(a=0;a<24;a++) scr_clearline(a);
              scr_screen.y=scr_screen.x=scr_careton=0; /* Home too */
              break;
              }
            }

          scr_screen.currentattrib=oldattrib;
          scr_end(); scr_start();
          break;
          }
        case 'K': /* Erase line */
          {
          switch(param[0])
            {
            case 0: /* Erase from cursor to end of line (inc) */
              {
              for(a=scr_screen.x;a<80;a++) scr_plotchar(32<<8,a,scr_screen.y);
              break;
              }
            case 1: /* Erase from start of line to cursor (inc) */
              {
              for(a=0;a<=scr_screen.x;a++) scr_plotchar(32<<8,a,scr_screen.y);
              break;
              }
            case 2: /* Erase whole line */
              {
              scr_clearline(scr_screen.y);
              break;
              }
            }
          scr_end(); scr_start();
          break;
          }
        case 'm': /* Change rendition */
          {
          for(b=0;b<paramc;b++)
            {
            a=param[b];
            if (a==0) scr_screen.currentattrib =(7<<FOREGROUND);
            if (a==1) scr_screen.currentattrib|=BOLD;
            if (a==5) scr_screen.currentattrib|=FLASH;
            if (a==7) scr_screen.currentattrib|=REVERSE;

            if (a>=30 && a<=37)
              {
              a-=30;
              scr_screen.currentattrib&=0x8fffffff;
              scr_screen.currentattrib|=(a<<FOREGROUND);
              }
            else
              {
              if (a>=40 && a<=47)
                {
                a-=40;
                if (1/*(current.ansi&S_BRIGHT)*/!=0 && a!=0)
                  {
                  a+=8;
                  }
                scr_screen.currentattrib&=0xf0ffffff;
                scr_screen.currentattrib|=(a<<BACKGROUND);
                }
              }
            }
          break;
          }
        case 'n': /* Device status report */
          {
          switch(param[0])
            {
            case 5:
              {
              /*scr_sendstring("\033[0n");*/ /* No malfunction */
              break;
              }
            case 6: /* Report cursor position */
              {
              char temp[16];

              sprintf(temp,"\033[%d;%dR",scr_screen.y+1,scr_screen.x+1);
              /*scr_sendstring(temp);*/
              break;
              }
            }
          break;
          }
       case 'r': /* Set scrolling region */
          {
          if (param[0]==0) param[0]=1;
          if (param[1]==0) param[1]=24;
          if (param[0]>=param[1]) break;
          if (param[1]>24) break;
          scr_screen.y=sregion_top=param[0]-1;
          sregion_bottom=param[1]-1;
          scr_screen.x=0;
          break;
          }
        case 's': /* Save cursor position */
        case 'u': /* Restore cursor position */
          {
          static int x,y;

          if (ch=='s') { x=scr_screen.x; y=scr_screen.y; }
          if (ch=='u') { scr_screen.x=x; scr_screen.y=y; }
          break;
          }
        }
      state=0;

      return(0);
      }
    }

  thetail:
  if ((clock()-lastredraw)>20)
    {
    scr_end(); scr_start();
    lastredraw=clock();
    }
  return(0);
  }

void scr_colourmap()
  {
  os_regset in; int n;
  
  /* Get colours */
  in.r[0]=-1; in.r[1]=3; os_swix(0x35,&in); /* OS_ReadModeVariable */
  is16=(in.r[2]==15)?TRUE:FALSE;

  if (is16)
    {
    int i; wimp_paletteword j;

    for(n=0;n<16;n++)
      {
      j.word=reqcol[n];
      colourtran_return_colourformode(j,12,(wimp_paletteword*)-1,&i);
      scr_colours[n]=i+(i<<4)+(i<<8)+(i<<12)+(i<<16)+(i<<20)+(i<<24)+(i<<28);
      }
    }
  else
    {
    int i;
    colourtran_select_table(12,(wimp_paletteword*)reqcol,-1,(wimp_paletteword*)-1,trans);
    for(n=0;n<16;n++)
      {
      i=n;
      scr_colours[n]=i+(i<<4)+(i<<8)+(i<<12)+(i<<16)+(i<<20)+(i<<24)+(i<<28);
      }
    }
  }

void scr_housekeep()
  {
  if ((clock()-lastredraw)>5)
    {
    scr_end(); scr_start();
    lastredraw=clock();
    scr_caret(1);
    }
  }

void scr_redraw(wimp_redrawstr *r,int *more)
  {
  int x,y;
  sprite_id id;

  id.s.addr=(sprite_ptr)scr_screen.spriteptr;
  id.tag=sprite_id_addr;
  x=r->box.x0-r->scx; y=r->box.y1-r->scy-(216*4);

  if (is16)
    {
    while(*more)
      {
      wimpt_complain(sprite_put_scaled(scr_screen.spritearea,&id,0,x,y,0,0));
      wimp_get_rectangle(r,more);
      }
    }
  else
    {
    while(*more)
      {
      wimpt_complain(sprite_put_scaled(scr_screen.spritearea,&id,0,x,y,0,trans));
      wimp_get_rectangle(r,more);
      }
    }
  }

void scr_carettype()
  {
  scr_caret(0);
  scr_screen.cursortype=1; /* 0 for block */
  scr_caret(1);
  }

void scr_caret(int on)
  {
  if (on)
    {
    scr_docursor(cursorphase);
    scr_careton=1;
    }
  else
    {
    scr_docursor(0);
    scr_careton=0;
    }
  }

void scr_setcaret(int on)
  {
  wimp_caretstr c;

  if (on==-1)
    {
    on=1; scr_careton=0;
    }

  scr_carethere=on;

  if (on)
    {
    if (scr_careton==0)
      {
      c.w=local_handle; c.i=-1; c.x=-100; c.y=0;
      c.height=0; wimp_set_caret_pos(&c);
      scr_docursor(cursorphase);
      scr_careton=1;
      /*scr_claimkeypad();*/
      }
    }
  else
    {
    scr_careton=0;
    }
  }

void scr_select(int a)
  {
  scr_direct(scr_screen.wordmap[a]|SELECT,a%80,a/80);
  }

void scr_deselect(int a)
  {
  scr_direct(scr_screen.wordmap[a]&~SELECT,a%80,a/80);
  }

int scr_getdata(int a)
  {
  return((scr_screen.wordmap[a]>>8)&0xff);
  }

int inkey(int key)
  { 
  int x=(key^0xff),y=0xff;

  os_byte(129,&x,&y);
  return(x==0xff && y==0xff);
  }

int open_spool(char *toleaf)
  {
  os_filestr create; os_regset r;
  os_error *error;

  r.r[0]=0x4f; r.r[1]=(int)toleaf;
  if ((error=os_find(&r))!=NULL)
    {
    create.action=11; create.name=toleaf;
    create.loadaddr=0xfff;
    create.start=create.end=0;
    if ((error=os_file(&create))!=NULL)
      {
      werr(0,"Can't create spoolfile (%s)",error->errmess);
      return(0);
      }
    }
  else
    {
    /* Close file! */
    r.r[1]=r.r[0];
    r.r[0]=0;
    os_find(&r);
    }

  r.r[0]=0xcf; r.r[1]=(int)toleaf;
  if ((error=os_find(&r))!=NULL)
    {
    werr(0,"Can't open spoolfile (%s)",error->errmess);
    remove(toleaf);
    return(0);
    }

  scr_spoolfile=r.r[0];
  scr_spoolstrip=1;

  /* Skip to end for append */
  r.r[0]=2; r.r[1]=scr_spoolfile; os_args(&r);
  r.r[0]=1; os_args(&r);

  return(TRUE);
  }

BOOL start_spool(char *toleaf,void *handle)
  {
  return(open_spool(toleaf));
  }

void close_spool()
  {
  if (scr_spoolfile!=0)
    {
    os_regset r;

    /* Close spoolfile */
    r.r[0]=0; r.r[1]=scr_spoolfile;
    os_find(&r);

    scr_spoolfile=0;
    }
  }

void do_local_menu(void *handle,char *hit)
  {
  handle=handle;

  switch(hit[0])
    {
    case 1:
    /* Spool to */
      {
      saveas(0xfff,"Spoolfile",0,start_spool,0,0,0);
      break;
      }
    case 2:
    /* Close spoolfile */
      {
      close_spool();
      break;
      }
    case 3:
    /* Screen save */
      {
      saveas(0xfff,"Screen",0,scr_screensave,0,0,0);
      break;
      }
    case 4:
    /* Sprite save */
      {
      saveas(0xff9,"Sprite",0,scr_spritesave,0,0,0);
      break;
      }
    case 5:
    /* Message save */
      {
      char name[10];
      
      sprintf(name,"Msg%07d",getmsgnr());
      saveas(0xfff,name,0,message_save,0,0,0);
      break;
      }
    }
  }

void snoop_buffer(int d)
  {
  scr_process(d);
  }
