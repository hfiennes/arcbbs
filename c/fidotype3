/*                _________________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Bundler                                     <]
Current version   [> 00.03                                       <]
Version date      [> 25-January-1990                             <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/1990 by  <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/
           
#include "include.h"   /* General includes */
#include "fidomail.h"  /* Fido mail structures */
#include "fidoa.h"     /* Assembler support */
#include "config.h"    /* ARCbbs config */
#include "mail.h"      /* Mail header */
#include "userlog.h"   /* User header */
#include "modcomm.h"   /* Module interface */
#include "servcomm.h"  /* Message-level interface */
         
/* Fido use varaibles */
struct { int area; struct _address_arch address;
         char name[20]; } fido_conflist[32];
int  fido_conflistlen;
struct _address_arch ouraddress;

extern char *message_buffer;

int fido_setup()
  {
  FILE *config; char line[80]; int end=0,count=0;

  if ((config=fopen("<ARCbbs$MiscData>.Fido.Setup","r"))==NULL)
    {
    return(ERROR);
    }
                       
  do
    {
    /* Read a line */
    if (fgets(line,80,config)!=NULL)
      {
      if (line[0]!=';')
        {
        if (strncmp("END",line,3)==0)
          {
          end=1; 
          }
        else
          {                             
          if (strncmp("OURNODE",line,7))
            {
            char tempname[20]; int temparea;
            struct _address_arch tempaddr;
       
            if (sscanf(line,"%s %d %d:%d/%d.%d",tempname,&temparea,
                       &tempaddr.zone,&tempaddr.net,&tempaddr.node,
                       &tempaddr.point)==6)
              {
              strcpy(fido_conflist[count].name,tempname);
              fido_conflist[count].area=temparea;
              fido_conflist[count].address=tempaddr;
              count++;
              }    
            else
              {
              werr(0,"Bad line in Fido.Setup : %s",line);
              }
            }
          else
            {
            if (sscanf(line,"OURNODE %d:%d/%d.%d",&ouraddress.zone,
                       &ouraddress.net,&ouraddress.node,&ouraddress.point)!=4)
              {
              werr(0,"Bad OURNODE setup : %s",line);
              fclose(config);
              return(ERROR);
              }
            }
          }
        }
      }
    }
  while(!feof(config) && end==0);
                      
  fido_conflistlen=count;
                   
  if (count==0)
    {
    werr(0,"No areas to process");
    return(NODATA);
    }

  /* Close the file */
  fclose(config);     

  /* Return */
  return(OK);
  }          

int fido_makebundle(struct _address_arch *to)
  {
  int  a,lastread=-1,end=0;
  char file[60],line[80],name[20],number[40],buffer[4096],*bufferend;
  FILE *temp,*bundle;
  struct _bundleheader *bundleh=(void*)buffer;
                    
  /* Open bundle */
  if ((bundle=fopen("<ARCbbs$misc>.Fido.BundleOUT","wb"))==NULL)
    {
    werr(0,"Can't open outbound bundle");
    return(ERROR);
    }

  /* Make filename */
  sprintf(name,"<ARCbbs$miscdata>.Fido.%d.%d.%d/%d",
                to->zone,to->net,to->node,to->point);

  /* Read node info */
  if ((temp=fopen(file,"r"))==NULL)
    {
    werr(0,"Can't open node info for %d:%d/%d.%d (%s)",to->zone,to->net,
                                                       to->node,to->point,
                                                       file);
    fclose(bundle);
    return(ERROR);
    }

  do
    {
    if (fgets(line,80,temp)!=NULL)
      {
      if (line[0]!=';')
        {              
        if (strncmp("END",line,3)!=NULL)
          {
          /* One of these will catch it! */
          sscanf(line,"Name %s",name);
          sscanf(line,"Number %s",number);
          sscanf(line,"Lastread %d",&lastread);
          }
        else
          {
          end=1;
          }
        }
      }
    }
  while(!feof(temp) && end==0);
           
  fclose(temp);

  if (name[0]==NULL)
    {
    werr(0,"No name found in %s",file);
    fclose(bundle);
    return(NODATA);
    }

  if (number[0]==NULL)
    {
    werr(0,"No number found in %s",file);
    fclose(bundle);
    return(NODATA);
    }

  if (lastread==-1)
    {
    werr(0,"No lastread found in %s",file);
    fclose(bundle);
    return(NODATA);
    }

  /* Make bundle header */
                        
  /* Fill in destination address */
  put_rint(to->zone        ,&bundleh->B_destination.zone );
  put_rint(to->net         ,&bundleh->B_destination.net  );
  put_rint(to->node        ,&bundleh->B_destination.node );
  put_rint(to->point       ,&bundleh->B_destination.point);
   
  /* Fill in origination address */
  put_rint(ouraddress.zone ,&bundleh->B_origination.zone );
  put_rint(ouraddress.net  ,&bundleh->B_origination.net  );
  put_rint(ouraddress.node ,&bundleh->B_origination.node );
  put_rint(ouraddress.point,&bundleh->B_origination.point);

  /* Fill in version */
  put_rint(3,&bundleh->B_version);

  /* Bundle creation time */
  put_rint((int)time(NULL),&bundleh->B_creationtime);

  /* Bundler version */
  put_rint(0 ,&bundleh->B_bundlermajor);
  put_rint(1 ,&bundleh->B_bundlerminor);
                  
  /* Password */
  put_zeropad("",9,(char*)bundleh->B_password);

  /* Bundler product code */
  strcpy((char*)bundleh->B_product,"ARM");

  /* Zero out FTSC stuff */
  bundleh->B_FTSC[0]=bundleh->B_FTSC[1]=
  bundleh->B_FTSC[2]=bundleh->B_FTSC[3]=0;

  /* Write it to file */
  if (fwrite(bundleh,sizeof(struct _bundleheader),1,bundle)!=1)
    {
    werr(0,"Can't write BundleHeader to file");
    fclose(bundle);
    return(ERROR);
    }
                                         
  /* If we're new to the area, don't export anything */
  if (lastread==0) lastread=mod_getlastlookup();

  /* Run through fido_conflist */
  for(a=0;a<fido_conflistlen;a++)
    { 
    /* Is this area to be shipped? */
    if (to->zone ==fido_conflist[a].address.zone  &&
        to->net  ==fido_conflist[a].address.net   &&
        to->node ==fido_conflist[a].address.node  &&
        to->point==fido_conflist[a].address.point)
      {
      mail_area echoarea;

      /* Yes, so read area */
      read_area(fido_conflist[a].area,&echoarea);

      if (echoarea.name[0]!=0 && echoarea.count!=0 &&
          echoarea.last_message>lastread)
        {
        mail_block header; int current_msg=lastread;
        struct _areaheader *aheader=(void*)buffer;
                     
        /* Ah! We DO have some messages to ship, make an
           'area header' */
                 
        /* Fill in header bitties */
        aheader->E_Version=3;
        aheader->E_PacketType=1;

        /* Fill in areaname */
        bufferend=put_shortstring(fido_conflist[a].name,(char*)&aheader->E_NameLength);

        /* Write to file */
        if (fwrite(buffer,(bufferend-buffer),1,bundle)!=1)
          {
          werr(0,"Can't write area header to file");
          fclose(bundle);
          return(ERROR);
          }

        do
          {
          int pointer,left;

          header.message_number=current_msg;
          get_message(&header,message_buffer);

          if (header.message_area==fido_conflist[a].area && header.status==1)
            {
            if ((header.flags&MSG_SYSOP)==0)
              {
              struct _messageheader *mheader=(void*)buffer;
              struct _text *mtext=(void*)buffer;
              char erasedchar;

              /* Make a 'message header' */
              mheader->M_Version=3;
              mheader->M_PacketType=2;
              
              /* Fill in destination address */
              put_rint(0,&mheader->M_destination.zone );
              put_rint(0,&mheader->M_destination.net  );
              put_rint(0,&mheader->M_destination.node );
              put_rint(0,&mheader->M_destination.point);
   
              /* Fill in origination address */
              put_rint(ouraddress.zone ,&mheader->M_origination.zone );
              put_rint(ouraddress.net  ,&mheader->M_origination.net  );
              put_rint(ouraddress.node ,&mheader->M_origination.node );
              put_rint(ouraddress.point,&mheader->M_origination.point);

              /* Creation time */
              put_rlong(header.date_sent,&mheader->M_CreationTime);
              
              /* Fido attributes */
              put_rint(0,&mheader->M_attributes);

              /* Little arcbbs fix */
              if (header.to_id==-1) strcpy(header.to,"All");

              /* From/to/subject length */
              mheader->M_FromLength=strlen(header.from);
              mheader->M_ToLength=strlen(header.to);
              mheader->M_SubjectLength=strlen(header.subject);

              /* Copy from/to/subject */
              bufferend=put_noterm(header.from,mheader->M_FromToSubject);
              bufferend=put_noterm(header.to,bufferend);
              bufferend=put_noterm(header.subject,bufferend);

              /* Write header to file */
              if (fwrite(buffer,(bufferend-buffer),1,bundle)!=1)
                {
                werr(0,"Can't write message header to bundle");
                fclose(bundle);
                return(ERROR);
                }

              /* Insert LF's */
              for (pointer=0;pointer<header.message_length;pointer++)
                {
                if (message_buffer[pointer]==0) message_buffer[pointer]=10;
                }
                  
              /* Write text packet(s) */
              left=header.message_length;
              pointer=0;

              do
                {
                /* Make text header */
                mtext->T_Version=3;
                mtext->T_PacketType=3;

                if (left>4096)
                  {
                  erasedchar=message_buffer[pointer+4096];
                  message_buffer[pointer+4096]=0;
                  }

                /* Copy msg text */
                bufferend=put_longstring((message_buffer+pointer),
                                         (char*)&mtext->T_ByteCount);

                /* Save that block */
                if (fwrite(buffer,(bufferend-buffer),1,bundle)!=1)
                  {
                  werr(0,"Can't write text block to bundle");
                  fclose(bundle);
                  return(ERROR);
                  }

                if (left>4096)
                  {                      
                  pointer+=4096; left-=4096;
                  message_buffer[pointer]=erasedchar;
                  }
                else
                  {
                  left=0;
                  }
                }
              while(left>0);
              }
            current_msg=header.message_forward;
            }
          else
            {
            current_msg+=1;
            } 
          }
        while(current_msg<=echoarea.last_message && current_msg!=-1);
        }
      }
    }
   
  /* Write bundle trailer */
  buffer[0]=3; buffer[1]=0;
  if (fwrite(buffer,2,1,bundle)!=1)
    {
    werr(0,"Can't write bundle trailer");
    fclose(bundle);
    return(ERROR);
    }             

  fclose(bundle);
  return(OK);
  }
