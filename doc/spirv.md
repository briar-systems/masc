# SPIR-V text

The spir-v set serves `spirv`, run by the `vulkan` and `opencl` systems. Its
one syntax is `spirv`, generated from the core grammar. The text is a
module's instructions in module order, one per line, and states no section,
no header and no id bound. The header's version comes from the target's
selection, and the bound follows from the ids.

| fact          | SPIR-V                     |
|---------------|----------------------------|
| line comment  | `;`                        |
| separator     | the end of the line        |
| directives    | none                       |

## Instructions

An instruction is its opname and its operands in order. The result it
defines is bound before it, as `%r = OpTypeInt 32 0`. An optional operand is
written only when present, and a mask's members are joined by `|`.
Mnemonics and names read with their case.

- An id is `%` and its number or a name. A name takes, at its first sight,
  the lowest id no number of the text spells and no name took.
- A string is between double quotes, and a backslash escapes within it.
- An integer is decimal, after `0x`, or octal after a leading `0`.
- A float is decimal or a hex float with its `p` exponent, rounded once to
  its type's encoding. Infinity and NaN are spelled only as hex floats.
- There are no expressions, so `7 %5` is two operands.

The module states its capabilities and extensions as instructions of its
own. `OpCapability` adds its capability and `OpExtension` its extension to
the selection the instructions after it are read under.

## Example

```
; constants, a float and a string
OpCapability Shader
OpCapability Float64
OpMemoryModel Logical GLSL450
OpEntryPoint GLCompute %main "main"
OpExecutionMode %main LocalSize 8 1 1
OpName %main "main"
%void = OpTypeVoid
%f32 = OpTypeFloat 32
%f64 = OpTypeFloat 64
%half = OpConstant %f32 0.5
%big = OpConstant %f64 0x1p+10
%fn = OpTypeFunction %void
%main = OpFunction %void None %fn
%entry = OpLabel
OpReturn
OpFunctionEnd
```

```
$ masc assemble b.spvasm -t vulkan-spirv -o b.spv
assembled b.spvasm
  target    vulkan-spirv
  sections  1
  time      2.4ms
  output    b.spv
$ masc disassemble b.spv -t vulkan-spirv
note[disasm.unwritten] the text does not state trait spirv-version 1.6, which only a source states (b.spv)

	OpCapability Shader
	OpCapability Float64
	OpMemoryModel Logical GLSL450
	OpEntryPoint GLCompute %1 "main"
	OpExecutionMode %1 LocalSize 8 1 1
	OpName %1 "main"
	%2 = OpTypeVoid
	%3 = OpTypeFloat 32
	%4 = OpTypeFloat 64
	%5 = OpConstant %3 0.5
	%6 = OpConstant %4 1024
	%7 = OpTypeFunction %2
	%1 = OpFunction %2 None %7
	%8 = OpLabel
	OpReturn
	OpFunctionEnd
```

The printer writes every id by its number, so its text reads back to the
same ids. Assembling the printed text again gives a file of the same bytes.
A module that passes through mink's neutral object has its ids renumbered
densely, so it reads back to an equivalent module.

## Targets

`vulkan-spirv` defaults to the convention `shader` and a selection of
`spv1.6` with the capabilities the Vulkan environment names, which `-vv`
prints in full. The SPIR-V version is a trait of the object that the text
does not state, as the note above says.

The set classifies no row's timing, so `walk` refuses constant-time code of
it (see [walk](walk.md)).
