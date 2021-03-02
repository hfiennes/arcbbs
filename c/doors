/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Doors interface                             <]
Current version   [> 00.08                                       <]
Version date      [> 29-January-1992                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"         /* Standard library files */
#include "config.h"          /* System configuration */
#include "servmess.h"        /* Server message info */
#include "port.h"            /* Port driver functions */
#include "userlog.h"         /* Userlog format */
#include "mail.h"            /* Mail format */
#include "portmisc.h"        /* Misc port commands */
#include "servcomm.h"        /* Misc server comm commands */
#include "mprintf.h"         /* Modem printf */
#include "modcomm.h"         /* Module interface */

#include "bbs.h"
#include "doors.h"
#include "doorsa.h"

doors_request doors_req;
int doors_inuse=0;

void doors_reset()
  {
  doors_clearinput(portnumber);
  doors_clearoutput(portnumber);
  doors_writestatus(portnumber,0);
  doors_resetstate(portnumber);
  doors_inuse=0;
  }

void doors_dorequest(doors_request *dr)
  {
  doors_reply r; int a;

  switch(dr->request)
    {
    case 0:
    /* Read general user information */
      {
      r.w[0]=loggedon?user.usernumber:-1;
      r.w[1]=user.t_firstlogon;
      r.w[2]=user.t_lastlogon;
      r.w[3]=user.mailcount;
      r.w[8]=user.termtype;
      r.w[9]=user.f_message;
      r.w[10]=user.f_file;
      r.w[11]=user.f_user;
      r.w[12]=user.ratio;
      r.w[13]=user.userlevel;
      r.w[14]=user.logons;
      r.w[15]=user.uploads;
      r.w[16]=user.downloads;
      r.w[17]=user.timeallowed;
      r.w[18]=user.timetoday;
      r.w[19]=user.netcredit;
      r.w[20]=user.fidoflags;
      r.w[21]=user.conference;
      r.w[22]=user.filebase;
      r.w[25]=user.outboxcount;
      for(a=0;a<64;a++) r.b[104+a]=user.conferences[a];
      r.w[42]=(clock()-logon_time)/100;
      r.w[43]=(thislogon*60)-r.w[42];
      r.w[44]=thislogon*60;
      r.w[45]=baudrate;
      r.b[184]=arq?1:0;
      r.b[185]=user.callrate;
      r.b[186]=user.pagelen;
      strcpy(&r.b[187],user.username);
      doors_sendreply(portnumber,&r);
      break;
      }
    case 1:
    /* Read user address */
      {
      strcpy(&r.b[0],user.username);
      strcpy(&r.b[31],user.realname);
      strcpy(&r.b[62],user.address[0]);
      strcpy(&r.b[93],user.address[1]);
      strcpy(&r.b[124],user.address[2]);
      strcpy(&r.b[155],user.address[3]);
      strcpy(&r.b[186],user.postcode);
      strcpy(&r.b[197],user.telephone);
      doors_sendreply(portnumber,&r);
      break;
      }
    default:
      {
      if (dr->request>=100 && dr->request<199)
        {
        get_user(&user);
        switch(dr->request)
          {
          case 100: user.uploads=dr->data.w[0]; break;
          case 101: user.downloads=dr->data.w[0]; break;
          case 102: user.ratio=dr->data.w[0]; break;
          case 103: thislogon=dr->data.w[0]/60; break;
          case 104: user.timeallowed=dr->data.w[0]; break;
          case 105: user.userlevel=dr->data.w[0]; break;
          case 106: user.f_message=dr->data.w[0]; break;
          case 107: user.f_file=dr->data.w[0]; break;
          case 108: user.f_user=dr->data.w[0]; break;
          }
        put_user(&user);
        doors_resetstate(portnumber);
        break;
        }

      if (dr->request>=200)
        {
        break;
        }
      break;
      }
    }
  }

int _doors_process()
  {
  /* Connection established, basically do it */
  do
    {
    while(port_rxbuffer() && doors_inputstatus(portnumber)<1024) doors_inputwrite(portnumber,port_rx());
    while(port_txbuffer() && doors_outputstatus(portnumber)<1024) port_txw(doors_outputread(portnumber));
    window_poll();
    }
  while(doors_readstatus(portnumber)==255 && port_dcd()!=0);

  /* Display any last data */
  if (port_dcd())
    {
    while(doors_outputstatus(portnumber)<1024)
      {
      while(port_txbuffer() && doors_outputstatus(portnumber)<1024) port_txw(doors_outputread(portnumber));
      window_poll();
      }
    }

  /* Close down link */
  doors_clearinput(portnumber);
  doors_clearoutput(portnumber);
  doors_writestatus(portnumber,0);

  doors_inuse=0;

  return(0);
  }

int doors_connect(int doornumber)
  {
  clock_t now=clock();

  doors_inuse=1;

  /* Clear buffers both ways */
  doors_clearinput(portnumber);
  doors_clearoutput(portnumber);
  doors_resetstate(portnumber);

  /* Set connect required status */
  doors_writestatus(portnumber,doornumber);

  /* Wait for response */
  while(doors_readstatus(portnumber)!=255 && (clock()-now)<500) window_poll();
  if (doors_readstatus(portnumber)!=255)
    {
    doors_writestatus(portnumber,0);
    doors_inuse=0;
    return(1);
    }

  return(_doors_process());
  }

int doors_forceconnect()
  {
  doors_inuse=1;

  /* Clear buffers both ways */
  doors_clearinput(portnumber);
  doors_clearoutput(portnumber);
  doors_resetstate(portnumber);

  /* Say we're connected */
  doors_writestatus(portnumber,255);

  return(_doors_process());
  }
