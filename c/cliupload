/*                  _____________________________________________
                  [>                                             <]
Project           [> PortNet communications network              <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> CLI upload                                  <]
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
int  main(int,char**),filesize(char*);
char *file_pathname(int);

/* Global variables */
int     size,filetype;
mail_block fileheader; mail_area area;
char    *message_buffer,*_message_buffer,buffer[1024];

void send_message(int task,int action,int word1,int word2,int word3,int word4)
  {
  }

int main(int argc,char *argv[])
  {
  mail_block header; user_block uploader;
  int flags=0,thisblock;
  char fbuffer[10240];
  FILE *inf,*outf;
  os_filestr attr; int attr_load,attr_exec;

  if (argc!=6) exit(100);

  memset(&header,0,255);

  _message_buffer=buffer;
  message_buffer=(_message_buffer+9);
                                  
  /* Get area number */
  if ((header.message_area=atoi(argv[1]))>areamax()) exit(1);
  read_area(header.message_area,&area);
  if ((area.areaflags&FLAG_FILEAREA)==0) exit(2);

  /* Get user number */                
  uploader.usernumber=atoi(argv[2]); get_user(&uploader);

  header.reply_from=-1;
  header.from_id=uploader.usernumber;
  strcpy(header.from,uploader.username);
  strcpy(header.to,argv[3]);
  strcpy(header.subject,argv[4]);

  header.to_id=0; /* Number of downloads */
  header.file_length=filesize(argv[5]);
  header.file_location=get_filenumber();

  /* Set date sent + expire 1 year afterwards */
  header.date_entered=header.date_sent=time(NULL);

  if (filetype==0xddc) flags|=FILE_ARC;
  if (filetype!=0xfff) flags|=FILE_BINARY;

  /* Set binary or whatever */
  header.flags=flags;

  attr.action=17;
  attr.name=argv[5];
  os_file(&attr);
  attr_load=attr.loadaddr;
  attr_exec=attr.execaddr;

  /* Copy upload file to filepath file */
  if ((inf=fopen(argv[5],"rb"))==NULL) exit(50);

  if ((outf=fopen(file_pathname(header.file_location),"wb"))==NULL) exit(51);

  /* Copy file, poll every 10k */
  do
    {
    thisblock=fread(fbuffer,1,10240,inf);
    fwrite(fbuffer,1,thisblock,outf);
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

  if (put_message(&header,buffer)!=OK) exit(3);
  
  exit(0);
  return(0);
  }


void window_poll() /* Fudge for servcomm.c */
  {
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
    case 0: exit(10);
    case 2: exit(20);
    }

  if ((file.loadaddr & 0xFFF00000)==0xFFF00000)
    {
    return(file.start);
    }
  exit(30);
  }


char *file_pathname(int filenumber)
  {
  static char pathname[80];
  sprintf(pathname,"<ARCbbs$filedata>.%06d.%06d.%06d",((filenumber/2500)*2500),
                   ((filenumber/50)*50),filenumber);
  return(pathname);
  }
