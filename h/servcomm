/*                  _____________________________________________
                  [>                                             <]
Project           [> ARCbbs                                      <]
Author            [> Hugo Fiennes                                <]
Date started      [> 04-April-1989                               <]
                  [>                                             <]
Module name       [> Server interface commands header            <]
Current version   [> 00.18                                       <]
Version date      [> 19-December-1991                            <]
State             [> Unfinished                                  <]
                  [>                                             <]
                  [>  This source is COPYRIGHT (c) 1989/90/91 by <]
                  [>      Hugo Fiennes of The Serial Port        <]
                  [>_____________________________________________<]
*/

extern int  get_user(user_block*),put_user(user_block*),delete_message(int),
            get_message(mail_block*,char*),get_messageh(mail_block*),
            hash_find(char*),hash_write(char*,int),find_free(void),
            get_filenumber(void),put_message(mail_block*,char*),
            put_messageh(mail_block*),read_filenumber(void),
            read_msgnumber(void),areamax(void),_mail_freeup(int),
            _mail_buffertodata(char*,int,int),
            _mail_buffertodatai(char*,int,int),
            put_messagei(mail_block*,char*),
            put_messagehi(mail_block*);

extern void window_poll(void),send_message(int,int,int,int,int,int),
            send_strmessage(int,int,int,int,int,char*),tolog(char*),
            read_area(int,mail_area*),errorlog(os_error*,char*),
            write_area(int,mail_area*),dumpfiles(void),hash_kill(char*);

extern int  got_data,got_code,portnumber,servertask;
