/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> BBS core header                             <]
Current version   [> 00.08                                       <]
Version date      [> 04-November-1992                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [> This source is COPYRIGHT © 1989/90/91/92 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

extern void window_poll(void),do_command(char*,int),mygets(char*,FILE*),
            file_download(int,int,char*,int,int),wimp_waitfor(int),
            conf_resign(unsigned char*,int),conf_join(unsigned char*,int),
            do_set(int,char*,int,int,int),flag_initialise(void),
            setup_mailheader(mail_block*,user_block*,int),_fclose(FILE**),
            get_item(char**,char*);
extern char *file_pathname(int),*fido_rules(char*),*fido_rules2(char*);             
extern int  conf_member(unsigned char*,int),xmodemrx(char*),
            xmodemtx(char*,int),seatx(char*),searx(char*),zmodemtx(char*),
            zmodemrx(char*),ymodemtx(char*),ymodemrx(char*),
            answer_routine(void),logon_routine(void),findflag_user(char*),
            findflag_msg(char*),findflag_file(char*);
extern FILE *bbs_file[10];

extern char tempbuffer[256],
            download_queuename[64][21],got_string[81],welcomename[80],
            welcomepassword[20],tempbuffer2[256],menupath[80],currentmenu[20],
            conferencename[61],filebasename[61];
extern int  got_type,portnumber,ourtask,baudrate,longpoll,bbs_currenttask,
            notimeout,msginterrupt,files_done,got_code3,got_code2,chatting,
            readnew,readnewfiles,savednr,usernumber,download_queue[64],
            download_queuesize[64],queue_length,beenchatting,noof_filed,
            arq,maxspeed_tx,maxspeed_rx,no_dcd,setbaud,enablecarrier,
            nicelogoff,online,loggedon,thislogon,filebase,conference,
            mail_event,scratchpad;
extern user_block user; 
extern user_list *online_users;
extern _address_arch ouraddress,fakenode;
extern char ourname[60];
extern clock_t logon_time,answer_time; 
extern time_t last_logon;
extern jmp_buf jmp_wimpexit,jmp_carrier,jmp_tomenu;
                       
#ifndef MAINMODULE
extern char *message_buffer;
#endif
