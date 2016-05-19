#!/usr/bin/env python
#

"""
Process Java logs and extracts SSL keys.

Pass -Djavax.net.debug=ssl,keygen when starting Java application.
Pipe output through this script.
Output is a (Pre)-Master-Secret log output for Wireshark's consumption.

Input example:
...
*** ServerHelloDone
*** ClientKeyExchange, RSA PreMasterSecret, TLSv1.2
main, WRITE: TLSv1.2 Handshake, length = 262
SESSION KEYGEN:
PreMaster Secret:
0000: 03 03 B1 49 3C F1 16 FF   3D 33 D0 D5 F2 46 CB E1  ...I<...=3...F..
0010: 68 BF DF 26 B7 74 EA B5   7D E7 8D E3 FF BB 8B 90  h..&.t..........
0020: E8 03 A0 CC 5F F3 5D BA   BD BB 42 E6 05 C6 36 3B  ...._.]...B...6;
CONNECTION KEYGEN:
Client Nonce:
0000: 57 3D 06 80 5D 59 A4 12   D2 24 05 56 9E 12 73 61  W=..]Y...$.V..sa
0010: 59 29 52 A5 07 1A 8E 43   1F E1 93 BF D4 50 C6 86  Y)R....C.....P..
Server Nonce:
0000: 57 3D 06 80 B0 E3 8F 7A   1E 12 DE D3 B7 65 AC 29  W=.....z.....e.)
0010: 44 66 C9 65 FD A8 53 B0   CA 7A CE D0 16 BD 5E F0  Df.e..S..z....^.
Master Secret:
0000: BA 05 35 3B EF B6 54 7B   6E F4 93 5A DE E7 CD 12  ..5;..T.n..Z....
0010: 43 E6 39 C9 FA A5 03 E0   1C 01 6D 67 2A 62 2E B0  C.9.......mg*b..
0020: 86 3C E0 5F 1C 4C B4 DE   22 38 7B BF F3 62 64 70  .<._.L.."8...bdp
Client MAC write Secret:
0000: 65 D5 C0 FF 59 E0 29 B4   54 98 3B 59 93 25 5D F8  e...Y.).T.;Y.%].
0010: D9 63 AB 91                                        .c..
Server MAC write Secret:
0000: 1A 87 29 D9 57 92 80 8E   73 B6 88 7E C8 5A C3 11  ..).W...s....Z..
0010: 0F E6 E6 40                                        ...@
Client write key:
0000: 72 2C 10 1D 75 F8 D8 35   86 99 CE 66 36 3D 63 A5  r,..u..5...f6=c.
0010: C1 01 52 4D EC 13 1E 91                            ..RM....
Server write key:
0000: CC 91 06 BB F0 E2 D4 64   48 68 26 DE D8 B7 0B EA  .......dHh&.....
0010: B7 B6 B0 1A 61 0A A6 C8                            ....a...
... no IV derived for this protocol
main, WRITE: TLSv1.2 Change Cipher Spec, length = 1
*** Finished
...
***
CONNECTION KEYGEN:
Client Nonce:
0000: 57 3D 1A E1 B4 A1 F2 6C   D1 5F D4 BB DF 90 7B 88  W=.....l._......
0010: DD 52 57 6F 76 E9 5E D0   75 91 03 FB 19 31 8A 1B  .RWov.^.u....1..
Server Nonce:
0000: 57 3D 1A E1 15 25 D8 7C   B1 1F DD E5 C1 D1 F2 75  W=...%.........u
0010: 21 7E 92 B9 3D F6 8A 87   5F 46 CE 61 F8 17 25 BD  !...=..._F.a..%.
Master Secret:
0000: D9 17 9B 11 2F B8 9D 0D   0A 42 C7 34 0E 4A 0B 4B  ..../....B.4.J.K
0010: 2F 94 F1 CC C0 63 93 19   17 03 D4 9A E8 03 6F D8  /....c........o.
0020: D2 66 CF D1 4E E2 8E 3F   E6 9D 41 3D E9 26 CD EB  .f..N..?..A=.&..
Client MAC write Secret:
0000: 03 D3 D2 48 A8 5F FD 40   B5 86 30 5A 9B 0A 13 89  ...H._.@..0Z....
0010: EC 78 B0 B6 A1 0E B7 0E   82 7B 50 B7 15 10 EA 60  .x........P....`
Server MAC write Secret:
0000: E7 E4 BC 3E 87 12 D7 2B   C6 47 2F 62 4F 49 FF 55  ...>...+.G/bOI.U
0010: 41 23 69 B2 13 B8 8B 24   0E 75 B0 06 FB 13 91 B4  A#i....$.u......
Client write key:
0000: 30 72 FA 01 95 80 88 D1   D8 99 A3 5C 6E A5 2B 0D  0r.........\n.+.
Server write key:
0000: 0D 7C D7 2E 68 2A 31 E2   C6 8E A6 B9 85 00 FB E5  ....h*1.........
... no IV derived for this protocol
...

Output example:
CLIENT_RANDOM 573D06805D59A412D22405569E127361592952A5071A8E431FE193BFD450C686 BA05353BEFB6547B6EF4935ADEE7CD1243E639C9FAA503E01C016D672A622EB0863CE05F1C4CB4DE22387BBFF3626470
CLIENT_RANDOM 573D1AE1B4A1F26CD15FD4BBDF907B88DD52576F76E95ED0759103FB19318A1B D9179B112FB89D0D0A42C7340E4A0B4B2F94F1CCC06393191703D49AE8036FD8D266CFD14EE28E3FE69D413DE926CDEB

This should show output example:
cat extract-pre-master-secret | ./extract-pre-master-secret
"""

import sys 
import re

def extract_data_from_line(line):
  m = re.match('\d+:([ 0-9A-F]{51}) .*', line)
  if m:
    return m.group(1).replace(' ', '')
  else:
    raise line


def main():
  parsing_mastersecret_line = None
  parsing_clientnonce_line = None
  for line in sys.stdin:
    if parsing_mastersecret_line:
      parsing_mastersecret_line += 1
    if parsing_clientnonce_line:
      parsing_clientnonce_line += 1

    if line == 'Client Nonce:\n':    
      parsing_clientnonce_line = 1
      cn = ""
    if 2 <= parsing_clientnonce_line <= 3:
      cn = cn + extract_data_from_line(line)

    if line == 'Master Secret:\n':
      parsing_mastersecret_line = 1
      ms = ""
    if 2 <= parsing_mastersecret_line <= 4:
      ms = ms + extract_data_from_line(line)
    
    if 5 == parsing_mastersecret_line:
      print 'CLIENT_RANDOM', cn, ms


if __name__=='__main__':
    sys.exit(main())
