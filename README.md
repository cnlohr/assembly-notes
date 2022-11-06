# assembly-notes

My repo for notes on assembly lagnuage, inling, etc.

## Overall



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


Example:
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

x86_64 assembly

strlen, but no loops (sort of)

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

(Link to youtube video will go here)

### References from the presentation:

