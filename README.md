# assembly-notes

My repo for notes on assembly lagnuage, inling, etc.

## Overall

## Dedicated Assembly

* https://github.com/AEFeinstein/Super-2023-Swadge-FW/blob/main/tools/sandbox_test/
* Example linker script: https://github.com/AEFeinstein/Super-2023-Swadge-FW/blob/main/tools/bootload_reboot_stub/esp32_s2_stub.ld

## Using Inline Assembly

Webpages (For inline assembly)
 * https://dmalcolm.fedorapeople.org/gcc/2015-08-31/rst-experiment/how-to-use-inline-assembly-language-in-c-code.html
 * https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Clobbers-and-Scratch-Registers
 * https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html

Tools:
 * https://godbolt.org
 * Get a map file from an elf:  `objdump -t <elf file>`
 * Get an assembly listing from an elf: `objdump -S <elf file>`
 * GCC Pre-link assmebly tools (you RARELY want these)
   * Make GCC produce in-line assembly: Add `-Wa,-a,-ad`
   * Make GCC produce in-line map file: Add `-Wl,-Map,test.debug.map`

Constraints:
 * `=a` write-only  -- note `&` is a different type of write-only constraint
 * `+a` read-and-write
 * `a` read-only
 * `g` pointer
 * `r` register

Labels in inline assembly: `%=` will create a unique thing for this section, so `skip_pix%=` jumps to `skip_pix%=:`


Register named syntax: Use `%[registername]` by specifying  `: [registername]"constraint"(C-value),... : ` 

```c
asm [volatile] ( ``AssemblerTemplate``
                 : ``OutputOperands``
                 [ : ``InputOperands``
                 [ : ``Clobbers`` ] ])

asm [volatile] goto ( ``AssemblerTemplate``
                      :
                      : ``InputOperands``
                      : ``Clobbers``
                      : ``GotoLabels``)
```


Example (Xtensa LX7, ESP32S2, ESP32):
```c
#define TURBO_SET_PIXEL(opxc, opy, colorVal ) \
	asm volatile( "mul16u a4, %[width], %[y]\nadd a4, a4, %[px]\nadd a4, a4, %[opx]\ns8i %[val],a4, 0" \
		: : [opx]"a"(opxc),[y]"a"(opy),[px]"a"(dispPx),[val]"a"(colorVal),[width]"a"(dispWidth) : "a4" );

// Very tricky: 
//   We do bgeui which checks to make sure 0 <= x < MAX
//   Other than that, it's basically the same as above.
#define TURBO_SET_PIXEL_BOUNDS(opxc, opy, colorVal ) \
	asm volatile( "bgeu %[opx], %[width], failthrough%=\nbgeu %[y], %[height], failthrough%=\nmul16u a4, %[width], %[y]\nadd a4, a4, %[px]\nadd a4, a4, %[opx]\ns8i %[val],a4, 0\nfailthrough%=:\nnop" \
		: : [opx]"a"(opxc),[y]"a"(opy),[px]"a"(dispPx),[val]"a"(colorVal),[width]"a"(dispWidth),[height]"a"(dispHeight) : "a4" );
```

x86_64 assembly: strlen, but no loops (sort of)
```c
	unsigned long long ret = 0;
	asm volatile("\n\
		xor %%rax, %%rax\n\
		xor %%rcx, %%rcx\n\
		dec %%rcx\n\
		mov %[strin], %%rdi\n\
		repnz scasb (%%rdi), %%al\n\
		not %%rcx\n\
		mov %%rcx, %[ret]\n\
		" : [ret]"=r"(ret) : [strin]"r"(str) : "rdi","rcx","rax" );
	return ret;
```


## The presentation.

https://drive.google.com/drive/folders/1WUkw5rC5yDKR2lT6nQkdDeGCpMPmpI3f?usp=sharing

### References from the presentation:

1) https://en.wikipedia.org/wiki/RollerCoaster_Tycoon_(video_game)#/media/File:RollerCoaster_Tycoon_Screenshot.png Retrieved 2022-10-24
2) https://www.reddit.com/r/ProgrammerHumor/comments/ji8sx6/still_assembly_is_one_of_my_favourite_language/ Retrieved 2022-10-24
3) https://www.youtube.com/watch?v=eAhWIO1Ra6M (Published Jun 6, 2015)
4) https://hacks.mozilla.org/2013/12/gap-between-asm-js-and-native-performance-gets-even-narrower-with-float32-optimizations/ (Published Dec 20, 2013)
5) https://www.reddit.com/r/ProgrammerHumor/comments/6tifn2/the_holy_trinity_and_javascript/ (Published August 13 2017)
6) https://knowyourmeme.com/memes/ackchyually-actually-guy (Retrieved 2022-10-25)
7) https://www.youtube.com/watch?v=-E36UNXMjnU (2018-12-21)
8) https://www.youtube.com/watch?v=QM1iUe6IofM (2016-01-18)
9) https://www.youtube.com/watch?v=GKYCA3UsmrU  (Cut from original video uploaded Jun 11, 2015)
10) https://en.wikipedia.org/w/index.php?title=Red%E2%80%93black_tree&oldid=675908841 (2015-08-13)
11) https://www.youtube.com/watch?v=m4f4OzEyueg ( 2014-09-05)
12) https://github.com/WebAssembly/design/issues/796 (Retrieved 2022-10-29)
13) https://imgur.com/lZLyoaA (2017-11-09)
14) http://ww1.microchip.com/downloads/en/devicedoc/atmel-0856-avr-instruction-set-manual.pdf (2016-11-01)
15) https://gist.github.com/cnlohr/e11d59f98c94f748683ba7ec80667d51 (2022-10-28)
16) https://www.youtube.com/watch?v=etdKnhwWtwA (2013-02-12)
17) https://luplab.cs.ucdavis.edu/2022/06/03/rvcodec-js.html (2022-06-03)
18) https://blog.pimaker.at/texts/rvc1/ (2021-08-25)
19) https://www.pinatafarm.com/memegenerator/4a771794-6d5a-42ef-a1c8-7744545233d8 (Retrieved 2022-11-04)
20) https://knowyourmeme.com/memes/surprised-pikachu (Retrieved 2022-11-04)
21) https://en.wikipedia.org/wiki/Amdahl%27s_law#/media/File:Optimizing-different-parts.svg  (Retrieved 2008-01-10)
22) https://azeria-labs.com/arm-data-types-and-registers-part-2/ (Retrieved 2022-11-04)
23) https://www.youtube.com/watch?v=wU2UoxLtIm8 (2021-02-09)
24) https://knowyourmeme.com/memes/salt-bae (Retrieved 2022-11-05)
25) https://github.com/cnlohr/esp32s2-cookbook (Retrieved 2022-11-06
