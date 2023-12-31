.. _mozilla_projects_nss_nss_sample_code_utiltiies_for_nss_samples:

Utilities for nss samples
=========================

.. _nss_sample_code_0_utilities.:

`NSS Sample Code 0: Utilities. <#nss_sample_code_0_utilities.>`__
-----------------------------------------------------------------

.. container::

   These utility functions are adapted from those found in the sectool library used by the NSS
   security tools and other NSS test applications. 

   It shows the following:

   -  Read DER from a file.
   -  Compile file size.
   -  Get seed From a noise gile.
   -  Generate random numbers.
   -  Get a module password.
   -  Extract the password from a text file.
   -  Print data as hexadecimal.

`util.h <#util.h>`__
~~~~~~~~~~~~~~~~~~~~

.. container::

   .. code:: brush:

      /* This Source Code Form is subject to the terms of the Mozilla Public
      * License, v. 2.0. If a copy of the MPL was not distributed with this
      * file, You can obtain one at https://mozilla.org/MPL/2.0/. */

      #ifndef _UTIL_H
      #define    _UTIL_H

      #include <prlog.h>
      #include <termios.h>
      #include <base64.h>
      #include <unistd.h>
      #include <sys/stat.h>
      #include "util.h"
      #include <prprf.h>
      #include <prerror.h>
      #include <nss.h>
      #include <pk11func.h>

      /*
      * These utility functions are adapted from those found in
      * the sectool library used by the NSS security tools and
      * other NSS test applications.
      */

      typedef struct {
          enum {
              PW_NONE = 0,        /* no password */
              PW_FROMFILE = 1,    /* password stored in a file */
              PW_PLAINTEXT = 2    /* plain-text password passed in  buffer */
              /* PW_EXTERNAL = 3  */
          } source;
          char *data;
          /* depending on source this can be the actual
           * password or the file to read it from
           */
      } secuPWData;

      /*
       * PrintAsAscii
       */
      extern void
      PrintAsAscii(PRFileDesc* out, const unsigned char *data, unsigned int len);

      /*
       * PrintAsHex
       */
      extern void
      PrintAsHex(PRFileDesc* out, const unsigned char *data, unsigned int len);

      /*
       * GetDigit
       */
      extern int
      GetDigit(char c);

      /*
       * HexToBuf
       */
      extern int
      HexToBuf(unsigned char *inString, SECItem *outbuf, PRBool isHexData);

      /*
       * FileToItem
       */
      extern SECStatus
      FileToItem(SECItem *dst, PRFileDesc *src);

      /*
       * CheckPassword
       */
      extern PRBool
      CheckPassword(char *cp);

      /*
       * GetPassword
       */
      extern char *
      GetPassword(FILE   *input,
                  FILE   *output,
                  char   *prompt,
                  PRBool (*ok)(char *));

      /*
       * FilePasswd extracts the password from a text file
       *
       * Storing passwords is often used with server environments
       * where prompting the user for a password or requiring it
       * to be entered in the commnd line is not a feasible option.
       *
       * This function supports password extraction from files with
       * multipe passwords, one for each token. In the single password
       * case a line would just have the passord whereas in the multi-
       * password variant they could be of the form
       *
       * token_1_name:its_password
       * token_2_name:its_password
       *
       */
      extern char *
      FilePasswd(PK11SlotInfo *
                 slot, PRBool retry, void *arg);

      /*
       * GetModulePassword
       */
      extern char *
      GetModulePassword(PK11SlotInfo *slot,
                        int          retry,
                        void         *pwdata);

      /*
       * GenerateRandom
       */
      extern SECStatus
      GenerateRandom(unsigned char *rbuf,
                     int           rsize);

      /*
       * FileToItem
       */
      extern SECStatus
      FileToItem(SECItem    *dst,
                 PRFileDesc *src);

      /*
       * SeedFromNoiseFile
       */
      extern SECStatus
      SeedFromNoiseFile(const char *noiseFileName);

      /*
       * FileSize
       */
      extern long
      FileSize(const char* filename);

      /*
       * ReadDERFromFile
       */
      extern SECStatus
      ReadDERFromFile(SECItem *der, const char *inFileName, PRBool ascii);

      #endif /* _UTIL_H */

`Util.c <#util.c>`__
~~~~~~~~~~~~~~~~~~~~

.. container::

   .. code:: brush:

      /* This Source Code Form is subject to the terms of the Mozilla Public
       * License, v. 2.0. If a copy of the MPL was not distributed with this
       * file, You can obtain one at https://mozilla.org/MPL/2.0/. */

      #include "util.h"

      /*
       * These utility functions are adapted from those found in
       * the sectool library used by the NSS security tools and
       * other NSS test applications.
       */

      /*
       * Newline
       */
      static void
      Newline(PRFileDesc* out)
      {
          PR_fprintf(out, "\n");
      }

      /*
       * PrintAsAscii
       */
      void
      PrintAsAscii(PRFileDesc* out, const unsigned char *data, unsigned int len)
      {
          char *b64Data = NULL;

          b64Data = BTOA_DataToAscii(data, len);
          PR_fprintf(out, "%s", b64Data);
          PR_fprintf(out, "\n");
          if (b64Data) {
              PORT_Free(b64Data);
          }
      }

      /*
       * PrintAsHex
       */
      void
      PrintAsHex(PRFileDesc* out, const unsigned char *data, unsigned int len)
      {
          unsigned i;
          int column;
          unsigned int limit = 15;
          unsigned int level  = 1;

          column = level;
          if (!len) {
              PR_fprintf(out, "(empty)\n");
              return;
          }

          for (i = 0; i < len; i++) {
              if (i != len - 1) {
                  PR_fprintf(out, "%02x:", data[i]);
                  column += 3;
              } else {
                  PR_fprintf(out, "%02x", data[i]);
                  column += 2;
                  break;
              }
              if (column > 76 || (i % 16 == limit)) {
                  Newline(out);
                  column = level;
                  limit = i % 16;
              }
          }
          if (column != level) {
              Newline(out);
          }
      }

      /*
       * GetDigit
       */
      int
      GetDigit(char c)
      {
          if (c == 0) {
              return -1;
          }
          if (c <= '9' && c >= '0') {
              return c - '0';
          }
          if (c <= 'f' && c >= 'a') {
              return c - 'a' + 0xa;
          }
          if (c <= 'F' && c >= 'A') {
              return c - 'A' + 0xa;
          }
          return -1;
      }

      /*
       * HexToBuf
       */
      int
      HexToBuf(unsigned char *inString, SECItem *outbuf, PRBool isHexData)
      {
          int len = strlen((const char *)inString);
          int outLen = len+1/2;
          int trueLen = 0;
          int digit1, digit2;

          outbuf->data = isHexData
              ? PORT_Alloc(outLen)
              : PORT_Alloc(len);
          if (!outbuf->data) {
              return -1;
          }
          if (isHexData) {
              while (*inString) {
                   if ((*inString == '\n') || (*inString == ':')) {
                       inString++;
                       continue;
                   }
                   digit1 = GetDigit(*inString++);
                   digit2 = GetDigit(*inString++);
                   if ((digit1 == -1) || (digit2 == -1)) {
                       PORT_Free(outbuf->data);
                       outbuf->data = NULL;
                       return -1;
                   }
                   outbuf->data[trueLen++] = digit1 << 4 | digit2;
              }
          } else {
              while (*inString) {
                  if (*inString == '\n') {
                      inString++;
                      continue;
                  }
                  outbuf->data[trueLen++] = *inString++;
              }
              outbuf->data[trueLen] = '\0';
              trueLen = trueLen-1;
          }
          outbuf->len = trueLen;
          return 0;
      }

      /*
       * FileToItem
       */
      SECStatus
      FileToItem(SECItem *dst, PRFileDesc *src)
      {
          PRFileInfo info;
          PRInt32 numBytes;
          PRStatus prStatus;

          prStatus = PR_GetOpenFileInfo(src, &info);

          if (prStatus != PR_SUCCESS) {
              return SECFailure;
          }

          dst->data = 0;
          if (SECITEM_AllocItem(NULL, dst, info.size)) {
              numBytes = PR_Read(src, dst->data, info.size);
              if (numBytes == info.size) {
                  return SECSuccess;
              }
          }
          SECITEM_FreeItem(dst, PR_FALSE);
          dst->data = NULL;
          return SECFailure;
      }

      /*
       * echoOff
       */
      static void echoOff(int fd)
      {
         if (isatty(fd)) {
             struct termios tio;
             tcgetattr(fd, &tio);
             tio.c_lflag &= ~ECHO;
             tcsetattr(fd, TCSAFLUSH, &tio);
         }
      }

      /*
       * echoOn
       */
      static void echoOn(int fd)
      {
         if (isatty(fd)) {
             struct termios tio;
             tcgetattr(fd, &tio);
             tio.c_lflag |= ECHO;
             tcsetattr(fd, TCSAFLUSH, &tio);
         }
      }

      /*
       * CheckPassword
       */
      PRBool CheckPassword(char *cp)
      {
          int len;
          char *end;
          len = PORT_Strlen(cp);
          if (len < 8) {
              return PR_FALSE;
          }
          end = cp + len;
          while (cp < end) {
              unsigned char ch = *cp++;
              if (!((ch >= 'A') && (ch <= 'Z')) &&
                  !((ch >= 'a') && (ch <= 'z'))) {
                  return PR_TRUE;
              }
         }
         return PR_FALSE;
      }

      /*
       * GetPassword
       */
      char* GetPassword(FILE *input, FILE *output, char *prompt,
                        PRBool (*ok)(char *))
      {
          char phrase[200] = {'\0'};
          int infd         = fileno(input);
          int isTTY        = isatty(infd);

          for (;;) {
              /* Prompt for password */
              if (isTTY) {
                  fprintf(output, "%s", prompt);
                  fflush (output);
                  echoOff(infd);
              }
              fgets(phrase, sizeof(phrase), input);
              if (isTTY) {
                  fprintf(output, "\n");
                  echoOn(infd);
              }
              /* stomp on newline */
              phrase[PORT_Strlen(phrase)-1] = 0;
              /* Validate password */
              if (!(*ok)(phrase)) {
                  if (!isTTY) return 0;
                  fprintf(output, "Password must be at least 8 characters long with one or more\n");
                  fprintf(output, "non-alphabetic characters\n");
                  continue;
              }
              return (char*) PORT_Strdup(phrase);
          }
      }

      /*
       * FilePasswd extracts the password from a text file
       *
       * Storing passwords is often used with server environments
       * where prompting the user for a password or requiring it
       * to be entered in the commnd line is not a feasible option.
       *
       * This function supports password extraction from files with
       * multipe passwords, one for each token. In the single password
       * case a line would just have the passord whereas in the multi-
       * password variant they could be of the form
       *
       * token_1_name:its_password
       * token_2_name:its_password
       *
       */
      char *
      FilePasswd(PK11SlotInfo *slot, PRBool retry, void *arg)
      {
          char* phrases, *phrase;
          PRFileDesc *fd;
          PRInt32 nb;
          char *pwFile = arg;
          int i;
          const long maxPwdFileSize = 4096;
          char* tokenName = NULL;
          int tokenLen = 0;

          if (!pwFile)
              return 0;

          if (retry) {
              return 0;  /* no good retrying - the files contents will be the same */
          }

          phrases = PORT_ZAlloc(maxPwdFileSize);

          if (!phrases) {
              return 0; /* out of memory */
          }

          fd = PR_Open(pwFile, PR_RDONLY, 0);
          if (!fd) {
              fprintf(stderr, "No password file \"%s\" exists.\n", pwFile);
              PORT_Free(phrases);
              return NULL;
          }

          nb = PR_Read(fd, phrases, maxPwdFileSize);

          PR_Close(fd);

          if (nb == 0) {
              fprintf(stderr,"password file contains no data\n");
              PORT_Free(phrases);
              return NULL;
          }

          if (slot) {
              tokenName = PK11_GetTokenName(slot);
              if (tokenName) {
                  tokenLen = PORT_Strlen(tokenName);
              }
          }
          i = 0;
          do {
              int startphrase = i;
              int phraseLen;

              /* handle the Windows EOL case */
              while (phrases[i] != '\r' && phrases[i] != '\n' && i < nb) i++;

              /* terminate passphrase */
              phrases[i++] = '\0';
              /* clean up any EOL before the start of the next passphrase */
              while ( (i<nb) && (phrases[i] == '\r' || phrases[i] == '\n')) {
                  phrases[i++] = '\0';
              }
              /* now analyze the current passphrase */
              phrase = &phrases[startphrase];
              if (!tokenName)
                  break;
              if (PORT_Strncmp(phrase, tokenName, tokenLen)) continue;
              phraseLen = PORT_Strlen(phrase);
              if (phraseLen < (tokenLen+1)) continue;
              if (phrase[tokenLen] != ':') continue;
              phrase = &phrase[tokenLen+1];
              break;

          } while (i<nb);

          phrase = PORT_Strdup((char*)phrase);
          PORT_Free(phrases);
          return phrase;
      }

      /*
       * GetModulePassword
       */
      char* GetModulePassword(PK11SlotInfo *slot, int retry, void *arg)
      {
          char prompt[255];
          secuPWData *pwdata = (secuPWData *)arg;
          char *pw;

          if (pwdata == NULL) {
              return NULL;
          }

          if (retry && pwdata->source != PW_NONE) {
              PR_fprintf(PR_STDERR, "Incorrect password/PIN entered.\n");
              return NULL;
          }

          switch (pwdata->source) {
          case PW_NONE:
              sprintf(prompt, "Enter Password or Pin for \"%s\":",
                      PK11_GetTokenName(slot));
              return GetPassword(stdin, stdout, prompt, CheckPassword);
          case PW_FROMFILE:
              pw = FilePasswd(slot, retry, pwdata->data);
              pwdata->source = PW_PLAINTEXT;
              pwdata->data = PL_strdup(pw);
              return pw;
          case PW_PLAINTEXT:
              return PL_strdup(pwdata->data);
          default:
              break;
          }
          PR_fprintf(PR_STDERR, "Password check failed:  No password found.\n");
          return NULL;
      }

      /*
       * GenerateRandom
       */
      SECStatus
      GenerateRandom(unsigned char *rbuf, int rsize)
      {
          char meter[] = {
                         "\r|                                |" };
          int            fd,  count;
          int            c;
          SECStatus      rv                  = SECSuccess;
          cc_t           orig_cc_min;
          cc_t           orig_cc_time;
          tcflag_t       orig_lflag;
          struct termios tio;

          fprintf(stderr, "To generate random numbers, "
                  "continue typing until the progress meter is full:\n\n");
          fprintf(stderr, "%s", meter);
          fprintf(stderr, "\r|");

          /* turn off echo on stdin & return on 1 char instead of NL */
          fd = fileno(stdin);

          tcgetattr(fd, &tio);
          orig_lflag = tio.c_lflag;
          orig_cc_min = tio.c_cc[VMIN];
          orig_cc_time = tio.c_cc[VTIME];
          tio.c_lflag &= ~ECHO;
          tio.c_lflag &= ~ICANON;
          tio.c_cc[VMIN] = 1;
          tio.c_cc[VTIME] = 0;
          tcsetattr(fd, TCSAFLUSH, &tio);
          /* Get random noise from keyboard strokes */
          count = 0;
          while (count < rsize) {
              c = getc(stdin);
              if (c == EOF) {
                  rv = SECFailure;
                  break;
              }
              *(rbuf + count) = c;
              if (count == 0 || c != *(rbuf + count -1)) {
                  count++;
                  fprintf(stderr, "*");
              }
          }
          rbuf[count] = '\0';

          fprintf(stderr, "\n\nFinished.  Press enter to continue: ");
          while ((c = getc(stdin)) != '\n' && c != EOF)
              ;
          if (c == EOF)
              rv = SECFailure;
          fprintf(stderr, "\n");

          /* set back termio the way it was */
          tio.c_lflag = orig_lflag;
          tio.c_cc[VMIN] = orig_cc_min;
          tio.c_cc[VTIME] = orig_cc_time;
          tcsetattr(fd, TCSAFLUSH, &tio);
          return rv;
      }

      /*
       * SeedFromNoiseFile
       */
      SECStatus
      SeedFromNoiseFile(const char *noiseFileName)
      {
          char buf[2048];
          PRFileDesc *fd;
          PRInt32 count;

          fd = PR_Open(noiseFileName, PR_RDONLY, 0);
          if (!fd) {
              fprintf(stderr, "failed to open noise file.");
              return SECFailure;
          }

          do {
              count = PR_Read(fd,buf,sizeof(buf));
              if (count > 0) {
                  PK11_RandomUpdate(buf,count);
              }
          } while (count > 0);

          PR_Close(fd);
          return SECSuccess;
      }

      /*
       * FileSize
       */
      long FileSize(const char* filename)
      {
          struct stat stbuf;
          stat(filename, &stbuf);
          return stbuf.st_size;
      }

      /*
       *  ReadDERFromFile
       */
      SECStatus
      ReadDERFromFile(SECItem *der, const char *inFileName, PRBool ascii)
      {
          SECStatus rv       = SECSuccess;
          PRFileDesc *inFile = NULL;

          inFile = PR_Open(inFileName, PR_RDONLY, 0);
          if (!inFile) {
              PR_fprintf(PR_STDERR, "Failed to open file \"%s\" (%ld, %ld).\n",
                         inFileName, PR_GetError(), PR_GetOSError());
              rv = SECFailure;
              goto cleanup;
          }

          if (ascii) {
              /* First convert ascii to binary */
              SECItem filedata;
              char *asc, *body;

              /* Read in ascii data */
              rv = FileToItem(&filedata, inFile);
              asc = (char *)filedata.data;
              if (!asc) {
                  PR_fprintf(PR_STDERR, "unable to read data from input file\n");
                  rv = SECFailure;
                  goto cleanup;
              }

              /* check for headers and trailers and remove them */
              if ((body = strstr(asc, "-----BEGIN")) != NULL) {
                  char *trailer = NULL;
                  asc = body;
                  body = PORT_Strchr(body, '\n');
                  if (!body)
                      body = PORT_Strchr(asc, '\r'); /* maybe this is a MAC file */
                  if (body)
                      trailer = strstr(++body, "-----END");
                  if (trailer != NULL) {
                      *trailer = '\0';
                  } else {
                      PR_fprintf(PR_STDERR,  "input has header but no trailer\n");
                      PORT_Free(filedata.data);
                      rv = SECFailure;
                      goto cleanup;
                  }
              } else {
                  body = asc;
              }

              /* Convert to binary */
              rv = ATOB_ConvertAsciiToItem(der, body);
              if (rv) {
                  PR_fprintf(PR_STDERR,  "error converting ascii to binary %s\n",
                             PORT_GetError());
                  PORT_Free(filedata.data);
                  rv = SECFailure;
                  goto cleanup;
              }

              PORT_Free(filedata.data);
          } else {
              /* Read in binary der */
              rv = FileToItem(der, inFile);
              if (rv) {
                  PR_fprintf(PR_STDERR, "error converting der \n");
                  rv = SECFailure;
              }
          }
      cleanup:
          if (inFile) {
              PR_Close(inFile);
          }
          return rv;
      }