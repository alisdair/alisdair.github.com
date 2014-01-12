---
layout: post
title: "ARM immediate value encoding"
description: ARM instructions use 12 bits to encode a 32-bit immediate value, using a 4-bit rotate and 8-bit constant. This is a great idea.
---

The [ARM instruction set][arm-ia] encodes immediate values in an unusual way. It's typical of the design of the processor architecture: elegant, pragmatic, and quirky. Despite only using 12 bits of instruction space, the immediate value can represent a useful set of 32-bit constants.

[arm-ia]: http://en.wikipedia.org/wiki/ARM_architecture#Instruction_set

## What?

Perhaps I should start with some background. [Machine code][machine-code] is what computer processors run on: [binary][binary] representations of simple instructions. All ARM processors (like [the one in your iPhone][apple-a7], or [the other dozen in various devices][arm7tdmi] around your home) have 16 basic data processing instructions.

[machine-code]: http://en.wikipedia.org/wiki/Machine_code
[binary]: http://en.wikipedia.org/wiki/Binary_number
[apple-a7]: http://en.wikipedia.org/wiki/Apple_A7
[arm7tdmi]: http://en.wikipedia.org/wiki/ARM7TDMI

Each data processing instruction can work with several combinations of operands. For example, here are three different `ADD` instructions:

{% highlight arm %}
ADD r0, r2, r3          ; r0 = r2 + r3
ADD r0, r2, r3, LSL #4  ; r0 = r2 + (r3 << 4)
ADD r0, r2, #&4F0000    ; r0 = r2 + 0x4F0000
{% endhighlight %}

The first is easy to understand: add two registers, and store in a third. The second example shows the use of the barrel shifter, which can shift or rotate the second operand before performing the operation. This allows for some fairly complex single-instruction operations, and more importantly lots of fun optimising your assembler code.

But the instruction I want to describe in more detail here is the third and simplest one: adding a register to a constant value. This value is encoded in the instruction, so that it's immediately available.

## Immediate value encoding

ARM, like other [RISC][risc] architectures [MIPS][mips] and [PowerPC][powerpc], has a fixed instruction size of 32 bits. This is a good design decision, and it makes instruction decode and pipeline management much easier than with the variable instruction size of [x86][x86] or [680x0][68k]. However, it means that any instruction with an immediate value operand cannot represent a full 32-bit number.

[risc]: http://en.wikipedia.org/wiki/Reduced_instruction_set_computing
[mips]: http://en.wikipedia.org/wiki/MIPS_architecture
[powerpc]: http://en.wikipedia.org/wiki/PowerPC
[x86]: http://en.wikipedia.org/wiki/X86
[68k]: http://en.wikipedia.org/wiki/Motorola_68000

Here's the bit layout of an ARM data processing instruction:

![ARM instruction encoding for data processing instruction](/images/arm-immediate/arm.svg)

Any instruction with bits 27 and 26 as 00 is data processing. The four-bit opcode field in bits 24&ndash;21 defines exactly which instruction this is: add, subtract, move, compare, and so on. `0100` is `ADD`.

Bit 25 is the "immediate" bit. If it's 0, then operand 2 is a register. If it's set to 1, then operand 2 is an immediate value.

Note that operand 2 is only 12 bits. That doesn't give a huge range of numbers: 0&ndash;4095, or a byte and a half. Not great when you're mostly working with 32-bit numbers and addresses.

Compare it also to the equivalent instruction in MIPS, `addi`:

![MIPS instruction encoding for ADDI instruction](/images/arm-immediate/mips.svg)

Or the strangely backwards PowerPC, also called `addi`:

![PowerPC instruction encoding for ADDI instruction](/images/arm-immediate/powerpc.svg)

Both have 16-bit immediate values: 0&ndash;65535, or two bytes. This is much more reasonable. Unfortunately, because of the 4-bit condition field in every ARM instruction, the immediate field has to be smaller. And therefore less useful.

## The clever part

But ARM doesn't use the 12-bit immediate value as a 12-bit number. Instead, it's an 8-bit number with a [4-bit rotation][rotation], like this:

[rotation]: http://en.wikipedia.org/wiki/Circular_shift

![ARM immediate value encoding](/images/arm-immediate/arm-immediate-value-encoding.svg)

The 4-bit rotation value has 16 possible settings, so it's not possible to rotate the 8-bit value to any position in the 32-bit word. The most useful way to use this rotation value is to multiply it by two. It can then represent all even numbers from zero to 30.
     
To form the constant for the data processing instruction, the 8-bit immediate value is extended with zeroes to 32 bits, then rotated the specified number of places to the right. For some values of rotation, this can allow splitting the 8-bit value between bytes. See the table below for all possible rotations.

![ARM immediate value rotations](/images/arm-immediate/rotations.svg)

## Examples

The rotated byte encoding allows the 12-bit value to represent a much more useful set of numbers than just 0&ndash;4095. It's occasionally even more useful than the MIPS or PowerPC 16-bit immediate value.

ARM immediate values can represent any power of 2 from 0 to 31. So you can set, clear, or toggle any bit with one instruction:

{% highlight arm %}
ORR r5, r5, #&8000     ; Set bit 15 of r5
BIC r0, r0, #&20       ; ASCII lower-case to upper-case
EOR r9, r9, #&80000000 ; Toggle bit 31 of r9
{% endhighlight %}

More generally, you can specify a byte value at any of the four locations in the word:

{% highlight arm %}
AND r0, r0, #&ff000000 ; Only keep the top byte of r0
{% endhighlight %}

In practice, this encoding gives a lot of values that would not be available otherwise. Large loop termination values, bit selections and masks, and lots of other weird constants are all available.

But what I find really compelling is the inventiveness of the design. Faced with the constraint of only having 12 bits to use, the ARM designers had the insight to reuse the idle [barrel shifter][barrel-shifter] to allow a wide range of useful numbers. To my knowledge, no other architecture has this feature. It's unique.

[barrel-shifter]: http://en.wikipedia.org/wiki/Barrel_shifter

<h2 id="play-with-it">Play with it</h2>

Here's an interactive version of an ARM assembler's immediate value encoder. Requires JavaScript and a modern browser.

Choose an immediate value and see its encoding. See which values can't be encoded. Rotate the constant to see what happens.

<p id="encoding_examples">Try these examples: <a href="#">0x3FC00</a>, <a href="#">0x102</a>, <a href="#">0xFF0000FF</a>, <a href="#">0xC0000034</a></p>

<form id="arm_immediate_encoding_form"><div class="button button_left"><button id="left_button">&lt;&lt;</button></div><div class="button button_left"><button id="right_button">&gt;&gt;</button></div><div class="button button_right"><button id="plus_button">+</button></div><div class="button button_right"><button id="minus_button">&minus;</button></div><div class="input"><input id="number_input" value="0x000000D7"></div></form>

{% raw %}
<svg width="642px" height="56px" viewBox="0 0 642 56" version="1.1" xmlns="http://www.w3.org/2000/svg" style="font-family: Menlo, DejaVu Sans Mono, Consolas, Courier">
<g stroke="none" stroke-width="1" fill="none" fill-rule="evenodd">
<g font-weight="normal" fill="#000000" font-size="10">
<g transform="translate(5, 9)">
<text x="0">31</text><text x="20">30</text><text x="40">29</text><text x="60">28</text><text x="80">27</text><text x="100">26</text><text x="120">25</text><text x="140">24</text><text x="160">23</text><text x="180">22</text><text x="200">21</text><text x="220">20</text><text x="240">19</text><text x="260">18</text><text x="280">17</text><text x="300">16</text><text x="320">15</text><text x="340">14</text><text x="360">13</text><text x="380">12</text><text x="400">11</text><text x="420">10</text><text x="443">9</text><text x="463">8</text><text x="483">7</text><text x="503">6</text><text x="523">5</text><text x="543">4</text><text x="563">3</text><text x="583">2</text><text x="603">1</text><text x="623">0</text>
</g>
</g>
<g transform="translate(0, 15)">
<rect fill="#F7F7F7" x="1" y="0" width="640" height="40"></rect>
<g id="decoded_bits" fill="#000000" font-weight="normal" font-size="14"><g transform="translate(0,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(20,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(40,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">0</text></g><g transform="translate(60,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(80,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">0</text></g><g transform="translate(100,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(120,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(140,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(160,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(180,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(200,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">0</text></g><g transform="translate(220,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(240,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">0</text></g><g transform="translate(260,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(280,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(300,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(320,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(340,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(360,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">0</text></g><g transform="translate(380,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(400,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">0</text></g><g transform="translate(420,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(440,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(460,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(480,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(500,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(520,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">0</text></g><g transform="translate(540,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(560,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">0</text></g><g transform="translate(580,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(600,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g><g transform="translate(620,0)"><rect x="0" y="0" width="20" height="40"></rect><text x="7" y="25">1</text></g></g>
<g transform="translate(1, 0)" stroke="#000000" stroke-width="2">
<g transform="translate(19, 1)" stroke-opacity="0.2" stroke-dasharray="5,28,5,28">
<path d="M1,0 L0,38"></path>
<path d="M21,0 L20,38"></path>
<path d="M41,0 L40,38"></path>
<path d="M61,0 L60,38"></path>
<path d="M81,0 L80,38"></path>
<path d="M101,0 L100,38"></path>
<path d="M121,0 L120,38"></path>
</g>
<g transform="translate(179, 1)" stroke-opacity="0.2" stroke-dasharray="5,28,5,28">
<path d="M1,0 L0,38"></path>
<path d="M21,0 L20,38"></path>
<path d="M41,0 L40,38"></path>
<path d="M61,0 L60,38"></path>
<path d="M81,0 L80,38"></path>
<path d="M101,0 L100,38"></path>
<path d="M121,0 L120,38"></path>
</g>
<g transform="translate(340, 1)" stroke-opacity="0.2" stroke-dasharray="5,28,5,28">
<path d="M1,0 L0,38"></path>
<path d="M21,0 L20,38"></path>
<path d="M41,0 L40,38"></path>
<path d="M61,0 L60,38"></path>
<path d="M81,0 L80,38"></path>
<path d="M101,0 L100,38"></path>
<path d="M121,0 L120,38"></path>
</g>
<g transform="translate(499, 1)" stroke-opacity="0.2" stroke-dasharray="5,28,5,28">
<path d="M1,0 L0,38"></path>
<path d="M21,0 L20,38"></path>
<path d="M41,0 L40,38"></path>
<path d="M61,0 L60,38"></path>
<path d="M81,0 L80,38"></path>
<path d="M101,0 L100,38"></path>
<path d="M121,0 L120,38"></path>
</g>
<g transform="translate(159, 1)" stroke-opacity="0.5" stroke-dasharray="8,24,8,24">
<path d="M1,0 L0,38"></path>
<path d="M161,0 L160,38"></path>
<path d="M321,0 L320,38"></path>
</g>
<rect x="0" y="0" width="640" height="40"></rect>
</g>
</g>
</g>
</svg>

<svg width="244px" height="86px" viewBox="0 0 244 86" version="1.1" xmlns="http://www.w3.org/2000/svg" style="font-family: Menlo, DejaVu Sans Mono, Consolas, Courier">
<g stroke="none" stroke-width="1" fill="none" fill-rule="evenodd">
<g transform="translate(3, 9)" font-weight="normal" fill="#000000" font-size="10">
<text x="3">11</text><text x="23">10</text><text x="46">9</text><text x="66">8</text><text x="86">7 </text><text x="106">6</text><text x="126">5</text><text x="146">4</text><text x="166">3</text><text x="186">2</text><text x="206">1</text><text x="226">0</text>
</g>
<rect fill="#F7F7F7" x="1" y="15" width="240" height="40"></rect>
<rect fill-opacity="0.5" fill="#F8E71C" x="1" y="15" width="80" height="40"></rect>
<rect fill-opacity="0.5" fill="#F5A623" x="81" y="15" width="160" height="40"></rect>
<g id="encoded_bits" transform="translate(0, 15)" font-weight="normal" fill="#000000" font-size="14"><g class="bit_11" transform="translate(0,0)"><text x="7" y="25">1</text></g><g class="bit_10" transform="translate(20,0)"><text x="7" y="25">0</text></g><g class="bit_9" transform="translate(40,0)"><text x="7" y="25">1</text></g><g class="bit_8" transform="translate(60,0)"><text x="7" y="25">1</text></g><g class="bit_7" transform="translate(80,0)"><text x="7" y="25">1</text></g><g class="bit_6" transform="translate(100,0)"><text x="7" y="25">1</text></g><g class="bit_5" transform="translate(120,0)"><text x="7" y="25">0</text></g><g class="bit_4" transform="translate(140,0)"><text x="7" y="25">1</text></g><g class="bit_3" transform="translate(160,0)"><text x="7" y="25">0</text></g><g class="bit_2" transform="translate(180,0)"><text x="7" y="25">1</text></g><g class="bit_1" transform="translate(200,0)"><text x="7" y="25">1</text></g><g class="bit_0" transform="translate(220,0)"><text x="7" y="25">1</text></g></g>
<g transform="translate(20, 16)" stroke="#000000" stroke-opacity="0.2" stroke-width="2" stroke-dasharray="5,28,5,28">
<path d="M1,0 L0,38"></path>
<path d="M21,0 L20,38"></path>
<path d="M41,0 L40,38"></path>
</g>
<g transform="translate(100, 16)" stroke="#000000" stroke-opacity="0.2" stroke-width="2" stroke-dasharray="5,28,5,28">
<path d="M1,0 L0,38"></path>
<path d="M21,0 L20,38"></path>
<path d="M41,0 L40,38"></path>
<path d="M61,0 L60,38"></path>
<path d="M81,0 L80,38"></path>
<path d="M101,0 L100,38"></path>
<path d="M121,0 L120,38"></path>
</g>
<g transform="translate(80, 16)" stroke="#000000" stroke-opacity="0.5" stroke-width="2" stroke-dasharray="8,24,8,24">
<path d="M1,0 L0,38"></path>
</g>
<g transform="translate(0, 84)" font-weight="normal" fill="#000000" font-size="14">
<text id="encoded_shift" x="30">0xB</text>
<text id="encoded_immediate" x="140">0xD7</text>
</g>
<rect stroke="#000000" stroke-width="2" x="1" y="15" width="240" height="40"></rect>
</g>
</svg>
{% endraw %}

<p id="encoder_output">Encoding requires JavaScript.</p>

{% raw %}
<script type="text/javascript">
(function(){
function rol(n, i) { return ((n << i)|(n >>> (32 - i))) >>> 0; }
function ror(n, i) { return ((n >>> i)|(n << (32 - i))) >>> 0; }
function hex(n, l) { var h = n.toString(16); while (h.length < l) h = "0" + h; return "0x" + h.toUpperCase(); };
function encode(n) { var i, m; for (i = 0; i < 16; i++) { m = rol(n, i * 2); if (m < 256) return (i << 8) | m; }; throw "Unencodable constant"; }
function modify(f) { var input = document.getElementById("number_input"); input.value = hex(f(parseInt(input.value)), 8); update(); }
function encoded_bits(n) { var i, e, shift = (n >>> 8) & 0xF, immediate = n & 0xFF, g = document.getElementById("encoded_bits"); for (i = 0; i < g.childNodes.length; i++) { g.childNodes[i].childNodes[0].textContent = ((n >>> (11 - i)) & 1).toString(); } document.getElementById("encoded_shift").textContent = hex(shift, 1); document.getElementById("encoded_immediate").textContent = hex(immediate, 1); decode = ror(immediate, shift * 2); e = document.getElementById("encoder_output"); e.textContent = hex(immediate, 1) + " ROR " + (shift * 2) + " = " + decode; e.className = ""; }
function bits(n) { var i, g = document.getElementById("decoded_bits"); for (i = 0; i < 32; i++) { g.childNodes[i].childNodes[1].textContent = ((n >>> (31 - i)) & 1).toString(); } }
function span(shift) { var i, bits = ror(0xFF, shift * 2), g = document.getElementById("decoded_bits"), r; for (i = 0; i < 32; i++) { r = g.childNodes[i].childNodes[0]; if (bits & (1 << (31-i))) { r.className = "span"; } else { r.className = ""; } } }
function encode_error() { var i, g = document.getElementById("encoded_bits"); for (i = 0; i < g.childNodes.length; i++) { g.childNodes[i].childNodes[0].textContent = " "; } document.getElementById("encoded_shift").textContent = "---"; document.getElementById("encoded_immediate").textContent = "-----"; }
function display_error(s) { var e = document.getElementById("encoder_output"); e.textContent = s; e.className = "error"; }
function fill_span(value) { var i, b, r = 0, m = 0xffffffff, n = m; for (i = 0; i < 32; i++) { n = rol(value, i); if (n < m) { r = i; m = n; }}; m--; m |= m >>> 1; m |= m >>> 2; m |= m >>> 4; m |= m >>> 8; m |= m >>> 16; m = m >>> 0; if (m > 0xff) { display_error("Rotated constant is too wide"); } else { display_error("Constant requires odd rotation"); } return ror(m, r); }
function span_error(value) { var i, bits = fill_span(value), g = document.getElementById("decoded_bits"), r; for (i = 0; i < 32; i++) { r = g.childNodes[i].childNodes[0]; if (bits & (1 << (31-i))) { r.className = "error"; } else { r.className = ""; } } }
function update() { var input = document.getElementById("number_input"); var value = parseInt(input.value); bits(value); try { var encoded = encode(value), shift = (encoded >>> 8) & 0xF; encoded_bits(encoded); input.classList.remove("error"); span(shift); } catch (e) { input.classList.add("error"); encode_error(); span_error(value); } }
var form = document.getElementById("arm_immediate_encoding_form");
form.onsubmit = function() { return false; };
document.getElementById("left_button").onclick = function() { modify(function(n) { return rol(n, 2); }) };
document.getElementById("right_button").onclick = function() { modify(function(n) { return ror(n, 2); }) };
document.getElementById("plus_button").onclick = function() { modify(function(n) { return n + 1; }) };
document.getElementById("minus_button").onclick = function() { modify(function(n) { return n - 1; }) };
document.getElementById("number_input").onchange = update;
document.getElementById("number_input").onkeyup = update;
update();
var examples = document.getElementById("encoding_examples").children;
var i;
function example() { document.getElementById("number_input").value = hex(parseInt(this.textContent), 8); update(); return false; }
for (i = 0; i < examples.length; i++) { examples[i].onclick = example; }
})();
</script>
{% endraw %}
