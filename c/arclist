/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> SPARK lister (With help from Alan Glover)   <]
                  [> (SPARK lister will cope with .PAK's too)    <]
                  [> ZIP-lister                                  <]
                  [>                                             <]
Current version   [> 00.14                                       <]
Version date      [> 12-January-1993                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT © 1989-1993 by    <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

#include "include.h"         /* Standard library files */
#include "config.h"          /* System configuration */
#include "servmess.h"        /* Server message info */
#include "port.h"            /* Port driver functions */
#include "userlog.h"         /* Userlog format */
#include "parser.h"          /* Command parser */
#include "mail.h"            /* Mail format */
#include "portmisc.h"        /* Misc port commands */
#include "servcomm.h"        /* Misc server comm commands */
#include "mprintf.h"         /* Modem printf */
#include "arclist.h"         /* ARC file lister */

int  level,total_files,unpacked_size,spark_flag,file_length,
     recursive_list(void);
extern FILE *bbs_file[10];

extern user_block user;

extern void show_zip(void),show_lzh(void);

void show_arc(char *filename,char *sysname)
  {
  int byte1,byte2,byte3,byte4;

  level=total_files=unpacked_size=spark_flag=0;

  if ((bbs_file[7]=fopen(filename,"rb"))!=NULL)
    {
    fseek(bbs_file[7],0,SEEK_END);
    mprintf(1,"Archive listing of '%s' (%d bytes)\n",sysname,
                 file_length=ftell(bbs_file[7]));
    fseek(bbs_file[7],0,SEEK_SET);
    }
  else
    {
    mprintf(1,"Could not open file %s.\n",filename);
    return;
    }

  byte1=fgetc(bbs_file[7]);
  byte2=fgetc(bbs_file[7]);
  fgetc(bbs_file[7]);
  byte3=fgetc(bbs_file[7]);
  byte4=fgetc(bbs_file[7]);
  fseek(bbs_file[7],0,SEEK_SET);

  /* Check for PKzip */
  if (byte1=='P' && byte2=='K')
    {
    show_zip();    
    fclose(bbs_file[7]);
    return;
    }
    
  /* Check for LHarc */
  if (byte3=='l' && byte4=='h')
    {
    show_lzh();
    fclose(bbs_file[7]);
    return;
    }

  if (byte1!=0x1a)
    {
    fclose(bbs_file[7]);
    mprintf(1,"ERROR: file is not an archive\n");
    return;
    }

  mprintf(1,"{bfgbg wb}Filename                       {fg y}Size now {fg w}SF {fg y}  Length {fg w}Filetype{std}\n");
  if (user.termtype!=TER_ANSI) mprintf(1,"------------------------------ -------- -- -------- --------\n");

  if (recursive_list())
    {
    fclose(bbs_file[7]);
    return;
    }

  mprintf(1,"                           --- -------- -- --------\n");
  mprintf(1,"                     Total %3d %8d    %8d\n",total_files,file_length,unpacked_size);    

  if (spark_flag)
    {
    mprintf(1,"This file has been created with Spark: either decompress it with SPARKPLUG\nor dearc all sub-ARCs into subdirectories of the same name.\n");
    }

  fclose(bbs_file[7]);
  }

int recursive_list()
  {
  int exit_ptr,byte1,byte2,archimedes,a,b,t,pct;
  char dataspace[40];

  level++;

  label1:
  byte1=fgetc(bbs_file[7]);
  byte2=fgetc(bbs_file[7]);

  label2:
  archimedes=(byte2>0x80)?1:0;
  if (byte2==0 || byte2==0x80)
    {
    level--;
    return(0);
    }

  fread(dataspace,archimedes?39:27,1,bbs_file[7]);
  total_files++;

  if (level)
    {
    for(t=1;t<level;t++) mprintf(1,"   ");
    }

  a=(dataspace[13])+(dataspace[14]<<8)+(dataspace[15]<<16)+(dataspace[16]<<24);

  unpacked_size+=(b=(dataspace[23])+(dataspace[24]<<8)+
                     (dataspace[25]<<16)+(dataspace[26]<<24));
  mprintf(1,"{bfg c}%-12s",dataspace);
  for(t=0;t<(18-(level*3));t++) port_txw(' '); 
  if (b!=0) pct=((a*100)/b)%100; else pct=0;
  mprintf(1,"    {fg w}%8d {fg y}%2d {fg w}%8d {fg c}",a,pct,b);

  if (!archimedes)
    {
    mprintf(1,"MSdos{std}\n");
    }
  else
    {
    int filetype=(dataspace[28]+(dataspace[29]<<8)) & 0xfff;
    os_regset in;

    if (filetype!=0xddc)
      {
      in.r[0]=18;
      in.r[2]=filetype;
      os_swi(0x29,&in); /* OS_FSControl */
      in.r[4]=0; /* Check zero-terminated */
      mprintf(1,"%s{std}\n",(char*)&in.r[2]);
      }
    else
      {
      int byte,oldptr;

      mprintf(1,"Directory\n");
      unpacked_size-=b;
      exit_ptr=a+ftell(bbs_file[7]);
      spark_flag=1;
      if (recursive_list()) return(1);

      oldptr=ftell(bbs_file[7]);
      byte=fgetc(bbs_file[7]);
      fseek(bbs_file[7],oldptr,SEEK_SET);

      if (byte!=0x1a)
        {
        fseek(bbs_file[7],exit_ptr,SEEK_SET);
        }
      goto label1;
      }
    }

  if (file_length<(ftell(bbs_file[7])+a))
    {
    mprintf(1,"\n{bfg r}Error: beyond end of file (file truncated?){std}\n");
    return(1);
    }

  fseek(bbs_file[7],ftell(bbs_file[7])+a,SEEK_SET);
  byte1=fgetc(bbs_file[7]);
  if (byte1!=0x1a)
    {
    mprintf(1,"\n{bfg r}Error: No header found{std}\n");
    return(1);
    }

  byte2=fgetc(bbs_file[7]);
  goto label2;

  return(0); /* Keep compiler happy! */
  }

void show_zip()
  {
  int four,compsize,uncompsize,a,total_files=0,total_comp=0,total_uncomp=0;
  short fnlen,extralen;
  char filename[30];

  mprintf(1,"{bfgbg wb}Filename                       {fg y}Size now {fg w}SF {fg y}  Length{std}\n");
  if (user.termtype!=TER_ANSI) mprintf(1,"------------------------------ -------- -- --------\n");

  do
    {
    /* Local file header signature */
    fread(&four,4,1,bbs_file[7]);
    if (four!=0x04034b50)
      {  
      if (four==0x06054b50 || four==0x02014b50) break;

      /* Not a ZIPfile */
      mprintf(1,"{bfg r}Oops, corrupted ZIPfile!{std}\n");
      return;
      }   
                                       
    total_files++;

    /* Skip version needed/bitflags/compression method/date/time/crc */
    fseek(bbs_file[7],14,SEEK_CUR);

    /* Get compressed/decompressed sizes */
    fread(&compsize,4,1,bbs_file[7]);
    fread(&uncompsize,4,1,bbs_file[7]);
    total_comp+=compsize;
    total_uncomp+=uncompsize;

    /* Get filename length/extra field length */
    fread(&fnlen,2,1,bbs_file[7]);
    fread(&extralen,2,1,bbs_file[7]);
                      
    /* Get filename */
    if (fnlen>28)
      {
      /* Not a ZIPfile */
      mprintf(1,"{bfg r}Oops, corrupted ZIPfile!{std}\n");
      return;
      }   

    a=0;
    while(fnlen--) filename[a++]=fgetc(bbs_file[7]);
    filename[a]=0;        

    /* Skip extra field */
    fseek(bbs_file[7],extralen,SEEK_CUR);              

    /* Skip compressed archive bit */
    fseek(bbs_file[7],compsize,SEEK_CUR);          
                                  
    if (uncompsize!=0) a=((compsize*100)/uncompsize)%100; else a=0;
    mprintf(1,"{bfg c}%-30s {fg w}%8d {fg y}%2d {fg w}%8d\n",
            filename,compsize,a,uncompsize);
 
    }
  while(1);

  mprintf(1,"                           --- -------- -- --------\n");
  mprintf(1,"                     Total %3d %8d    %8d\n",total_files,
                                                total_comp,total_uncomp);    
  }

void show_lzh()
  {
  int compsize,uncompsize,a,total_files=0,total_comp=0,total_uncomp=0;
  char filename[30],fnlen;

  mprintf(1,"{bfgbg wb}Filename                       {fg y}Size now {fg w}SF {fg y}  Length{std}\n");
  if (user.termtype!=TER_ANSI) mprintf(1,"------------------------------ -------- -- --------\n");

  do
    { 
    char byte[5];
    
    if (fgetc(bbs_file[7])==0) break;
    fgetc(bbs_file[7]);

    fread(byte,5,1,bbs_file[7]);
    if (byte[0]!='-' || byte[1]!='l' || byte[2]!='h' || byte[4]!='-')
      {
      /* End of file I guess... */
      break;
      }
                                       
    total_files++;

    /* Get compressed/decompressed sizes */
    fread(&compsize,4,1,bbs_file[7]);
    fread(&uncompsize,4,1,bbs_file[7]);
    total_comp+=compsize;
    total_uncomp+=uncompsize;

    /* Get filename length/extra field length */
    fseek(bbs_file[7],6,SEEK_CUR);
    fread(&fnlen,1,1,bbs_file[7]);
                      
    /* Get filename */
    a=0;
    while(fnlen--) filename[a++]=fgetc(bbs_file[7]);
    filename[a]=0;              

    /* Skip compressed archive bit (& CRC) */
    fseek(bbs_file[7],compsize+2,SEEK_CUR);          
                                  
    if (uncompsize!=0) a=((compsize*100)/uncompsize)%100; else a=0;
    mprintf(1,"{bfg c}%-30s {fg w}%8d {fg y}%2d {fg w}%8d\n",
            filename,compsize,a,uncompsize);
 
    }
  while(1);

  mprintf(1,"                           --- -------- -- --------\n");
  mprintf(1,"                     Total %3d %8d    %8d\n",total_files,
                                                total_comp,total_uncomp);    
  }
