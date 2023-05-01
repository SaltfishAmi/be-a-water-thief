# Be a water thief: Hacking the dormitory water boiler in a top 20 university in China

**The research was completed in 2015. The vulnerable system was removed in 2016 without regard to the existence of this research. The paper was written in 2018 when I was still an undergraduate in medical school with no CS background, and translated into English in 2023. Don't laugh at my stupidity.**

## Background

In China, tap water is generally unsafe to drink directly, and Chinese people have developed the habit of boiling water before drinking it since the 1950s. That is why in most university dormitories, the management team either allows kettles in individual rooms or provides convenient access to public boilers.

I was a student from out of the province, was adjusted to the most miserable major in the university, Clinical Medicine (in China, it's not a big money maker), and spent most of my time on the oldest and shabbiest campus. The dormitory buildings were old, the maintenance was poor, and the electrical circuits were worn out. It was rumored that one year before we came, there was a fire incident caused by overloaded electric cables. Therefore, kettles were strictly forbidden in our rooms. The way we obtained boiled water was as follows. Unless otherwise stated, any currency in this paper is the Chinese Yuan Renminbi (CNY).

1. Buy a Water Card from the management staff. It is a standalone device without connection to our student ID card systems. Accessing boilers is its only functionality.

2. Grab your bottle and card and head to any boiler in your building. Put the bottle in place, swipe the card briefly, and then take it away.

3. The boiler makes a beep, displays your current balance, and starts pouring boiled water.

4. The displayed balance decreases as the water come out at a rate of roughly one cent per 3 seconds.

   a. After the balance decreased by 16 cents, your average-sized bottle was filled nicely. The water stops, and the display goes out after a few seconds.

   b. If you do not need as much water, you swipe the card once again. The water stream stops with a brief beep, and the display goes out after a few seconds.

   c. If your card balance was less than 16 cents, the water stops when it reaches zero, and the display goes out after a few seconds.

5. If you need more balance, you can only recharge it with cash at the management staff's desk.

After a simple analysis of the system, I concluded that it was perfectly possible to crack it. There were a few reasons:

1. Lack of network connectivity

   This was a very rudimentary system. The appearance of the water cards was very rough, and there was no association between the card and your identity. By inspecting the pipes and cables behind the boilers, I found no network cable, and the machine itself absolutely did not look capable of making wireless connections. Therefore, it is very likely that the system was not network connected at all, and it was possible to crack the system and tamper with data with minimal risk.

2. Data storage

   By analyzing the steps of getting water, I assumed that the data in the cards were stored and modified in the following manner:

   a. Upon each card read, the machine inspects the available balance, puts it into temporary storage, and displays it on the screen.

      i. If it was no less than 16 cents, deduct 16 cents and write to the card.

      ii. If it was less than 16 cents, write zero to the card.

   b. The balance in the temporary storage is decreased over time.

      i. If you swipe the card again, the balance in the card is overwritten by that in the temporary storage.

      ii. If not, then the balance in the card has already been nicely reduced by 16 cents.

   Considering how rudimentary the system looked, I would never believe the card was a cutting-edge super mass storage device that logs every transaction. It was very likely that it simply stored the current balance in numbers.

3. Existing technology

   A quick web told me that the most popular low-cost NFC solution in China at the time was called the M1 card, or S50 card, and it could be cracked by some techniques I could not understand.

   I did not remember how I found out that the water cards were indeed M1 cards.

4. Equipment

   It was possible to purchase a card reader and writer from Taobao (the Chinese version of Amazon) for less than 25 US dollars. I got one for myself and downloaded all the necessary software for free while waiting for the courier.

## Beginning

On that day, my courier finally arrived. It was a very easy-to-understand device: a USB-A male connector, a flat panel with the letters "NFC" printed, and an LED indicator. Plugging it into my laptop, the card reader made a very easy-to-understand beep, corresponding with the lower-pitched and smoother beep emitted by the boiler. Excitedly, I put my water card onto it. The LED indicator turned green, and the card reader made a beep again. This meant that my card was recognized. First step success.

Then I dug out the cracking software `mfoc.exe`, opened a command prompt window, and entered the command:

```
mfoc -O temp.dmp
```

to crack the current card with default configuration, and save the dump file to `temp.dmp`.

The process survived shortly before running into a fatal error. I took the card away and put it on the reader again. No beeps. The reader entered the vegetable state. A web search told me that NFC card readers are extremely unstable and almost unusable on Windows. I had to use Linux to make it function correctly, and Kali Linux was the most suitable distro for this task. So I downloaded its LiveCD ISO with the desktop option, burnt it onto a DVD, and rebooted my laptop. Yeah, I still use optical drives. I'm old.

## Progression

It was 3 a.m. in the winter. I sighed at the miserable speed of booting by CDROM and entered the desktop of Kali Linux 2.0. From the Applications menu at the top-left corner, I selected `Wireless Attack`, `RFID & NFC tools`, then `mfoc`. A terminal popped out with directions for using the tool. I typed the same command as I tried in Windows and successfully hacked my water card (UID: 1E83471B). I opened the dump with a hex editor.

The card contained 16 sectors and could store 1,024 bytes of data. But in this dump, except for the first sector with the UID, only the seventh sector (offset: 0x0180) contained data. The data was:

```
01 33 98 00 01 A4 0F FF FF FF FF FF FF FF FF FF
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
1F B5 04 72 33 90 FF 07 80 69 FF FF FF FF FF FF
```

By reading the technical specifications of the card, I learned that the last line was the encryption key (`KeyA` and `KeyB`) for that sector. Therefore any actual data must lie in the non-empty first line of the sector. Specifically, the first half of the first line:

```
01 33 98 00 01 A4 0F FF
```

When I attacked this card, the balance was CN¥4.20 or 420 cents. My calculator told me that 420 cents were 0x01a4 cents in hex. That was too accurate to be a coincidence. So I noted my guess: card balance, offset 0x0184, size 2 bytes, big-endian. By the way, how many of you have ever used floating point numbers to store money? C'mon, don't lie; I know you have.

I then fetched half a bottle of water, and the balance went down to CN¥4.10. I reread the card (there was no need to crack it because the encryption keys were revealed after the last crack was completed) and discovered that the data area was like this.

```
01 33 98 00 01 9A 31 FF
```

The first four and last byte were confirmed to be fixed after testing with some other water cards. Obviously, 0x019a equals 410. But how about the following byte, 0x0f and 0x31? I tried to ignore them, only altered the two-byte card balance, and got a merciless error just as I expected. I guessed that they must be some kind of data integrity checking.

## Climax

After drinking a lot of water, I got plentiful test data as follows:

```
01 A4 0F
01 A2 09
01 9F 34
01 9A 31
01 98 33
01 8D 26
01 86 2D
01 80 2B
01 70 DB
01 2C 87
01 2B 80
01 1C B7
00 FE 54
00 F0 5A
00 ED 47
00 E3 49
00 D1 7B
00 00 AA
```

My first hypothesis was that the checksum byte is calculated from UID and card balance with some logical operator like XOR. But further observation revealed that there was actually nothing to do with UID.

Excerpt from my research scratch paper:

```
x = original bits, y = where the bit needs to be flipped
00000000 xxxxxxxx yxyxyxyx
00000001 xxxxxxxx yxyxyxyy
00000010 xxxxxxxx yxyxyxxx ? 02 00 a8
00000010 xxxxxxxx yxyxyyxx ? 02 00 ac (FAIL)
```

In this scratch paper, the first eight bits of each line were the higher bits of the card balance (e.g., 0x01 in 0x01a4) in binary; the middle eight characters were the lower bits (e.g., 0xa4 in 0x01a4) in binary. The last eight characters were the method to derive the checksum from the balance. For example, given data `01 A4 0F`, the balance can be written in binary as `00000001 10100100`. Mask it with the pattern `yxyxyxyy` (i.e., XOR with `10101011`), and you can get `00001111`, i.e., 0x0f. I called this pattern the "Checksum Generation Pattern (CGP)". The base case CGP should be `10101010`. XOR it with the higher bits, and then XOR again with lower bits to get the final checksum byte. For many computer scientists, XOR is an operator as simple and easy to think of as the basic plus and minus operators. But my 2015 self was just a poor medical student with no formal CS background, and it took me great effort to come up with a hell lot of patterns and bullshit, unaware of the more straightforward and elegant way.

In shitty code written by my 2015 self, who didn't think of XOR at all:

```c++
int flipByCgp(int operand, int CGP)
{
   bitset<8> opr(operand);
   bitset<8> cgp(CGP);
   for(inti = 0; i < 8; i++) {
       if(cgp[i] == 1) opr.flip(i);
    }
   return opr.to_ulong();
}

int genCgpByHigher(int higher)
{
   return flipByCgp(0xAA, higher);
}

int getChecksumBySeperateBits(int lower, int higher)
{
   return flipByCgp(lower, genCgpByHigher(higher));
}

int getChecksum(int operand)
{
   operand = operand % 0x10000;
   return getChecksumBySeperateBits(operand % 0x0100, operand / 0x0100);
}
```

(No, don't run it)

As such, I have solved the problem of the checksum byte. I tried to set the balance to CN¥10.00, calculated the checksum, and wrote the data to my card. I got the water successfully. Then I happily changed my balance to CN¥327.67 and succeeded again.

Finally, I wrote a program to automate the balance editing process.

## Ending

The only open question was the relationship between UID and the sector encryption key. I borrowed many cards from my classmates and collected raw data as follows:

```plaintext
    UID       Sector 7 KeyA
1E 83 47 1B .. 1F B5 04 72
0E 36 62 1B .. 0F 00 21 72
FE 00 51 1B .. FF 36 12 72
4E 9B 67 1B .. 4F AD 24 72
EE 91 44 1B .. EF A7 07 72
FE 4E 9C C9 .. FF 78 DF A0
3E DB 55 E6 .. 3F ED 16 8F
0C C9 56 D5 .. 0D FF 15 BC
```

Similar to the card balance checksum, I discovered that the KeyA was calculated from the UID in two-byte units with initial CGP = 0x01364369. At this point, the water card no longer had any secret.

## Epilogue

Roughly one year has passed. The university removed all those boiler systems, and students returned their water cards. New boilers taking student ID cards as payment methods were installed. According to rumors, these new boilers were superior in all aspects: they were safer and cheaper. The only problem was that:

They were only installed on the first floor.

I lived on the seventh floor. There was no elevator.

Fuck you.

FUCK YOU.

(end)

## Postscript

1. The student ID of my university can be cracked with the same tools. The ID runs on a semi-online model: the card stores information about both your identification and your account transactions. Therefore, if you copy the card, never attempt any financial activity. On the contrary, scenarios where only your identification information was used, like access control, are pretty safe.

2. New students after 2015 are getting upgraded student ID cards. Those cards are "UltraSecure" branded security-enhanced cards, a.k.a. "hardened Mifare Classic 1K cards" in the technical community. Such cards could not be cracked with ordinary mfoc utility. Please search "hardnest attack" for cracking solutions. A clean Linux installation without mfoc and libnfc is advised if you want to try.

3. The employee cards in our affiliated teaching hospital run on a completely opposite model from the water cards - they are purely online. They only store identification tokens, and other sections of the cards are blank. They don't even have encryption. Therefore, all operations, including access control and dining hall transactions, rely on a network connection. Nothing is ever written to the card throughout its lifetime.

   For example, my department director has 1,000 bucks on his card. I copied his card and returned him the original. When I swipe my copied card at the register, it says the balance is 1,000 bucks. If the director ordered a deluxe pork ramen at the dining hall and paid 20, and I swipe my copied card again, it would tell me the balance is 980. The card readers will never find out the difference between a real one and a fake one. But my director will, as both cards are linked to his account.
