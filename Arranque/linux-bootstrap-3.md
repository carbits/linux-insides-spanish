Kernel booting process. Part 3.
================================================================================

Video mode initialization and transition to protected mode
--------------------------------------------------------------------------------

This is the third part of the `Kernel booting process` series. In the previous [part](linux-bootstrap-2.md#kernel-booting-process-part-2), we stopped right before the call of the `set_video` routine from the [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L181). In this part, we will see:
- video mode initialization in the kernel setup code,
- preparation before switching into the protected mode,
- transition to protected mode

**NOTE** If you don't know anything about protected mode, you can find some information about it in the previous [part](linux-bootstrap-2.md#protected-mode). Also there are a couple of [links](linux-bootstrap-2.md#links) which can help you.

As I wrote above, we will start from the `set_video` function which defined in the [arch/x86/boot/video.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/video.c#L315) source code file. We can see that it starts by first getting the video mode from the `boot_params.hdr` structure:

```C
u16 mode = boot_params.hdr.vid_mode;
```

which we filled in the `copy_boot_params` function (you can read about it in the previous post). `vid_mode` is an obligatory field which is filled by the bootloader. You can find information about it in the kernel boot protocol:

```
Offset	Proto	Name		Meaning
/Size
01FA/2	ALL	    vid_mode	Video mode control
```

As we can read from the linux kernel boot protocol:

```
vga=<mode>
	<mode> here is either an integer (in C notation, either
	decimal, octal, or hexadecimal) or one of the strings
	"normal" (meaning 0xFFFF), "ext" (meaning 0xFFFE) or "ask"
	(meaning 0xFFFD).  This value should be entered into the
	vid_mode field, as it is used by the kernel before the command
	line is parsed.
```

So we can add `vga` option to the grub or another bootloader configuration file and it will pass this option to the kernel command line. This option can have different values as we can mentioned in the description, for example it can be an integer number `0xFFFD` or `ask`. If you pass `ask` t `vga`, you will see a menu like this:

![video mode setup menu](http://oi59.tinypic.com/ejcz81.jpg)

which will ask to select a video mode. We will look at it's implementation, but before diving into the implementation we have to look at some other things.

Kernel data types
--------------------------------------------------------------------------------

Earlier we saw definitions of different data types like `u16` etc. in the kernel setup code. Let's look on a couple of data types provided by the kernel:


| Type | char | short | int | long | u8 | u16 | u32 | u64 |
|------|------|-------|-----|------|----|-----|-----|-----|
| Size |  1   |   2   |  4  |   8  |  1 |  2  |  4  |  8  |

If you read source code of the kernel, you'll see these very often and so it will be good to remember them.

Heap API
--------------------------------------------------------------------------------

After we have `vid_mode` from the `boot_params.hdr` in the `set_video` function we can see call to `RESET_HEAP` function. `RESET_HEAP` is a macro which defined in the [boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/boot/boot.h#L199). It is defined as:

```C
#define RESET_HEAP() ((void *)( HEAP = _end ))
```

If you have read the second part, you will remember that we initialized the heap with the [`init_heap`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) function. We have a couple of utility functions for heap which are defined in `boot.h`. They are:

```C
#define RESET_HEAP()
```

As we saw just above it resets the heap by setting the `HEAP` variable equal to `_end`, where `_end` is just `extern char _end[];`

Next is `GET_HEAP` macro:

```C
#define GET_HEAP(type, n) \
	((type *)__get_heap(sizeof(type),__alignof__(type),(n)))
```

for heap allocation. It calls internal function `__get_heap` with 3 parameters:

* size of a type in bytes, which need be allocated
* `__alignof__(type)` shows how type of variable is aligned
* `n` tells how many bytes to allocate

Implementation of `__get_heap` is:

```C
static inline char *__get_heap(size_t s, size_t a, size_t n)
{
	char *tmp;

	HEAP = (char *)(((size_t)HEAP+(a-1)) & ~(a-1));
	tmp = HEAP;
	HEAP += s*n;
	return tmp;
}
```

and further we will see its usage, something like:

```C
saved.data = GET_HEAP(u16, saved.x * saved.y);
```

Let's try to understand how `__get_heap` works. We can see here that `HEAP` (which is equal to `_end` after `RESET_HEAP()`) is the address of aligned memory according to `a` parameter. After it we save memory address from `HEAP` to the `tmp` variable, move `HEAP` to the end of allocated block and return `tmp` which is start address of allocated memory.

And the last function is:

```C
static inline bool heap_free(size_t n)
{
	return (int)(heap_end - HEAP) >= (int)n;
}
```

which subtracts value of the `HEAP` from the `heap_end` (we calculated it in the previous [part](linux-bootstrap-2.md)) and returns 1 if there is enough memory for `n`.

That's all. Now we have simple API for heap and can setup video mode.

Setup video mode
--------------------------------------------------------------------------------

Now we can move directly to video mode initialization. We stopped at the `RESET_HEAP()` call in the `set_video` function. Next is the call to  `store_mode_params` which stores video mode parameters in the `boot_params.screen_info` structure which is defined in the [include/uapi/linux/screen_info.h](https://github.com/0xAX/linux/blob/master/include/uapi/linux/screen_info.h).

If we will look at `store_mode_params` function, we can see that it starts with the call to `store_cursor_position` function. As you can understand from the function name, it gets information about cursor and stores it.

First of all `store_cursor_position` initializes two variables which has type - `biosregs`, with `AH = 0x3` and calls `0x10` BIOS interruption. After interruption successfully executed, it returns row and column in the `DL` and `DH` registers. Row and column will be stored in the `orig_x` and `orig_y` fields from the the `boot_params.screen_info` structure.

After `store_cursor_position` executed, `store_video_mode` function will be called. It just gets current video mode and stores it in the `boot_params.screen_info.orig_video_mode`. 

After this, it checks current video mode and sets the `video_segment`. After the BIOS transfers control to the boot sector, the following addresses are for video memory:

```
0xB000:0x0000 	32 Kb 	Monochrome Text Video Memory
0xB800:0x0000 	32 Kb 	Color Text Video Memory
```

So we set the `video_segment` variable to `0xB000` if current video mode is MDA, HGC, VGA in monochrome mode or `0xB800` in color mode. After setup of the address of the video segment font size needs to be stored in the `boot_params.screen_info.orig_video_points` with:

```C
set_fs(0);
font_size = rdfs16(0x485);
boot_params.screen_info.orig_video_points = font_size;
```

First of all we put 0 to the `FS` register with `set_fs` function. We already saw functions like `set_fs` in the previous part. They are all defined in the [boot.h](https://github.com/0xAX/linux/blob/master/arch/x86/boot/boot.h). Next we read value which is located at address `0x485` (this memory location is used to get the font size) and save font size in the `boot_params.screen_info.orig_video_points`.

```
 x = rdfs16(0x44a);
 y = (adapter == ADAPTER_CGA) ? 25 : rdfs8(0x484)+1;
```

Next we get amount of columns by `0x44a` and rows by address `0x484` and store them in the `boot_params.screen_info.orig_video_cols` and `boot_params.screen_info.orig_video_lines`. After this, execution of the `store_mode_params` is finished.

Next we can see `save_screen` function which just saves screen content to the heap. This function collects all data which we got in the previous functions like rows and columns amount etc. and stores it in the `saved_screen` structure, which is defined as:

```C
static struct saved_screen {
	int x, y;
	int curx, cury;
	u16 *data;
} saved;
```

It then checks whether the heap has free space for it with:

```C
if (!heap_free(saved.x*saved.y*sizeof(u16)+512))
		return;
```

and allocates space in the heap if it is enough and stores `saved_screen` in it.

The next call is `probe_cards(0)` from the [arch/x86/boot/video-mode.c](https://github.com/0xAX/linux/blob/master/arch/x86/boot/video-mode.c#L33). It goes over all video_cards and collects number of modes provided by the cards. Here is the interesting moment, we can see the loop:

```C
for (card = video_cards; card < video_cards_end; card++) {
  /* collecting number of modes here */
}
```

but `video_cards` not declared anywhere. Answer is simple: Every video mode presented in the x86 kernel setup code has definition like this:

```C
static __videocard video_vga = {
	.card_name	= "VGA",
	.probe		= vga_probe,
	.set_mode	= vga_set_mode,
};
```

where `__videocard` is a macro:

```C
#define __videocard struct card_info __attribute__((used,section(".videocards")))
```

which means that `card_info` structure:

```C
struct card_info {
	const char *card_name;
	int (*set_mode)(struct mode_info *mode);
	int (*probe)(void);
	struct mode_info *modes;
	int nmodes;
	int unsafe;
	u16 xmode_first;
	u16 xmode_n;
};
```

is in the `.videocards` segment. Let's look in the [arch/x86/boot/setup.ld](https://github.com/0xAX/linux/blob/master/arch/x86/boot/setup.ld) linker file, we can see there:

```
	.videocards	: {
		video_cards = .;
		*(.videocards)
		video_cards_end = .;
	}
```

It means that `video_cards` is just memory address and all `card_info` structures are placed in this  segment. It means that all `card_info` structures are placed between `video_cards` and `video_cards_end`, so we can use it in a loop to go over all of it.  After `probe_cards` executed we have all structures like `static __videocard video_vga` with filled `nmodes` (number of video modes).

After `probe_cards` execution is finished, we move to the main loop in the `set_video` function. There is infinite loop which tries to setup video mode with the `set_mode` function or prints a menu if we passed `vid_mode=ask` to the kernel command line or video mode is undefined. 

The `set_mode` function is defined in the [video-mode.c](https://github.com/0xAX/linux/blob/master/arch/x86/boot/video-mode.c#L147) and gets only one parameter, `mode` which is the number of video mode (we got it or from the menu or in the start of the `setup_video`, from kernel setup header). 

`set_mode` function checks the `mode` and calls `raw_set_mode` function. The `raw_set_mode` calls `set_mode` function for selected card i.e. `card->set_mode(struct mode_info*)`. We can get access to this function from the `card_info` structure, every video mode defines this structure with values filled depending upon the video mode (for example for `vga` it is `video_vga.set_mode` function, see above example of `card_info` structure for `vga`). `video_vga.set_mode` is `vga_set_mode`, which checks the vga mode and calls the respective function:

```C
static int vga_set_mode(struct mode_info *mode)
{
	vga_set_basic_mode();

	force_x = mode->x;
	force_y = mode->y;

	switch (mode->mode) {
	case VIDEO_80x25:
		break;
	case VIDEO_8POINT:
		vga_set_8font();
		break;
	case VIDEO_80x43:
		vga_set_80x43();
		break;
	case VIDEO_80x28:
		vga_set_14font();
		break;
	case VIDEO_80x30:
		vga_set_80x30();
		break;
	case VIDEO_80x34:
		vga_set_80x34();
		break;
	case VIDEO_80x60:
		vga_set_80x60();
		break;
	}
	return 0;
}
```

Every function which setups video mode, just calls `0x10` BIOS interrupt with certain value in the `AH` register.

After we have set video mode, we pass it to the `boot_params.hdr.vid_mode`.

Next `vesa_store_edid` is called. This function simply stores the [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data) (**E**xtended **D**isplay **I**dentification **D**ata) information for kernel use. After this `store_mode_params` is called again. Lastly, if `do_restore` is set, screen is restored to an earlier state.

After this we have set video mode and now we can switch to the protected mode.

Last preparation before transition into protected mode
--------------------------------------------------------------------------------

We can see the last function call - `go_to_protected_mode` in the [main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L184). As the comment says: `Do the last things and invoke protected mode`, so let's see these last things and switch into the protected mode.

`go_to_protected_mode` defined in the [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pm.c#L104). It contains some functions which make last preparations before we can jump into protected mode, so let's look on it and try to understand what they do and how it works.

First is the call to `realmode_switch_hook` function in the `go_to_protected_mode`. This function invokes real mode switch hook if it is present and disables [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt). Hooks are used if bootloader runs in a hostile environment. You can read more about hooks in the [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) (see **ADVANCED BOOT LOADER HOOKS**).

`readlmode_swtich` hook presents pointer to the 16-bit real mode far subroutine which disables non-maskable interrupts. After `realmode_switch` hook (it isn't present for me) is checked, disabling of Non-Maskable Interrupts(NMI) occurs:

```assembly
asm volatile("cli");
outb(0x80, 0x70);	/* Disable NMI */
io_delay();
```

At first there is inline assembly instruction with `cli` instruction which clears the interrupt flag (`IF`). After this, external interrupts are disabled. Next line disables NMI (non-maskable interrupt).

Interrupt is a signal to the CPU which is emitted by hardware or software. After getting signal, CPU suspends current instructions sequence, saves its state and transfers control to the interrupt handler. After interrupt handler has finished it's work, it transfers control to the interrupted instruction. Non-maskable interrupts (NMI) are interrupts which are always processed, independently of permission. It cannot be ignored and is typically used to signal for non-recoverable hardware errors. We will not dive into details of interrupts now, but will discuss it in the next posts.

Let's get back to the code. We can see that second line is writing `0x80` (disabled bit) byte to the `0x70` (CMOS Address register). After that call to the `io_delay` function occurs. `io_delay` causes a small delay and looks like:

```C
static inline void io_delay(void)
{
	const u16 DELAY_PORT = 0x80;
	asm volatile("outb %%al,%0" : : "dN" (DELAY_PORT));
}
```

Outputting any byte to the port `0x80` should delay exactly 1 microsecond. So we can write any value (value from `AL` register in our case) to the `0x80` port. After this delay `realmode_switch_hook` function has finished execution and we can move to the next function.

The next function is `enable_a20`, which enables [A20 line](http://en.wikipedia.org/wiki/A20_line). This function is defined in the [arch/x86/boot/a20.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/a20.c) and it tries to enable A20 gate with different methods. The first is `a20_test_short` function which checks is A20 already enabled or not with `a20_test` function:

```C
static int a20_test(int loops)
{
	int ok = 0;
	int saved, ctr;

	set_fs(0x0000);
	set_gs(0xffff);

	saved = ctr = rdfs32(A20_TEST_ADDR);

    while (loops--) {
		wrfs32(++ctr, A20_TEST_ADDR);
		io_delay();	/* Serialize and make delay constant */
		ok = rdgs32(A20_TEST_ADDR+0x10) ^ ctr;
		if (ok)
			break;
	}

	wrfs32(saved, A20_TEST_ADDR);
	return ok;
}
```

First of all we put `0x0000` to the `FS` register and `0xffff` to the `GS` register. Next we read value by address `A20_TEST_ADDR` (it is `0x200`) and put this value into `saved` variable and `ctr`.

Next we write updated `ctr` value into `fs:gs` with `wrfs32` function, then delay for 1ms, and then read the value into the `GS` register by address `A20_TEST_ADDR+0x10`, if it's not zero we already have enabled A20 line. If A20 is disabled, we try to enable it with a different method which you can find in the `a20.c`. For example with call of `0x15` BIOS interrupt with `AH=0x2041` etc.

If `enabled_a20` function finished with fail, print an error message and call function `die`. You can remember it from the first source code file where we started - [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S):

```assembly
die:
	hlt
	jmp	die
	.size	die, .-die
```

After the A20 gate is successfully enabled, `reset_coprocessor` function is called:
 ```C
outb(0, 0xf0);
outb(0, 0xf1);
```
This function clears the Math Coprocessor by writing `0` to `0xf0` and then resets it by writing `0` to `0xf1`.

After this `mask_all_interrupts` function is called:
```C
outb(0xff, 0xa1);       /* Mask all interrupts on the secondary PIC */
outb(0xfb, 0x21);       /* Mask all but cascade on the primary PIC */
```
This masks all interrupts on the secondary PIC (Programmable Interrupt Controller) and primary PIC except for IRQ2 on the primary PIC.

And after all of these preparations, we can see actual transition into protected mode.

Setup Interrupt Descriptor Table
--------------------------------------------------------------------------------

Now we setup the Interrupt Descriptor table (IDT). `setup_idt`:

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

which setups the Interrupt Descriptor Table (describes interrupt handlers and etc.). For now IDT is not installed (we will see it later), but now we just load IDT with `lidtl` instruction. `null_idt` contains address and size of IDT, but now they are just zero. `null_idt` is a `gdt_ptr` structure, it as defined as:
```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

where we can see - 16-bit length(`len`) of IDT and 32-bit pointer to it (More details about IDT and interruptions we will see in the next posts). ` __attribute__((packed))` means here that size of `gdt_ptr` minimum as required. So size of the `gdt_ptr` will be 6 bytes here or 48 bits. (Next we will load pointer to the `gdt_ptr` to the `GDTR` register and you might remember from the previous post that it is 48-bits in size).

Setup Global Descriptor Table
--------------------------------------------------------------------------------

Next is the setup of Global Descriptor Table (GDT). We can see `setup_gdt` function which sets up GDT (you can read about it in the [Kernel booting process. Part 2.](linux-bootstrap-2.md#protected-mode)). There is definition of the `boot_gdt` array in this function, which contains definition of the three segments:

```C
	static const u64 boot_gdt[] __attribute__((aligned(16))) = {
		[GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
		[GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
		[GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
	};
```

For code, data and TSS (Task State Segment). We will not use task state segment for now, it was added there to make Intel VT happy as we can see in the comment line (if you're interesting you can find commit which describes it - [here](https://github.com/torvalds/linux/commit/88089519f302f1296b4739be45699f06f728ec31)). Let's look on `boot_gdt`. First of all note that it has `__attribute__((aligned(16)))` attribute. It means that this structure will be aligned by 16 bytes. Let's look at a simple example:
```C
#include <stdio.h>

struct aligned {
	int a;
}__attribute__((aligned(16)));

struct nonaligned {
	int b;
};

int main(void)
{
	struct aligned    a;
	struct nonaligned na;

	printf("Not aligned - %zu \n", sizeof(na));
	printf("Aligned - %zu \n", sizeof(a));

	return 0;
}
```

Technically structure which contains one `int` field, must be 4 bytes, but here `aligned` structure will be 16 bytes:

```
$ gcc test.c -o test && test
Not aligned - 4
Aligned - 16
```

`GDT_ENTRY_BOOT_CS` has index - 2 here, `GDT_ENTRY_BOOT_DS` is `GDT_ENTRY_BOOT_CS + 1` and etc. It starts from 2, because first is a mandatory null descriptor (index - 0) and the second is not used (index - 1).

`GDT_ENTRY` is a macro which takes flags, base and limit and builds GDT entry. For example let's look on the code segment entry. `GDT_ENTRY` takes following values:

* base  - 0
* limit - 0xfffff
* flags - 0xc09b

What does it mean? Segment's base address is 0, limit (size of segment) is - `0xffff` (1 MB). Let's look on flags. It is `0xc09b` and it will be:

```
1100 0000 1001 1011
```

in binary. Let's try to understand what every bit means. We will go through all bits from left to right:

* 1    - (G) granularity bit
* 1    - (D) if 0 16-bit segment; 1 = 32-bit segment
* 0    - (L) executed in 64 bit mode if 1
* 0    - (AVL) available for use by system software
* 0000 - 4 bit length 19:16 bits in the descriptor
* 1    - (P) segment presence in memory
* 00   - (DPL) - privilege level, 0 is the highest privilege
* 1    - (S) code or data segment, not a system segment
* 101  - segment type execute/read/
* 1    - accessed bit

You can read more about every bit in the previous [post](linux-bootstrap-2.md) or in the [Intel® 64 and IA-32 Architectures Software Developer's Manuals 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html).

After this we get length of GDT with:

```C
gdt.len = sizeof(boot_gdt)-1;
```

We get size of `boot_gdt` and subtract 1 (the last valid address in the GDT).

Next we get pointer to the GDT with:

```C
gdt.ptr = (u32)&boot_gdt + (ds() << 4);
```

Here we just get address of `boot_gdt` and add it to address of data segment left-shifted by 4 bits (remember we're in the real mode now).

Lastly we execute `lgdtl` instruction to load GDT into GDTR register:

```C
asm volatile("lgdtl %0" : : "m" (gdt));
```

Actual transition into protected mode
--------------------------------------------------------------------------------

It is the end of `go_to_protected_mode` function. We loaded IDT, GDT, disable interruptions and now can switch CPU into protected mode. The last step we call `protected_mode_jump` function with two parameters:

```C
protected_mode_jump(boot_params.hdr.code32_start, (u32)&boot_params + (ds() << 4));
```

which is defined in the [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S#L26). It takes two parameters:

* address of protected mode entry point
* address of `boot_params`

Let's look inside `protected_mode_jump`. As I wrote above, you can find it in the `arch/x86/boot/pmjump.S`. First parameter will be in `eax` register and second is in `edx`.

First of all we put address of `boot_params` in the `esi` register and address of code segment register `cs` (0x1000) in the `bx`. After this we shift `bx` by 4 bits and add address of label `2` to it (we will have physical address of label `2` in the `bx` after it) and jump to label `1`. Next we put data segment and task state segment in the `cs` and `di` registers with:

```assembly
movw	$__BOOT_DS, %cx
movw	$__BOOT_TSS, %di
```

As you can read above `GDT_ENTRY_BOOT_CS` has index 2 and every GDT entry is 8 byte, so `CS` will be `2 * 8 = 16`, `__BOOT_DS` is 24 etc.

Next we set `PE` (Protection Enable) bit in the `CR0` control register:

```assembly
movl	%cr0, %edx
orb	$X86_CR0_PE, %dl
movl	%edx, %cr0
```

and make long jump to the protected mode:

```assembly
	.byte	0x66, 0xea
2:	.long	in_pm32
	.word	__BOOT_CS
```

where
* `0x66` is the operand-size prefix which allows to mix 16-bit and 32-bit code,
* `0xea` - is the jump opcode,
* `in_pm32` is the segment offset
* `__BOOT_CS` is the code segment.

After this we are finally in the protected mode:

```assembly
.code32
.section ".text32","ax"
```

Let's look at the first steps in the protected mode. First of all we setup data segment with:

```assembly
movl	%ecx, %ds
movl	%ecx, %es
movl	%ecx, %fs
movl	%ecx, %gs
movl	%ecx, %ss
```

If you read with attention, you can remember that we saved `$__BOOT_DS` in the `cx` register. Now we fill with it all segment registers besides `cs` (`cs` is already `__BOOT_CS`). Next we zero out all general purpose registers besides `eax` with:

```assembly
xorl	%ecx, %ecx
xorl	%edx, %edx
xorl	%ebx, %ebx
xorl	%ebp, %ebp
xorl	%edi, %edi
```

And jump to the 32-bit entry point in the end:

```
jmpl	*%eax
```

Remember that `eax` contains address of the 32-bit entry (we passed it as first parameter into `protected_mode_jump`).

That's all we're in the protected mode and stop at it's entry point. What happens next, we will see in the next part.

Conclusion
--------------------------------------------------------------------------------

It is the end of the third part about linux kernel internals. In next part we will see first steps in the protected mode and transition into the [long mode](http://en.wikipedia.org/wiki/Long_mode).

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes, please send me a PR with corrections at [linux-internals](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [VGA](http://en.wikipedia.org/wiki/Video_Graphics_Array)
* [VESA BIOS Extensions](http://en.wikipedia.org/wiki/VESA_BIOS_Extensions)
* [Data structure alignment](http://en.wikipedia.org/wiki/Data_structure_alignment)
* [Non-maskable interrupt](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [A20](http://en.wikipedia.org/wiki/A20_line)
* [GCC designated inits](https://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Designated-Inits.html)
* [GCC type attributes](https://gcc.gnu.org/onlinedocs/gcc/Type-Attributes.html)
* [Previous part](linux-bootstrap-2.md)
