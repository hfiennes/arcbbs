/* -> ZCRC.h
*/
/*************************** -*- Mode: ANSI C -*- ***************************/
/* Title:       X/Zmodem CRC calculation routine header file                */
/* Author:      Hugo Fiennes (with credit to those listed below)            */
/*              (c)1987/1988/1989/1990 Hugo Fiennes                         */
/****************************************************************************/

/*
  Revision: 1.03
  Author  : Hugo Fiennes and those listed below
  Date    : 09-Mar-1990
  Source  : ARCterm disk
  State   :
*/

extern unsigned short crctab[256];
#define updcrc(cp, crc) ( crctab[((crc >> 8) & 255)] ^ (crc << 8) ^ cp)

extern unsigned long cr3tab[256];
#define Z_32UpdateCRC(c,crc) (cr3tab[((short) crc ^ c) & 0xff] ^ ((crc >> 8) & 0x00FFFFFF))
