# Selicre's ASM cookbook

Welcome! This is a compilation of interesting ASM techniques that will come in handy when writing 65816 assembly. This guide assumes you know every instruction and what it does; if you don't, feel free to read [this](http://www.6502.org/tutorials/65c816opcodes.html) page.

# Generic 65816 techniques

## Using data tables

There's a variety of ways to read and modify tables on 65816.

### Indexed

```
code:
    ldx index
    lda table,x
    ...
    
table:
    db $00, $FF
```

You're probably aware of this one already: you have a static table somewhere in rom or ram, and you read it with an offset. It's often used for unstructured data, and `lda long,x` lets you read/write anywhere on the system bus without having to set the DBR.

### Indirect indexed

```
code:
    lda #table
    sta $00
    lda ($00),y
; OR
code:
    lda #table
    sta $00
    ldx #table>>16
    stx $02
    lda [$00],y

table:
    incbin "data.bin"
```

This addressing mode is widely used internally when accessing level data, spawning sprites, etc.; you can use it for the same purposes, or to build your own system entirely, for example, for textboxes. ($00),y uses a word pointer extended by the DBR, while [$00],y uses a 24-bit pointer. This technique performs best when you do lots of reads using the same pointer with different offsets.

Note that ($00) and [$00], unindexed, also exist, which might be useful in some more niche scenarios (like reading the header byte).

### Indexed indirect

```
code:
    ;; X = table
    lda ($00,x)
    inc $00,x
    
; RAM at `table`: table of pointers 
```

This setup isn't common. Since this instruction is only available as (dp,x), it's stuck in bank 0 and thus is really limited to low ram, since in smw hacking you don't have much control over bank 0. Furthermore, it doesn't let you add an offset to the pointer in memory, meaning you have to modify the contents of memory to be able to fully use this technique. However, if you just so happen to have to keep track of multiple parallel streams of data, or if you have sprite tables with a stride of 2, being aware of this addressing mode might help.


## Stack nonsense

Stack manipulation must be done with care. You need to be aware that at any point, an interrupt may fire, potentially totally clobbering whatever you were trying to set up. Nevertheless, there's still some interesting things you can do.

### Return chaining

```
code:
    pea rt1-1
    pea rt2-1
    pea rt3-1
    jsr rt4
```

Pushing multiple return addresses allows you to save a negligible amount of cycles if you need to call multiple subroutines in sequence, since each rts would instantly move execution to the next function. Outside of saving cycles, it's useful if you need to collate subroutine calls first, possibly do extra setup, then execute all of them.

### Inline subroutine arguments

If you have a subroutine that takes a large amount of arguments that do not need to be dynamic per call, you can use the following snippet for convenience:

```
; 16-bit AX
code:
    jsr call_fn
    dw $1234, $abcd, $5678
    ...
    
call_fn:
    lda 1,s             ; Read the return address
    tax                 ; Save it in X
    clc : adc #$0006    ; Adjust return address
    sta 1,s             ; Advance return address to skip the inline data
    lda $0001,x         ; Read the data ($1234)
    lda $0003,x         ; Read the data ($abcd)
    lda $0005,x         ; Read the data ($5678)
    rts
```

## Coroutines

Coroutines are a comprehensive replacement for state machines, and they are useful for things like boss sprites and very complex script behaviour. This technique is used in the Dram 3 Bowser, as well as the battle sequence in the fish troll level.

### Stackless

A coroutine, conceptually, is just a pausable function. Thus, it needs to preserve all of its state somewhere. In order to do this, a yielding subroutine or macro is used. Here's a setup for a simple coroutine:

```
; 16-bit AX
coro_init:
    lda #coro_entrypoint
    sta coro_ptr
coro_run:
    jmp (coro_ptr)


; This is where the coroutine starts execution.
coro_entrypoint:
    ; Do something
    jsr yield
    ; Do other thing
    jsr yield
    ; Do yet more things
    ; Terminate by looping around the yield function
-
    jsr yield
    bra -

; The subroutine you call to pause execution
yield:
    lda 1,s     ; Get the return address
    inc         ; Adjust it to work with jmp
    sta coro_ptr
    rts
```

This is a decent start, however, you'll find out that, expectedly, none of the registers are preserved. This can be changed by, indeed, stashing them in `yield` and then restoring them in `coro_run`, making loops much easier. This is a more than decent approach for simple coroutines, and it requires very little free RAM or setup. However, keep in mind that `yield` only works in the top-level subroutine.
    
### Stackful

If we want to support yielding inside nested calls, we'll have to be clever. Some high-level languages like Rust and C++ support this in stackless coroutines or futures, but in these languages, the compiler transforms the entire control flow graph into a state machine. This isn't feasible in asm, and not writing state machines is why we went to coroutines in the first place, so we'll have to bring our own stack.

In smw, the stack generally won't grow tall enough to cover all of the space allocated for it, thus, you can put your own stack in that space. There are other large buffers unused by the game during the level, but Lunar Magic tends to eat all of them. $160 is a good place to start your coroutine's stack at.

```
stack_start = $160-2
yield:
    php
    pha
    phx
    phy
    jsr yield_raw
    ply
    plx
    pla
    plp
    rts
    
yield_raw:
    tsc
    sta coro_run_ptr
    lda coro_ret_ptr
    tcs
    rts

coro_init:
    lda #stack_start-1      ; initialize coro_run_ptr
    sta coro_run_ptr
    lda #coro_entrypoint-1  ; we're using rts to jump into this, so -1 is necessary
    sta stack_start
coro_run:
    tsc
    sta coro_ret_ptr
    sta coro_run_ptr
    tcs
    rts

coro_entrypoint:
    lda #$0010
    jsr wait_frames
    ; do other stuff
    ...

wait_frames:
-
    jsr yield
    dec
    bne -
    rts
```

This example really only uses this whole machinery to demonstrate waiting for 16 frames, but I hope this shows how powerful this setup can be, since these can be nested arbitrarily deep, and a single subroutine call can implement an entire sprite animation sequence. The Dram 3 Bowser uses this, for example, to encapsulate turning Bowser around into a single subroutine.

#### Cancellation

Generally, bosses can be hurt in any state, which may pose a challenge to a coroutine implementation. Dram 3 Bowser simply pauses coroutine execution for a couple frames, but you may want to advance the boss phase by changing what the boss is doing entirely. You may opt for a solution such as checking collision flags within your waiting or yield subroutine, but that wouldn't quite work if the wait is deeply nested. Instead, you should reset the coroutine state entirely to a known value:

```
coro_hurt:
    lda #stack_start-1      ; initialize coro_run_ptr
    sta coro_run_ptr
    lda #coro_hurt_entrypoint-1  ; we're using rts to jump into this, so -1 is necessary
    sta stack_start
```

This fully cancels whatever was being executed, so you have to make sure that you know that any yield point may never actually return.

As a sidenote, I did not do this in Dram 3 Bowser, so he will not "die" until he finishes its move. This is less of a bug and more of a nod to the Ridley fight from Super Metroid.


#### Microtasks

If you want a composable system to do something each frame while waiting within the main coroutine - for example, physics, animation or rendering, microtasks can supplement coroutines; they are, in essence, just a very minimalistic sprite system. You can implement them in a variety of ways, but here's a version using structs-of-arrays with stride 2:

```
run_utasks:
    ldx #$0000
-
    lda utask_ptrs,x
    beq +
    jsr (utask_ptrs,x)
+
    inx #2
    cpx #utask_len
    bne -
    rts

spawn_utask:
    pha
    ldx #$0000
-
    lda utask_ptrs,x
    bne +
    inx #2
    cpx #utask_len
    bne -
    ; all slots filled, crash loudly
    %unreachable()
+
    pla
    sta utask_ptrs,x
    rts

; usage
init:
    lda #utask_do_gravity
    jsr spawn_utask
    lda #$0030          ; gravity value
    sta utask_data0,x
    ; You can either use hardcoded offsets for
    ; less dynamic usecases, or stash X somewhere
    ; to maintain a reference to the task
    rts


utask_do_gravity:
    lda utask_data0,x
    clc : adc boss_vel_y
    sta boss_vel_y
    rts

; More complex object that behaves similarly to a stackless coroutine
utask_do_animation:
    lda #$0010
    sta utask_data0,x
    lda #+                  ; "yield point"
    sta utask_ptrs,x
+
    dec utask_data0,x
    lda #!some_value
    sta boss_animation
    lda utask_data0,x
    beq +
    rts
+
    stz utask_ptrs,x        ; despawn self
    rts

    
; RAM allocation
org <freeram>
    utask_len = 16
    utask_ptrs: skip 16
    utask_data0: skip 16
    utask_data1: skip 16
    ...
```

## Runtime-modifiable code

In cases where you need to squeeze out the highest possible performance, such as NMI or IRQ handling, self-modifying code or code generation may come in handy. Generally, self-modifying code changes itself, mostly changing operands to avoid complex addressing modes, while code generation produces code to be run at a later time (e.g. during NMI). Self-modifying code is quite useful for decompressors - while not explicitly time sensitive, they contain a lot of tight loops that would benefit from the speedup, or the `mvn` instruction, which has inline bank bytes that can only be made dynamic with self-modifying code. Code generation, on the other hand, can find a use in NMI handling, where DMA setup time must be kept to a minimum, which is what I'll demonstrate here.

```
macro offset(name)
    !offset_<name> = template_dma_rt_<name>-template_dma_rt+1
endmacro

template_dma_rt:
%offset(param)
    lda.w #$0000
    sta.w dma_param
%offset(a_addr)
    lda.w #$0000
    sta.w dma_a_addr
%offset(size)
    lda.w #$0000
    sta.w dma_size
%offset(vram_addr)
    lda.w #$0000
    sta.w vram_addr
    ldy.w #$01
    sty.w dma_enable
    rtl
.end:

code:
    rep #$30
    phb
    ; Use mvn to copy the template subroutine
    lda.w #template_dma_rt_end-template_dma_rt-1
    ldx.w #template_dma_rt   ; source
    ldy.w dyn_rt_ptr         ; pointer into the current dynamic routine
    mvn <:dyn_rt_space, <:template_dma_rt
    plb
    
    ; Modify parameters within the routine
    lda dyn_rt_ptr
    sta $00
    lda.w #<:dyn_rt_space
    sta $02
    lda input_param
    ldy.w #!offset_param
    sta [$00],y
    
    lda input_a_addr
    ldy.w #!offset_a_addr
    sta [$00],y
    
    lda input_size
    ldy.w #!offset_size
    sta [$00],y
    
    lda input_vram_addr
    ldy.w #!offset_vram_addr
    sta [$00],y
    
    ; Advance pointer to be right before the rtl (so we overwrite it with the next call)
    lda dyn_rt_ptr
    clc : adc.w #template_dma_rt_end-template_dma_rt-1
    sta dyn_rt_ptr
    
    rts
```










