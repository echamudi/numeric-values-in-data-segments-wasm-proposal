# WebAssembly Proposal: Numeric Values in WAT Data Segments

This proposal proposes the ability of writing integers and floating point values in the data segments of WebAssembly Text Format (WAT).
This document is the summarization of [this issue](https://github.com/WebAssembly/design/issues/1348) in WebAssembly design discussion repo.

**Try live demo :** https://wasmprop-numerical-data.netlify.app/wat2wasm

**Updates:**

9/6/2020: This proposal has been presented in [9/6/2020 WebAssembly CG Meeting](https://github.com/WebAssembly/meetings/blob/a55ad5c884653db19d81f3684a8026c01052a24b/main/2020/CG-06-09.md#discuss-numerical-values-in-data-segments-proposal-discussion-link-and-semi-formal-description-repo).

12/6/2020: Added [data alignment](#data-alignment), [out of range values](#out-of-range-values), and [wasm2wat translation](#wasm2wat-translation) subsection. These items were asked during the meeting.

23/6/2020: Advanced to phase 1 in [23/6/2020 WebAssembly CG Meeting](https://github.com/WebAssembly/meetings/blob/8630cd6ad2c34fc7f26537f78e04c0a79e5e53a1/main/2020/CG-06-23.md#poll-on-general-interest-in-this-proposal-and-advancing-to-phase-1)

26/6/2020: Official [spec fork repo](https://github.com/WebAssembly/wat-numeric-values) is made.

## Current State

In the current grammar, it allows us to write the data segments only in the form of strings.

```ebnf
data ::= '(' 'data' x:memidx '(' 'offset' e:expr ')' b*:datastring ')'

datastring ::= (b*:string)*
```

For example:

```wat
(data (offset (i32.const 0)) "abc")
```
```wat
(data (offset (i32.const 0)) "a" "b" "c")
```

This rule is good enough for writing textual data to data segment.

The problem comes when we want to enter numerical values. We have to encode the numbers manually in advance and write it in string by using escape character (`\`).

Here's an example of data initialization from a [raytracer](https://github.com/binji/raw-wasm/blob/499bbff77564047f7d73332e18cb5a121ceb8f2e/raytrace/ray.wat#L6-L12) code by Ben Smith.

```wat
(data (i32.const 0)
  "\00\00\00\00" ;; ground.x
  "\00\00\c8\42" ;; ground.y
  "\00\00\00\00" ;; ground.z
  "\00\00\c2\42" ;; ground.r
  "\3b\df\6f\3f" ;; ground.R
  "\04\56\8e\3e" ;; ground.G
  "\91\ed\bc\3e" ;; ground.B
)
```

The data above are actually f32 numbers. This approach feels hacky, and it's not easy to see the values of those float values from a single glance.

## Survey

x86-64 GCC and x86-64 Clang have [assembler directives](https://ftp.gnu.org/old-gnu/Manuals/gas-2.9.1/html_chapter/as_7.html) that can be used to save arbritary number to data segments.

Some of them are:

```s
data:
  .ascii  "abcd"         #; 61 62 63 64
  .byte   1, 2, 3, 4     #; 01 02 03 04
  .2byte  5, 6           #; 05 00 06 00
  .4byte  0x89ABCDEF     #; EF CD AB 89
  .8byte  -1             #; FF FF FF FF FF FF FF FF
  .float  62.5           #; 00 00 7A 42
  .double 62.5           #; 00 00 00 00 00 40 4F 40
```

In NASM, there are [pseudo-instructions](http://www.tortall.net/projects/yasm/manual/html/nasm-pseudop.html), for example:

```asm
data:
  db          'abcd', 0x01, 2, 3, 4   ; 61 62 63 64 01 02 03 04
  dw          5, 6                    ; 05 00 06 00
  dd          62.5                    ; 00 00 7A 42
  dq          62.5                    ; 00 00 00 00 00 40 4F 40
  times 4 db  0xAB                    ; AB AB AB AB
```

These directives help programmers to write and see the data in the code directly in human readable format rather than the encoded format.

## Proposed Changes

### Grammar

After a discussion in [the issue thread](https://github.com/WebAssembly/design/issues/1348), we've come to a possible enhanced grammar for data segments in WAT, roughly:

```ebnf
data ::= '(' 'data' memidx '(' 'offset' expr ')' dataval* ')'

dataval ::= string
        | numvec

numvec ::= '(' 'i8'  i8* ')'
        | '(' 'i16' i16* ')'
        | '(' 'i32' i32* ')'
        | '(' 'i64' i64* ')'

        | '(' 'f32' f32* ')'
        | '(' 'f64' f64* ')'
```

It should also be available in the inline data segment in a memory module:

```ebnf
mem ::= '(' 'memory' id '(' 'data' dataval* ')' ')'
      |  ...
```

Usage examples:

```wat
;; XYZ coordinate points 
(data (offset (i32.const 0))
  (f32 0.2 0.3 0.4)
  (f32 0.4 0.5 0.6)
  (f32 0.4 0.5 0.6)
)

;; Writing 1001st ~ 1010th prime number
(data (offset (i32.const 0x100))
  (i16 7927 7933 7937 7949 7951 7963 7993 8009 8011 8017)
)

;; PI
(data (offset (i32.const 0x200))
  (f64 3.14159265358979323846264338327950288)
)

;; Inline in memory modules
(memory (data (i8 1 2 3 4)))
```

And the early raytracer example can be rewritten as:

```wat
(data (offset (i32.const 0))
  (f32 0 100.0 0 97.0)     ;; ground.{x,y,z,r}
  (f32 0.937 0.278 0.369)  ;; ground.{R,G,B}
)
```

By using this updated grammar, it's easy for WAT programmers to input the data directly in integers and floating point formats.

### Execution

The conversion of numvec to data in data segments happens during the wat2wasm compilation. 
Which means there is no change needed in the binary format, execution spec, or the structure spec.

So, the following two snippents:

```wat
...
(memory 1)
(data (offset (i32.const 0))
  "abcd"
  (i16 -1)
  (f32 62.5)
)
...
```
```wat
...
(memory 1)
(data (offset (i32.const 0))
  "abcd"
  "\FF\FF"
  "\00\00\7a\42"
)
...
```

will output the same binary code:

```
...
; data segment header 0
0000010: 00                                        ; segment flags
0000011: 41                                        ; i32.const
0000012: 00                                        ; i32 literal
0000013: 0b                                        ; end
0000014: 0a                                        ; data segment size
; data segment data 0
0000015: 6162 6364 ffff 0000 7a42                  ; data segment data
000000e: 10                                        ; FIXUP section size
...
```

#### Encoding

The encoding should use two's complement for integers and IEEE754 for float, which is similar to the `t.store` memory instructions.

This encoding is used to make sure that when we load the value from memory using the `load` memory instructions, the value will be consistant whether the data was stored by using `(data ... )` initialization or `t.store` instructions.

#### Data Alignment

Unaligned placements are allowed. For example:

```wat
...
(memory 1)
(data (offset (i32.const 0))
  (i8 1)    ;; will go to address 0
  (i16 2)   ;; will go to address 1
)
...
```
compiles to: `0102 00`

#### Out of Range Values

Out of range values should throw error during wat2wasm compilation.

```wat
(memory 1)
(data (offset (i32.const 0))
  (i8 256)        ;; Error
  (i8 -129)       ;; Error
)
```

#### wasm2wat Translation

The data segments in the compiled binary do not contain any information about their original form in WAT state.
Therefore, the translation from the binary format back to the text format will use the default string form.

### Backward Compatibility

As the proposed grammar still accepts the string form, all existing WAT codes should work fine.

## Author

- Ezzat Chamudi - [echamudi](https://github.com/echamudi)

This proposal also incorporates suggestions from Ben Smith ([binji](https://github.com/binji)), Jacob Gravelle ([jgravelle-google](https://github.com/jgravelle-google)), and Andreas Rossberg ([rossberg](https://github.com/rossberg)).
