/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Line editor                                 <]
Current version   [> 00.21                                       <]
Version date      [> 12-November-1992                            <]
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
#include "crc.h"             /* CRC generator */
#include "parser.h"          /* Command parser */
#include "mail.h"            /* Mail format */
#include "portmisc.h"        /* Misc port commands */
#include "servcomm.h"        /* Misc server comm commands */
#include "mprintf.h"         /* Modem printf */
#include "arclist.h"         /* ARC file lister */
#include "modcomm.h"         /* Module interface */
#include "fido.h"            /* Fido stuff */

#include "miscbbs.h"
#include "bbs.h"

int xstrlen(char *s)
  {
  int a=0;

  while(*s++>31) a++;
  return(a);
  }
                    
int entertext(mail_block *header,int file)
  {
  int pos,line=0,newpos=-1,temp,ch,quit;
  char *array=message_buffer,*menu,*orig_text=NULL,initials[5],
       *quoteptr=NULL,*disp;  
  mail_block *orig_header=NULL;

  /* Setup so no timeput when entering a message */
  notimeout=1;
    
  /* Check to see if we are replying for quoting purposes */
  if (header->reply_from!=-1)
    {                        
    /* Allocate some memory for the message being replied to */
    if ((orig_header=malloc(256))==0)
      {
      mprintf(1,"{bfg r}Couldn't allocate memory for quoting buffer. Quoting disabled.{std}");     
      }
    else
      {                       
      orig_header->message_number=header->reply_from;

      /* Get the header */
      get_messageh(orig_header);

      /* Allocate memory for the actual text */
      if ((orig_text=malloc(orig_header->message_length+10))==0)
        {                       
        /* Can't allocate text space, free header space. */
        free(orig_header);
        mprintf(1,"{bfg r}Couldn't allocate memory for quoting buffer. Quoting disabled.{std}");     
        }
      else
        {                     
        char *init=orig_header->from;
        int iptr=1;
     
        /* Read in the whole message */
        memset(orig_text,0,orig_header->message_length+9);
        get_message(orig_header,orig_text); 

        /* Make up initials */
        initials[0]=*init++;
        while(*init && iptr<4)
          {
          if (*init==' ' && *(init+1) && *(init+1)!=' ') 
            {
            initials[iptr++]=toupper(*(init+1));
            }
          init++;
          }
        initials[iptr]=0;
        quoteptr=orig_text;
        }      
      }
    }

  /* Clear text buffer lines */
  for(temp=0;temp<100;temp++) array[temp*80]=0;
             
  /* Setup correct text file */
  if (file)
    {
    menu="<ARCbbs$prompts>.enter_f";
    }
  else
    {
    menu=(header->message_area==0)?"<ARCbbs$prompts>.enter_r":"<ARCbbs$prompts>.enter_v";
    }
     
  if (file==0)
    {
    /* Get subject */
    port_txstring("{fg c}Subject: ",1);         
    port_readline(header->subject,60,EXISTING);
    port_crlf();
    }
  else
    {
    /* Get short description */
    port_txstring("{fg c}Short description: ",1);         
    port_readline(header->subject,60,EXISTING);
    mprintf(1,"\n{bfgbg cn}Long description:\n");
    }               

  mprintf(1,"\n{bfgbg cb}ARCbbs line editor v1.05 {fg w flash}Type /? {std bfgbg wb}on blank line for help{eol std}\n\n");

  quit=0;
  do
    {
    /* Send prompt */
    if (newpos==-1)
      {
      array[line*80]=0; pos=0;
      }
    else
      {
      pos=newpos; newpos=-1;
      }
    mprintf(1,"{bfg y}%02d>{fg c}",line+1);
    disp=array+(line*80);
    while(*disp>31) port_txw(*disp++);

    /* Get text */
    do
      { 
      getagain:
      ch=port_get();
    
      lastline:     
      if ((pos==0 && ch=='/') || line>99)
        {
        /* Ed command */
        gothelp:
        mprintf(1,"\015{bfg g}Ed>{fg w}");
        switch(toupper(port_get()))
          {
          case '?': /* Help */
            {
            bbs_sendfile(menu,0,0);
            goto gothelp;
            }            
          case 'Q': /* Quote from parent message */
            { 
            if (quoteptr)
              {
              /* Grab next line from text, if it's hidden (^A) skip it */
              grabagain:
              if ((quoteptr-orig_text)>=orig_header->message_length)
                {
                quoteptr=orig_text;
                }
              if (*quoteptr==1)
                {
                /* It's a hidden fidonet line, skip it */
                quoteptr+=(strlen(quoteptr)+1);       
                goto grabagain;
                }

              /* Have a good go at wrapping it (if it's too long, here 'too
                 long' refers to 70 characters!) */
              if (strlen(quoteptr)>70)
                {
                char *oldquoteptr=quoteptr,oldchr;

                /* Skip to 70 and work back, looking for a space, ,;:.?! */
                quoteptr+=70;
                while(quoteptr>oldquoteptr)
                  {
                  if (strchr(" !?.;:,",*quoteptr)!=NULL)
                    {
                    break;
                    }
                  else
                    {
                    quoteptr--;
                    }
                  }                                     
                /* If we couldn't wrap, just split! */
                if (quoteptr==oldquoteptr) quoteptr+=70;

                /* Save 'next' character */
                oldchr=*(quoteptr+1);
                *(quoteptr+1)=0;
                sprintf(&array[line*80],"%s> %s",initials,oldquoteptr);
                *(quoteptr+1)=oldchr;
                }
              else
                {
                sprintf(&array[line*80],"%s> %s",initials,quoteptr);
                quoteptr+=(strlen(quoteptr)+1);
                }
                
              mprintf(1,"\015{fg y}Qt>{fg c}");
              port_txstring(&array[line*80],16+1);
              notit:
              switch(port_get())
                {
                case '+':
                case '/': 
                  {
                  if (user.termtype==TER_VT100 || user.termtype==TER_ANSI)
                    {
                    mprintf(1,"\015{eol}");
                    }
                  else
                    {
                    int a;
                    port_txw(13);
                    for(a=0;a<79;a++) port_txw(32);
                    }
                  goto grabagain;
                  }
                case '-':
                  {          
                  quoteptr-=120;
                  while(quoteptr>orig_text) if (strchr(" !?.;:,",*quoteptr)!=NULL) break; else quoteptr--;
                  if (quoteptr<orig_text) quoteptr=orig_text;
                  if (user.termtype==TER_VT100 || user.termtype==TER_ANSI)
                    {
                    mprintf(1,"\015{eol}");
                    }
                  else
                    {
                    int a;
                    port_txw(13);
                    for(a=0;a<79;a++) port_txw(32);
                    }
                  goto grabagain;
                  }
                case 13:
                case 10:
                  {
                  mprintf(1,"\015{fg y}%02d>\n",line+1); break;
                  } 
                default:
                  {
                  goto notit;
                  }
                }
              line++;
              }
            break;
            }
          case 'L': /* List message */
            {
            int l;

            port_txstring("List",1);
            for(l=0;l<line;l++)
              {
              mprintf(1,"\n{bfg y}%02d>{fg c}",l+1);
              disp=&array[l*80]; while(*disp>31) port_txw(*disp++);
              }
            port_crlf();
            break;
            }
          case 'R': /* Recorded delivery */
            {
            if (header->message_area==0)
              {
              port_txstring("Recorded delivery\n",1);
              header->flags|=MSG_CONFIRM;
              }
            break;
            }
          case 'X': /* eXpress */
            {
            if (header->message_area==0)
              {
              port_txstring("Express delivery\n",1);
              header->flags|=MSG_EXPRESS;
              }
            break;
            }
          case 'V': /* Voting message */
            {
            if (header->message_area!=0)
              {
              header->flags^=MSG_VOTE;
              port_txstring("Voting ",1);
              port_txstring((header->flags&MSG_VOTE)?"ON\n":"OFF\n",1);
              }
            break;
            }
          case '+': /* File link */
            {
            char filenr[10];

            port_txstring("File link : Link message to file #",1);
            port_readline(filenr,9,NUMONLY);
            port_crlf();

            if (filenr[0])
              {
              int fnr=atoi(filenr);
        
              if (fnr<0 || fnr>read_filenumber())
                {
                port_txstring("{bfg r}No such filenumber{std}\n",1);
                }
              else
                {
                int count,found=0,max=areamax();
                mail_area area; mail_block fileheader;
                clock_t lastpoll=clock();

                port_txstring("{bfg c}Searching for file in area: {bfg r}",1);
                   
                for(count=0;count<max && found==0;count++)
                  {
                  read_area(count,&area);

                  if ((clock()-lastpoll)>=MAXIMUM_POLL)
                    {
                    window_poll(); lastpoll=clock();
                    }

                  if ((area.areaflags&FLAG_FILEAREA)!=0 && can_read(&area) && area.name[0]!=0)
                    {
                    int current=area.last_message;

                    mprintf(1,"%3d\010\010\010",count);

                    if (current!=-1)
                      {
                      do
                        {
                        if ((clock()-lastpoll)>=MAXIMUM_POLL)
                          {
                          window_poll(); lastpoll=clock();
                          }
                        fileheader.message_number=current;
                        get_messageh(&fileheader);
                        current=fileheader.message_backward;
                        }
                      while(fileheader.file_location!=fnr && current!=-1);
                
                      if (fileheader.file_location==fnr) found=1;
                      }
                    }
                  }

                if (found==0)
                  {
                  port_txstring("{std}\n{bfg r}File not found{std}\n",1);
                  }
                else
                  {
                  header->file_location=fileheader.message_number;
                  header->file_length=fileheader.file_length;
                  mprintf(1,"{std}\n{bfg g}File found: '%s', length=%d bytes{std}\n",
                       fileheader.to,fileheader.file_length);
                  }
                }
              }
            break;
            }      
          case 'E': /* Edit line */
            {
            char tb[3],l[82],*d,*d2,old_eol;
            int lin;

            port_txstring("Edit : Line #",1);
            port_readline(tb,2,0);
            port_crlf();

            lin=atoi(tb);
            if (lin!=0)
              {
              mprintf(1,"{bfg y}%02d>{fg c}",lin);
              d=disp=&array[(lin-1)*80]; d2=l;
              while(*d>31) *d2++=*d++; *d2=0; old_eol=*d;
              port_readline(l,76,EXISTING);
              d=disp; d2=l;
              while(*d2) *d++=*d2++; *d=old_eol;
              }
            port_crlf();
            break;
            }
          case 'D': /* Delete line */
            {
            char tb[3];
            int lin;

            port_txstring("Delete : Line #",1);
            port_readline(tb,2,0);

            lin=atoi(tb);
            if (lin!=0)
              {
              int a;
              for(a=(lin-1);a<99;a++) strcpy(&array[a*80],&array[(a+1)*80]);
              array[7920]=0;
              line--;

              port_txstring("{bfg g} - deleted{std}\n",1);
              }
            else
              {
              port_txstring("{bfg g} - delete {fg r}aborted{std}\n",1);
              }

            break;
            }
          case 'I': /* Insert line */
            {
            char tb[3];
            int lin;

            port_txstring("Insert : Before line #",1);
            port_readline(tb,2,0);

            lin=atoi(tb);
            if (lin!=0)
              {
              int a;
              for(a=99;a>=lin;a--) strcpy(&array[a*80],&array[(a-1)*80]);
              array[80*(lin-1)]=0;
              line++;
              port_txstring("{bfg g} - blank line inserted{std}\n",1);
              }
            else
              {
              port_txstring("{bfg g} - insert {fg r}aborted{std}\n",1);
              }
            break;
            }
          case 'A': /* Abort */
            {
            if (port_yesno("Abort : Are you sure? {fg y}"))
              {
              notimeout=0;
              if (orig_header) free(orig_header);
              if (orig_text) free(orig_text);
              return(0);
              }
            port_crlf();
            break;
            }
          case 'S': /* Send */
            {
            int answer;
            port_txstring("Send : [Return] to confirm",1);
            do
              {
              answer=port_get();
              }
            while(answer!=13 && answer!=8 && answer!=127);
            if (answer==13)
              {
              int pos=0,l=0,p;

              /* Pack message */
              for(;l<line;l++)
                {
                p=0;
                while((array[pos++]=array[l*80+p])>31) p++;
                if (array[pos-1]==1) array[pos-1]=32;
                }
              header->message_length=pos;
              port_crlf();                    

              if (orig_header) free(orig_header);
              if (orig_text) free(orig_text);
              notimeout=0; return(1);
              }
            port_crlf();
            break;
            }
          case 'F': /* Find & replace */
            {
            char tb[3],find[84],replace[84],*lpoint,*fpoint;
            int lin;

            port_txstring("Find & Replace : Line #",1);
            port_readline(tb,2,0);
            port_crlf();           

            lin=atoi(tb);
            if (lin!=0)
              {             
              lpoint=&array[(lin-1)*80];
              port_txstring("{bfg g}Find string\n",1);
              port_readline(find,40,0); port_crlf();
              if (find[0])
                {                                      
                if ((fpoint=strstr(lpoint,find))==NULL)
                  {
                  port_txstring("{bfg r}Can't find that string!{std}\n",1);
                  }
                else
                  {
                  port_txstring("{bfg g}Replace string\n",1);
                  port_readline(replace,(76-xstrlen(lpoint))+strlen(find),0);
                  port_crlf();
                  strcpy(find,fpoint+strlen(find));
                  strcpy(fpoint,replace);
                  strcat(fpoint,find);
                  mprintf(1,"{bfg y}%02d>{fg c}",lin);
                  while(*lpoint>31) port_txw(*lpoint++);
                  port_crlf();
                  }
                }
              }
            break;
            }
          case 'M': /* Modify subject/desc */
            {
            if (file==0)
              {
              /* Get subject */
              port_txstring("Subject: ",1);         
              port_readline(header->subject,60,EXISTING);
              port_crlf();
              }
            else
              {
              /* Get short description */
              port_txstring("Short desc: ",1);         
              port_readline(header->subject,60,EXISTING);
              }               
            break;
            }
          case 'C': /* Centre line above */
            {     
            mprintf(1,"Centre previous line\n");

            if (line!=0)
              {
              char *lpoint=&array[(line-1)*80],newline[80];
              int a,len=strlen(lpoint);
                           
              if (len<76)
                {
                for(a=0;a<((76-len)/2);a++)
                  {
                  newline[a]=32;
                  }
                strcpy(newline+a,lpoint);
                strcpy(lpoint,newline);
                mprintf(1,"{fg y}%02d>{fg c}%s\n",line,lpoint);
                }
              else
                {
                mprintf(1,"{bfg r}Too long!{std}\n");
                }
              }
            break;
            }
          case '/': /* Enter a '/' */
            {             
            mprintf(1,"\015{bfg y}%02d>{fg c}",line+1);
            goto messagecont;
            }
          default:
            {
            port_txw(13);
            break;
            }
          }
        if (line<100) mprintf(1,"{bfg y}%02d>{fg c}",line+1);
        goto getagain;
        }         
      
      if (line>99) goto lastline;

      messagecont:
      switch(ch)
        {
        case 8:
        case 127:
          {
          if (pos)
            {
            port_txstring("\010 \010",1);
            pos--;
            }
          break;
          }
        case 9:
          {
          if (pos<65)
            {
            int a;
            
            for(a=0;a<10;a++) port_txw(array[line*80+(pos++)]=32);
            }
          break;
          }
        default:
          {
          if (ch>31)
            {
            if (pos>75)
              {
              if (ch!=32)
                {
                int p=75;
                /* Wrap last word */
                while(array[line*80+p]!=' ' && p!=40) p--;
                if (array[line*80+p]==' ')
                  {
                  int a,b;
                  array[line*80+p]=1;
                  for(a=0,p++;p<76;) array[(line+1)*80+(a++)]=array[line*80+(p++)];
                  for(b=0;b<a;b++) port_txw(8);
                  for(b=0;b<a;b++) port_txw(' ');
                  array[(line+1)*80+(a++)]=ch;
                  array[(line+1)*80+a]=0;
                  newpos=a;
                  }
                else
                  {
                  array[(line+1)*80]=ch;
                  array[(line+1)*80+1]=0;
                  newpos=1;
                  }
                }
              else
                {
                newpos=0;
                }
              }
            else
              {
              port_txw(array[line*80+(pos++)]=ch);
              }
            }
          break;
          }
        }
      }
    while(newpos==-1 && ch!=10 && ch!=13);
    array[line*80+pos]=0; port_crlf();
    line++;
    }
  while(1);
  return(0);
  }
