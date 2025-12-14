# CIH Virus Explanation
## Main Function
### Modify IDT
* This function is to modify the IDT
* We'll need to get ring 0 priviledge
* The procedure explanation
  1. Line 207 push eax to allocate 4 bytes onto stack
  2. Line 208 sidt [esp-02h]
    * sidt instruction writes 6 bytes (2 bytes length + 4 bytes address) from IDT
    * Write the length at esp-2, cause we don't care about that
    * The IDT base address will be write at the start 4 bytes of esp (original eax)
  3. Line 209 pop ebx will pop the IDT address just written to ebx
  4. Line 211 add ebx, HookExceptionNumber*08h+04h
    * The IDT struct [Offset Low (2)] [Selector (2)] [Flags (2)] [Offset High (2)]
    * EBX points to the 4th bytes (The starting position of Flags)
    * It is convenient for accessing later (Minus for accessing Low Bytes, Add for accessing High Bytes)
  5. Line 213 cli turn off interrupt to avoid crashing during interruption and accessing data hasn't been all written
  6. Line 215 mov ebp, [ebx] read the 4th ~ 7th bytes on ebx (Flags and Offset High)
  7. Line 216 mov bp, [ebx-04h] read the 0th ~ 1st bytes on ebx (Offset Low) and cover the low bytes on EBP
  8. Line 218 lea esi, MyExceptionHook-@1[ecx] calculates the address of the new function MyExceptionHook, and loads into lea
    * The Delta Offset (-@1[ecx]) to support position-independent code (PIC)
    * It is common for the hooked code, ensuring to calculate the right position of relative position, no matter where the memory location the code is loaded into
    * If directly use lea esi, MyExceptionHook , assembly will fill the address during assembling
    * @1 is where we stand
    * MyExceptionHook-@1[ecx] tells CPU to calculate how many steps do we need to walk from @1 to the new function entry
  9. Line 220 push esi for the rest function
  10. Let's assume ESI as 0x12345678
  11. Line 222 mov [ebx-04h], si loads 5678h into [Base+0]
  12. Line 223 shr esi, 16 shifts 16 right, and esi becomes 0x00001234
  13. Line 224 mov [ebx+02h], si loads 1234h into [Base+6]
  14. Line 226 pop esi to recover the original new function address
  15. Line 232 int HookExceptionNumber
### When to Explode?
* This procedure is to explode on a specific date
* Please check the table of CMOS reference about ten clock data registers
  * You'll know why we need to put 07h into al
  * You'll konw why we need to use port 70h, 71h on the title "Aceessing the CMOS"
[CMOS reference](./reference/CMOS-reference.txt)
* The procedure explanation
  1. Line 1208 mov al, 07h puts 07h on al
  2. Line 1209 out 70h, al sends al (07h) to port 70h
  3. Line 1210 in al, 71h returns date in port 71h
  4. Line 1212 xor al, 26h
  5. Line 1217 jnz DisableOnBusy checks condition
    * if al==26h ? 1 : 0
    * if today's not 26th, then jnz would jump to DisableOnBusy (Not execute virus)
### Kill BIOS EEPROM
* The procedure explanation
  1. Line 1280-1284 uses a specific memory mapping address to enable program mode on flash rom
  2. Line 1288 loop $ just a spinloop. Decrements ecx until it becomes zero. The CPU waits for the slowly flash rom finishing the word
  3. Line 1297 xor ah, ah set ah as zero
  4. Line 1298 loads zero (ah) into the address of eax (00E00000)
  5. Line 1300-1301 exchange eax and ecx. The ecx becomes a very large number (memory address), and the spinloop would be long enough for flash rom
  6. Line 1311-1327 kills the BIOS Main part, similar to the 6 points above
  * The BIOS memory map is divided into 2 parts: main and extra
    * The first 64KB is Main BIOS
    * The second 64KB is Extra BIOS
    * If CIH only attacks Main BIOS, Boot Block may find the Main BIOS checksum is invalid and prompt user to insert into rescue disk
    * If CIH only attacks Extra BIOS, the computer may still be able to start but something wrong happeded while reading extra part, eventually entering into DOS
    * Killing the entire 128KB chip would be better because all of them would be garbage wherever the CPU reading
