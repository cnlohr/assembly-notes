# assembly-notes

My repo for notes on assembly lagnuage, inling, etc.

## Overall

## Dedicated Assembly

* https://github.com/AEFeinstein/Super-2023-Swadge-FW/blob/main/tools/sandbox_test/
* Example linker script: https://github.com/AEFeinstein/Super-2023-Swadge-FW/blob/main/tools/bootload_reboot_stub/esp32_s2_stub.ld
* Manual on special commands for dedicated (gas) assembly: http://www.math.utah.edu/docs/info/as_7.html

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

Constraint Modifiers:
 * `=` write-only -- note `&` is a different type of write-only constraint
 * `+` read-and-write
 * `&` early-clobber. Applies to both `=` and `+` Without this, input registers are allowed to be assigned to the same register as your output. See below for when this may be important.
   * NOTE: If you must have read-and-write inputs be two separate registers, you must make them `+&`.
 * NOTE: Do not modify InputOperands! It will break things. By telling the compiler something is an input operand you are saying it **will not** be written to.
 * `g` pointer
 * `r` register
 * `a` architecture-specific, see https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html for more information

Demonstration: read-after-write can lead to undesired behavior

```c
static __attribute__((noinline)) uint32_t incorrect_1(uint32_t a) {
    uint32_t ret = 0; // NOTE: it would technically be legal to keep this value uninitlalized
    asm volatile(
        "mov $10, %0\n"
        "add %1, %0"// add eax, eax -> bad!
        :"=r"(ret):"r"(a));
    return ret;
}

static __attribute__((noinline)) uint32_t correct_1(uint32_t a) {
    uint32_t ret = 0;
    asm volatile(
        "mov $10, %0\n"
        "add %1, %0" // add eax, edx -> fine
        :"=&r"(ret):"r"(a));
    return ret;
}

int main() {
    incorrect_1(0);
    correct_1(0);
}
```

Demonstration: _never_ write to input constraints!
```c
uint32_t incorrect_2() {
    uint32_t ret = 0;
    asm volatile(
        "mov $10, %0\n"
        ::"r"(ret));
    return ret; // returns 10
}

uint32_t incorrect_3() {
    uint32_t ret = 0;
    asm volatile(
        "mov $10, %0\n"
        ::"r"(ret));
    return ret + 1; // returns 1!
}
```

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

## `"memory"` Clobber.
If you modify memory inside your assembly, you, many times will need to add `"memory"` to your clobber list.** There are two major uses, one is that if it's possible for code after your code to read memory that has been altered by your code, you need to specify `memory` in your clobber so that GCC isn't using cached (into a register or something) versions of the memory that your code has modified. **And** the other use is to guarantee that any memory accesses have been accomplished before you execute further code.  I.e. the following code:

```
static inline void spin_unlock(spinlock_t *lock)
{
   __asm__ __volatile__(""::: "memory");
   lock->lock = 0;
}
```

## More examples.

Example: Get current # of cycles (processor cycles) counter on ESP32, ESP8266:

```c
static inline uint32_t getCycleCount() {
   uint32_t ccount;
   asm volatile("rsr %0,ccount":"=a" (ccount));
   return ccount;
}
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


Side-note:  you can put raw opcodes in your code, in case the compiler doesn't know about some processor quirk.

```c
static void CUSTOMLOG( const char * c ) {
   asm volatile( "csrrw x0, 0x137, %0\n.long 0xff100073" : : "r" (c));
}
```

## Relative Jump Labels

You can use relative jump labels.  You can jump forward with a number and `f` and back with a number and `b`.

```c
asm volatile(
"	la a0, _sbss\n\
	la a1, _ebss\n\
	li a2, 0\n\
	bge a0, a1, 2f\n\
1:	sw a2, 0(a0)\n\
	addi a0, a0, 4\n\
	blt a0, a1, 1b\n\
2:"
	// This loads DATA from FLASH to RAM.
"	la a0, _data_lma\n\
	la a1, _data_vma\n\
	la a2, _edata\n\
1:	beq a1, a2, 2f\n\
	lw a3, 0(a0)\n\
	sw a3, 0(a1)\n\
	addi a0, a0, 4\n\
	addi a1, a1, 4\n\
	bne a1, a2, 1b\n\
2:\n" );
```

## Random Tricks:

* Prevent the compiler from emitting dumb l32r's, i.e. Tell the compiler "Please, do not load another literal, please instead compute `c1` from `c0` - thanks, @duk-37

```c
	uint32_t c0 = 0xF0F0F0F0;
	asm("":"+r"(c0)); // Force the compiler to not do an l32r.
	uint32_t c1 = c0 >> 4;
	// c1 = 0x0f0f0f0f, now, but without extra l32r
```

* Super-fast integer absolute value

```c
// .725 seconds
static inline ABS(x) { if( x < 0 ) return -x; else return x; }

// .727 seconds
static inline ABS(x) { int mask = mask = x>>31; return (mask + x)^mask; }

// .673 seconds
#define ABS(x) abs(x) 

// .554 seconds!!!
static inline ABS( int x ) { asm volatile("\nmov %[x], %%ebx\nneg %[x]\ncmovl %%ebx,%[x]\n" : : [x]"r"(x) : "ebx" ); return x; }
```
(-O4, gcc 9.4.0, x86_64, Run times are my day 15 Advent of Code 2022 Challenge, Part 1, which uses a lot of abs's)

### Random example

This does an integer multiply on systems taht don't have a multiply instruction.

```c
	uint32_t ret = 0;
	asm volatile( "\n\
		.option   rvc;\n\
	1:	andi t0, %[small], 1\n\
		beqz t0, 2f\n\
		add %[ret], %[ret], %[big]\n\
	2:	srli %[small], %[small], 1\n\
		slli %[big], %[big], 1\n\
		bnez %[small], 1b\n\
	" :
		[ret]"=&r"(ret) , [big]"+&r"(big_num), [small]"+&r"(small_num) : :
		"t0" );
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
