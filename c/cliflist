/*                  _____________________________________________
                  [>                                             <]
Project           [> PortNet communications network              <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> CLI filelist generation                     <]
Current version   [> 00.22                                       <]
Version date      [> 04-November-1992                            <]
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

/* PreDeclarations */
int  main(int,char**);

/* Global variables */
char    *message_buffer,*_message_buffer,buffer[1024];

mail_block fileheader;

void send_message(int task,int action,int word1,int word2,int word3,int word4)
  {
  }
                  
struct { char key,length,title[60],format[8]; int type; void *thing; } fields[]=
  { {'#', 6,"File #"                        ,"%06d " ,0,&fileheader.file_location},
    {'n',20,"Name                "          ,"%-20s ",1,fileheader.to            },
    {'l', 6,"Length"                        ,"%6d "  ,0,&fileheader.file_length  },
    {'c', 6,"Dloads"                        ,"%6d "  ,0,&fileheader.to_id        },
    {'s',60,"Short description"
                                            ,"%s"    ,1,fileheader.subject       },
    {'d', 8,"Date    "                      ,""      ,2,0                        },
    {'t', 8,"Time    "                      ,""      ,3,0                        },
    {  0, 0,""                              ,0,0                                 } };

int main(int argc,char *argv[])
  {
  int filearea,current,max=areamax(),total_here,total=0,a,b,c=0;
  mail_area area; char *p;
  FILE *out;
  struct tm *ti;

  if (argc<2) exit(1);
  if (argc==2) argv[2]="#nls";

  if ((out=fopen(argv[1],"w"))==NULL) exit(2);

  for(filearea=0;filearea<max;filearea++)
    {
    read_area(filearea,&area);
    if ((area.areaflags&FLAG_FILEAREA)!=0 &&
        (area.areaflags&FLAG_READABLE)!=0)
      {          
      total_here=0;
      fprintf(out,"\nArea: %s (%d)\n",area.name,filearea);
      
      /* Generate titles */
      p=argv[2];
      while(*p)
        {
        a=0; while(fields[a].key) if (fields[a].key==*p) break; else a++;
        if (fields[a].key) fprintf(out,"%s ",fields[a].title);
        p++;
        }
      fprintf(out,"\n");
      p=argv[2];
      while(*p)
        {
        a=0; while(fields[a].key) if (fields[a].key==*p) break; else a++;
        if (fields[a].key) { b=fields[a].length; while(b--) fputc('-',out); fputc(' ',out); }
        p++;
        }
      fprintf(out,"\n");

      current=area.last_message;
      if (current==-1)
        {
        fprintf(out,"No files\n");
        }   
      else
        {
        do
          {
          fileheader.message_number=current;
          get_messageh(&fileheader);

          if ((fileheader.flags&FILE_SYSOP)==0)
            {
            fileheader.subject[40]=0;

            p=argv[2];
            while(*p)
              {
              a=0; while(fields[a].key) if (fields[a].key==*p) break; else a++;
              if (fields[a].key)
                {
                switch(fields[a].type)
                  {
                  case 0:
                    fprintf(out,fields[a].format,*((int*)fields[a].thing));
                    break;
                  case 1:
                    fprintf(out,fields[a].format,(char*)fields[a].thing);
                    break;
                  case 2:
                    ti=localtime(&fileheader.date_sent);
                    fprintf(out,"%2d/%02d/%02d ",ti->tm_mday,ti->tm_mon+1,ti->tm_year);
                    break;
                  case 3:
                    ti=localtime(&fileheader.date_sent);
                    fprintf(out,"%02d:%02d:%02d ",ti->tm_hour,ti->tm_min,ti->tm_sec);
                    break;
                  }
                }
              p++;
              }
            fprintf(out,"\n");

            total_here+=fileheader.file_length;
            total+=fileheader.file_length;

            c++;
            if ((c%10)==0) printf("\rProcessed %5d",c);
            }
          current=fileheader.message_backward;  
          }
        while(current!=-1);
        }
      fprintf(out,"\nSize of area %dk\n\n\n",total_here/1024);
      }
    }
  fprintf(out,"\n\nTotal size of filebase %dk\n",total/1024);
  fclose(out);

  printf("\rProcessed %5d\n",c);

  exit(0);
  return(0);
  }


void window_poll() /* Fudge for servcomm.c */
  {
  }
