/*                  _____________________________________________
                  [>                                             <]
Project           [> PortNet communications network              <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Chat window                                 <]
Current version   [> 00.04                                       <]
Version date      [> 04-November-1992                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"
#include "config.h"
#include "servmess.h"
#include "userlog.h"
#include "modcomm.h"
#include "chat.h"

#define COLOUR_1 11
#define COLOUR_2 15

static short chat_map[24][80];
static short *chat_ptr[24];
int  chat_cursorx=0,chat_cursory=0;
extern wimp_w chat_handle;

void chat_redraw()
  {
  int left,right,top,bottom,a,b,c,last;
  wimp_redrawstr myredraw; 
  BOOL more;

  myredraw.w=chat_handle;
  wimp_redraw_wind(&myredraw,&more);

  while (more)
    {
    left  = (myredraw.g.x0-(myredraw.box.x0-myredraw.scx))/16;    /* Find the text lines  */
    right = (myredraw.g.x1-(myredraw.box.x0-myredraw.scx))/16;    /* we NEED to redraw    */
    top   =-(myredraw.g.y1-(myredraw.box.y1-myredraw.scy))/32;    /* And then redraw them */
    bottom=-(myredraw.g.y0-(myredraw.box.y1-myredraw.scy))/32;
    right++;

    if (left<0) left=0;
    if (top<0) bottom=0;
    if (right>79) right=79;
    if (bottom>23) bottom=23;

    for(a=top;a<=bottom;a++)
      {
      bbc_move(((left*16)+myredraw.box.x0),(myredraw.box.y1-(32*a))-1);

      bbc_gcol(0,COLOUR_1); last=0;
      for(b=left;b<=right;b++) 
        {
        c=chat_ptr[a][b];
        if ((c&256)==0 && last!=0) { bbc_gcol(0,COLOUR_1); last=0; }
        if ((c&256)!=0 && last==0) { bbc_gcol(0,COLOUR_2); last=1; }
        bbc_vdu(c&255);
        }
      }

    wimp_get_rectangle(&myredraw,&more);
    }
  }
  
void chat_data(int ch)
  {
  int old_x=chat_cursorx,old_y=chat_cursory;
  wimp_redrawstr fred; wimp_caretstr temp;         

  switch(ch)
    {
    /*********** Clear screen ? If so, fool wimp to clear the window for us, */
    /**************************************** and then clear our screen map. */
    case 12:
      {
      BOOL more; wimp_redrawstr myredraw;

      chat_clearmap(); chat_cursorx=chat_cursory=0;
      myredraw.box.x0=0;    myredraw.box.x1=1200;
      myredraw.box.y0=-900; myredraw.box.y1=0;
      myredraw.w=chat_handle;
      wimp_update_wind(&myredraw,&more);
      while(more)
        {
        wimp_get_rectangle(&myredraw,&more);
        }
      goto ret;
      }

    /********************************************************** Home cursor? */
    case 30:
      {
      chat_cursorx=chat_cursory=0;
      goto ret;
      }

    /******************************************************************* Cr? */
    case 13:
      {
      chat_cursorx=0;
      goto ret;
      }

    /********* LF? If so, and scroll is necessary, do some sneaky redrawing: */
    /**************** First, we scroll each rectangle we are asked to update */
    /********* Then, we redraw the bottom line of each rectangle's text area */
    /************************ (ie anything that was uncovered by the scroll) */
    case 10:
      {
      int a;

      chat_cursorx=0;

      if (++chat_cursory==24)
        {
        wimp_box block2copy;
        short *newline=chat_ptr[0];                           

        for(a=1;a<24;a++) chat_ptr[a-1]=chat_ptr[a];
        chat_ptr[23]=newline;
        for(a=0;a<80;a++) chat_ptr[23][a]=0;
        chat_cursory=23;
        
        wimp_get_caret_pos(&temp);
        if (temp.w==chat_handle)
          {
          /* Turn off caret */
          temp.i=-1;
          temp.x=chat_cursorx<<4;            
          temp.y=-(chat_cursory<<5)-32;
          temp.height=0;
          temp.index=-1;
          wimp_set_caret_pos(&temp);
          }

        /* Scroll! */
        block2copy.x0=0;    block2copy.y0=-1024;
        block2copy.x1=1279; block2copy.y1=-32;
        wimp_blockcopy(chat_handle,&block2copy,0,-992);
        }
      goto ret;
      }

    /******************************************** Non-destructive backspace? */
    case -8:
      {
      if (--chat_cursorx==-1)
        { 
        chat_cursorx=0;
        if (--chat_cursory<0) chat_cursory=0;
        }
      goto ret;
      }

    /********************************************************* Forwardspace? */
    case 9:
      {
      if (++chat_cursorx==80)
        {
        chat_cursorx=0;
        chat_data(10);
        }
      goto ret;
      }

    /************************************************ Destructive backspace? */
    case 8:
    case 127:
      {
      chat_data(-8); chat_data(32); chat_data(-8);
      break;
      }  

    /********************************************************** Normal text? */
    default:
      {
      if (ch>31) 
        {
        chat_ptr[chat_cursory][chat_cursorx]=ch;
        chat_data(9);
        }
      break;
      }
    }

  /********* If we've fallen through to here, update the char. space printed */
  fred.w=chat_handle;
  fred.box.x0=(16*old_x);
  fred.box.y0=-(32*(old_y+1));
  fred.box.x1=(16*(old_x+1));
  fred.box.y1=-(32*old_y);
  wimp_force_redraw(&fred);

  ret:
  wimp_get_caret_pos(&temp);
  if (temp.w==chat_handle)
    {
    temp.i=-1;
    temp.x=chat_cursorx<<4;
    temp.y=-(chat_cursory<<5)-32;
    temp.height=32;
    temp.index=-1;

    /* Stick caret in that window */
    wimp_set_caret_pos(&temp);
    }
  }

void chat_clearmap()
  {
  int a,b;
  for(a=0;a<24;a++)
    {
    chat_ptr[a]=chat_map[a];
    for(b=0;b<80;b++) chat_map[a][b]=32;
    }
  }
