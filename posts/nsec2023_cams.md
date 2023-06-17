# nsec 2023: Office Surveillance

> Since I'm writing quite late after the event, the details of the challenge might be a little
> bit wrong and my memory fuzzy, but it matters very little. Enjoy :)

The premise of this year's nsec involved us being lowly employees at a "big corp",
something that none of us can relate to. Being spied by our corporate overlords, 
we needed to act out by hacking our way out, basically.

## The challenge

Cameras were (in this make belief world) installed at our workstations to make sure we 
performed as required and were not distracted by whatever. Peek productivity must be
preserved. Since this is an unacceptable privacy violation, we needed to disable said
cameras.

We were provided an URL and a port. This was a secret communication channel with someone
that could provide us with the firmware of the cameras. Once we had the firmware, we
had to figure out the camera password to disable them. Oh, and since this is a secret
communication channel, it would be closed by the server after ~2 minutes. So you had
to be fast. No manual work here.

I started this challenge late day 2 out 3 and finished it few minutes before the
end countdown. Needless to say, I rushed this quite a bit.

## The solve

This was not trivial. The first thing I did was acquire a firmware for
exploration. By whispering a sweet sentence to our secret lover on the other
side of this socket, he would provide us with a *fresh* firwmare. This means
that every time we attempt to solve the challenge, the firmware is different
(this is where the automatic reverse enginnering part comes in, alongside the 2
minutes constraint).

```python
URL = "9000:ff:1ce:ff:216:3eff:fe8c:4a0c"
PORT = 8000
PASSWORD = "I have come for the shadow training\r\n"
LOCAL_PATH = "/home/syvon/Desktop/nsec/fmw3.elf"
ACTUAL_CONNECT = not True


if ACTUAL_CONNECT:
	s = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
	s.settimeout(5)
	s.connect((URL, PORT))
else:
	s = None

# Read question prompt

def sendmsg(msg: str, eol="\r\n") -> int:
	return s.send(f"{msg}{eol}".encode())

def read_until_line() -> str:
	str_buff = ""

	try:
		while True:
			b = s.recv(1).decode()
			str_buff += b
			if b == "\n":
				break
        except socket.timeout:
            ...

        return str_buff

if ACTUAL_CONNECT:
	_prompt = read_until_line()

	s.send(PASSWORD.encode())

	# Read warning
	_warning = read_until_line()

	sendmsg("")  # ok

	# Read firmware as a string
	fmw_b64_s = read_until_line().strip()

	raw_fmw = base64.b64decode(fmw_b64_s, validate=True)
```

The firmware was base64 encoded and the secret agent on the otherside of the
channel expected a base64 encoded answer as well. Notice that I use raw sockets
and basically sanitize everything by hand. nsec would have been a lot easier
for me if I knew more about [pwntools](http://docs.pwntools.com). Oh well,
we'll see at hackfest.

Anyways. I now have a firwmare. This is what it looks like once dissasembled:

```C
undefined8 main(void)

{
  uint local_1c;
  uint local_18;
  uint local_14;
  int local_10;
  int local_c;
  
  __isoc99_scanf(&DAT_00402004,&local_1c);
  __isoc99_scanf(&DAT_00402004,&local_18);
  __isoc99_scanf(&DAT_00402004,&local_14);
  __isoc99_scanf(&DAT_00402004,&local_10);
  __isoc99_scanf(&DAT_00402004,&local_c);
  if (local_1c % 0x55380c8a != 0x40f3c813) {
    puts("Wrong password.");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  if (local_18 % 0x2fb1bc7 != 0x2c23061) {
    puts("Wrong password.");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  if (local_14 % 0x1258e69c != 0x86e4e46) {
    puts("Wrong password.");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  if (local_10 != -0x3fbdb917) {
    puts("Wrong password.");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  if (local_c != -0x2af9c156) {
    puts("Wrong password.");
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  puts("Deactivated.");
  return 0;
}
```

That does not look *so* bad. The scanf format specifier wants 10 characters
everytime to make a uint (`%10u`). The password is thus a 50 character string
of digits that must satisfy two types of criteria. A modulo one, which means
there are possibly more than a single valid value (which will come back to bite
me in the ass) and an equality condition, which is simple enough to fullfill.

- For the equality, we simply provide the value wanted. We can go fetch it in the
  code. It's all uints, but for some reason ghidra displayed it negative. At the end
  of the day however, this is all binary numbers strings. (I also don't know how to
  use ghidra like a real NSA spy).

- For the modulo, the strategy would be simple. This is where my first mistake started.
  I'd take a number that satisfies the condition and go with it, assuming the challenge
  would be solved if I could exit with code 0. Anyways, with `x % a != b`, `x` has to
  be `n * a + b` for it to pass. If `b < a`, we can use `n = 0`.

Looking at the dissasembly (in x86-64 asm) for the modulo part, it seems straigtforward
enough to parse:

```asm
        004011c8 8b 45 ec        MOV        EAX,dword ptr [RBP + local_1c]
        004011cb b9 8a 0c        MOV        ECX,0x55380c8a
                 38 55
        004011d0 31 d2           XOR        EDX,EDX
        004011d2 f7 f1           DIV        ECX
        004011d4 81 fa 13        CMP        EDX,0x40f3c813
                 c8 f3 40
```

The values are right there for picking. Great!

For the equality part, I wasted a bunch of time with extracting the values:

```asm
        00401252 05 68 86        ADD        EAX,0xab2d8668
                 2d ab
        00401257 3d 51 cd        CMP        EAX,0x6b6fcd51
                 6f 6b
        0040125c 74 19           JZ         LAB_00401277
        0040125e 48 bf 09        MOV        RDI,s_Wrong_password._00402009                   = "Wrong password."
                 20 40 00 
                 00 00 00 00
        00401268 e8 c3 fd        CALL       <EXTERNAL>::puts                                 int puts(char * __s)
                 ff ff
        0040126d bf 01 00        MOV        EDI,0x1
                 00 00
        00401272 e8 d9 fd        CALL       <EXTERNAL>::exit                                 void exit(int __status)
                 ff ff
                             -- Flow Override: CALL_RETURN (CALL_TERMINATOR)
```

Notice that the `cmp` value (the `!=` part) is right at instruction `00401257`, which
is simply enough to extract. At first, I did not notice that there is a sneaky little
`ADD` instruction right before, which ghidra "takes care" of simplifying for us. This cost
me a **lot** of time (second mistake!).

Trying out a manually crafted string on this firmware, I was able to solve it
manually without too much trouble, hooray!

### Automatically doing this

Again, `pwntools` would have been great to know more about, because it provides
easier wrappers for all you are about to see. I did know about `subprocess` and
`objdump` however :D

```python
	proc_ret = subprocess.run(["objdump", LOCAL_PATH, "-d"], capture_output=True)
	dis = proc_ret.stdout.decode()

	main_section = extract_main(dis)
	logic_blocks = extract_logic_blocks(main_section)
	numbers = list(filter(len, map(logic_block_to_number, logic_blocks)))
```

I basically open an `objdump` process, extract the output to get basically what
ghidra sees, get the `main` block, split it in logical parts (every `if`
condition in the C snippet) and get valid numbers. Some logical blocks do not
map to numbers so I filter them out. Apparently there's a `ghidra`-headless
mode that I could have used, but I don't know anything about that.

> Also `gdb` could have provided a clean output as I discovered from another solve:
> `gdb -batch -ex 'file fmw1.elf' -ex 'disassemble main' > run.asm`

I can then send back the answer to the server:

```python
    for i, num in enumerate(numbers):
        print(f"{i + 1}. {num} ({len(num)})")

    answer = "".join(numbers)

    print(f"[AS]=> {answer} ({len(answer)})")

    answer_64 = base64.b64encode(answer.encode())

    print(f"[64]=> {answer_64}")

    # Encode in b64 and send
    if ACTUAL_CONNECT:
        sendmsg(answer_64.decode())

        # Read response from server
        print(read_until_line())
        print(read_until_line())

```

Let's explore how we actually extract things. Extracting the main section is
straight forward enough:

```python
def extract_main(main_dis: str):
    start = "<main>"
    end = ".fini"

    start_i = main_dis.index(start)
    end_i = main_dis.index(end)

    sub = main_dis[start_i: end_i + 1]
    lines = sub.splitlines()[1:-1]

    main_dis = "\n".join(lines)
    if DEBUG:
        print("MAIN DIS")
        print(main_dis)

    return main_dis
```

Basically find the "<main>" identifier and the next ".fini" part and grab that. This
gives us the entire main function in x86-64 (in `objdump` output format).

Next up, we look at the logical blocks that we can extract:

```python
def extract_logic_blocks(main_section: str) -> List[str]:
    zero_block = "00 00 00"

    blocks = []

    s = 0
    lines = main_section.splitlines()
    for i, line in enumerate(lines):
        if line.strip().endswith(zero_block):
            blocks.append(
                "\n".join(lines[s:i])
            )
            s = i

    return blocks
```

We split by `00 00 00`. I don't remember exactly what that divides, I think it might be the
`exit` syscall. Either way, splitting over the zero block worked, but returned a section
or two which had no logic to extract (thus the aforementionned `len` filter).

> After a bit of looking, it's simply the end of the `mov` addresses, so it splits right
> before the `puts` calls.

### The actual logic extraction

Every block is then mapped to this function:

```python
def logic_block_to_number(logic_block: str) -> str:
    if "cmp" not in logic_block:
        return ""

    # mod block
    if "div" in logic_block:
        r = parse_div_block(logic_block)
    else:
        r = parse_neq_block(logic_block)

    l = len(r)

    if l > 0:
        r = "0" * (10 - l) + r

    return r
```

As you can see, if no `cmp` instruction is contained in the block, there's no
analysis to be done, so we can skip it. Next, we check if there is a `div`
instruction. This is what makes the `modulo` operation (`div` computes the
integer division and the remainder, all in one instruction).

```python
def parse_neq_block(neq_block: str) -> str:
    add_value = _parse_int32(ADD_REGEX.findall(neq_block)[0])
    cmp_value = _parse_int32(CMP_REGEX.findall(neq_block)[0])

    num = np.uint32(cmp_value - add_value)

    value_as_str = str(int(num))
    return value_as_str
```

This parses the `neq` blocks to find the proper number to feed. Since there's a
sneaky `add`, we also find it. We can compute what ghidra shows us by simply
subtracting the two numbers. Notice how I do perform this with numpy. This was
a real shitshow. Python is great for people that don't care about fixed integer
precision and signedness. However, when you do, you will suffer. Since
everything needed to be unsigned, I had to provide positive numbers. However,
computing the two's complement did not work; because of the "infinite" number
of trailing zeros (in retrospec, just `& 0xFFFFFFFF`) would have worked). I
wasted a bunch of time on this as well, but in the end `numpy` came in to the
rescue with a simple enough solution.

The poorly named function `_parse_int32` actually parses a `uint32`:

```python
def _parse_int32(subs) -> int:
    if len(subs) != 8:
        missing = 8 - len(subs)
        subs = "0" * missing + subs

    parts = [
        int(subs[i:i + 2], 16) for i in range(0, 8, 2)
    ]

    return int.from_bytes(bytes(parts), byteorder='big', signed=False)
```

Again, there's probably a better way of doing this but when you've got little time,
you do what you got to do. 

> Now that I write this, this can be replaced with `int(sub, 16)` , oops!
> I went through a bunch of iterations because I was fighting mistake #2
> in the wrong places.

```python
def parse_div_block(div_block: str) -> str:
    mod = _parse_int32(MOD_REGEX.findall(div_block)[0])
    neq = _parse_int32(CMP_REGEX.findall(div_block)[0])

    if mod > neq:
        add = neq
    else:
        add = np.int32(mod + neq)

    add_as_str = str(add)

    return add_as_str
```

The `div` block is actually quite easy to parse, with a little bit of `np`'s help.

All the regexes used are fairly simple and match the output of my local `objdump`:

```python
ADD_REGEX = re.compile(r"add    \$0x(.*),%eax")
CMP_REGEX = re.compile(r"cmp    \$0x(.*),")
MOD_REGEX = re.compile(r"mov    \$0x(.*),%ecx")
```

And this is it! When I assembled it all, I got a tiny script and was rewarded with a 
flag worth a meaningful amount of points, right?

No.

Unfortunately, it seems I misunderstood a part of the challenge. While my
script was indeed finding a correct solution for every firmware thrown my way,
I was never able to get the points. Every 50 characters input string produced
by my script would reach the mighty `exit 0`. Base64 encoding the answer back
was easy enough, and I've sent data over a socket, so I don't think that was
the issue.

Looking at the official solve, it seems the problem lies with the fact that I
compute a **single** solution for every modulo. Other solutions compute a bunch
and send them all in sequence, until the secret agent at the other end of the
line accepts one of them. It's a bit frustrating, but ultimately minor changes
to my solution can compute more modulo alternatives. Speaking with the
challenge creator afterwards, he mentionned that any modulo alternative should
have worked. This also leads me to believe that I might just needed to solve
the firmware for more than a single camera in a row until I got a firmware. 
When replying with the answer, the agent said something along the lines of
"try again", so I wasn't sure if I had to do it again for the same firwmare,
or on another firmware (every "try again" message was followed by another
firmware, but I recall it looking similar in base64).

I guess I'll never know :)

What really angered me however, is the official solve. The problem is basically
free if you know about this one tool, appropriately named
[`angr`](https://angr.io/). It performs exactly what I did here, but as a
general constrained problem. Given a set of valid input strings (expressed as
constraints), it will solve the constraints to give you an input that reaches
the desired state of the program, in this cased a "Deactivated" output string.

Ouch.


## Reflecting

Overall, this is a pretty amateurish solve in the CTF world (technically, it
does not even work!). However, I'm quite satisfied, this was an involved
challenge without `angr`. There was nothing outlandishly complicated with this
challenge, just a lot of tiny annoying parts, especially doing this under the
constraints of the event (ie, tired) and doing this without any tool to
facilitate.

The binary extraction part definitely could have been better. With a better
dissasembly than what is provided by `objdump` it might have been cleaner to
extract the relevant parts. Getting rid of the `_parse_int32` would also make
it cleaner. But like someone at the event said: "it's a CTF 🙂
hacky/botchy/suboptimal approaches that work are what makes it amazing 😄"
(however mine does not really work hehe)

I also did a lot of useless validation on what I answered back. (Length check
and all). This isn't prod, this is a CTF! The proper solution should have
gotten the right answer. Again, I was fighting the server not accepting my
answers, so I was validating everything.

Next time I'll use `pwntools` and `angr`!

Here is the full program. It never checks for a flag in an answer because it does
not repeat. I expected the flag to come right after.


```python
import base64
import numpy as np
import re
import shutil
import socket
import subprocess
from pathlib import Path
from tempfile import NamedTemporaryFile
from typing import List

URL = "9000:ff:1ce:ff:216:3eff:fe8c:4a0c"
PORT = 8000
PASSWORD = "I have come for the shadow training\r\n"
LOCAL_PATH = "/home/syvon/Desktop/nsec/fmw3.elf"

DEBUG = True
ACTUAL_CONNECT = not True

ADD_REGEX = re.compile(r"add    \$0x(.*),%eax")
CMP_REGEX = re.compile(r"cmp    \$0x(.*),")
MOD_REGEX = re.compile(r"mov    \$0x(.*),%ecx")


def _parse_int32(subs) -> int:
    if len(subs) != 8:
        missing = 8 - len(subs)
        subs = "0" * missing + subs

    parts = [
        int(subs[i:i + 2], 16) for i in range(0, 8, 2)
    ]

    return int.from_bytes(bytes(parts), byteorder='big', signed=False)


def parse_div_block(div_block: str) -> str:
    mod = _parse_int32(MOD_REGEX.findall(div_block)[0])
    neq = _parse_int32(CMP_REGEX.findall(div_block)[0])

    if mod > neq:
        add = neq
    else:
        add = np.int32(mod + neq)

    add_as_str = str(add)

    return add_as_str


def parse_neq_block(neq_block: str) -> str:
    add_value = _parse_int32(ADD_REGEX.findall(neq_block)[0])
    cmp_value = _parse_int32(CMP_REGEX.findall(neq_block)[0])

    num = np.uint32(cmp_value - add_value)

    value_as_str = str(int(num))
    return value_as_str


def logic_block_to_number(logic_block: str) -> str:
    if "cmp" not in logic_block:
        return ""

    # mod block
    if "div" in logic_block:
        r = parse_div_block(logic_block)
    else:
        r = parse_neq_block(logic_block)

    l = len(r)

    if l > 0:
        r = "0" * (10 - l) + r

    return r


def extract_logic_blocks(main_section: str) -> List[str]:
    zero_block = "00 00 00"

    blocks = []

    s = 0
    lines = main_section.splitlines()
    for i, line in enumerate(lines):
        if line.strip().endswith(zero_block):
            blocks.append(
                "\n".join(lines[s:i])
            )
            s = i

    return blocks


def extract_main(main_dis: str):
    start = "<main>"
    end = ".fini"

    start_i = main_dis.index(start)
    end_i = main_dis.index(end)

    sub = main_dis[start_i: end_i + 1]
    lines = sub.splitlines()[1:-1]

    main_dis = "\n".join(lines)
    if DEBUG:
        print("MAIN DIS")
        print(main_dis)

    return main_dis


def main():
    if ACTUAL_CONNECT:
        s = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
        s.settimeout(5)
        s.connect((URL, PORT))
    else:
        s = None

    # Read question prompt

    def sendmsg(msg: str, eol="\r\n") -> int:
        return s.send(f"{msg}{eol}".encode())

    def read_until_line() -> str:
        str_buff = ""

        try:
            while True:
                b = s.recv(1).decode()
                str_buff += b
                if b == "\n":
                    break
        except socket.timeout:
            ...

        return str_buff

    if ACTUAL_CONNECT:
        _prompt = read_until_line()

        s.send(PASSWORD.encode())

        # Read warning
        _warning = read_until_line()

        sendmsg("")  # ok

        # Read firmware as a string
        fmw_b64_s = read_until_line().strip()

        raw_fmw = base64.b64decode(fmw_b64_s, validate=True)

        with NamedTemporaryFile(prefix="fmw_", suffix=".elf") as ntf:
            ntf.write(raw_fmw)
            ntf.flush()
            path = Path(ntf.name)

            shutil.copy(path, LOCAL_PATH)

            proc_ret = subprocess.run(["objdump", str(path), "-d"], capture_output=True)
            dis = proc_ret.stdout.decode()
    else:
        proc_ret = subprocess.run(["objdump", LOCAL_PATH, "-d"], capture_output=True)
        dis = proc_ret.stdout.decode()

    # Compute the answer
    main_section = extract_main(dis)
    logic_blocks = extract_logic_blocks(main_section)
    numbers = list(filter(len, map(logic_block_to_number, logic_blocks)))

    for i, num in enumerate(numbers):
        print(f"{i + 1}. {num} ({len(num)})")

    answer = "".join(numbers)

    print(f"[AS]=> {answer} ({len(answer)})")

    answer_64 = base64.b64encode(answer.encode())

    print(f"[64]=> {answer_64}")

    # Encode in b64 and send
    if ACTUAL_CONNECT:
        sendmsg(answer_64.decode())

        # Read response from server
        print(read_until_line())
        print(read_until_line())


if __name__ == '__main__':
    main()
```

Just to prove to myself that it does work, here is how it works on 3 firmwares that I have
saved from the event: (I cut the output of my script which prints a bunch of irrelevant things)


```shell
$ ./main.py # hardcoded fmw1.elf in local path
...
 [AS]=> 33539974671917731587009210099410422577653903669765 (50)
$ echo "33539974671917731587009210099410422577653903669765" | ./fmw1.elf
  Deactivated.

$ ./main.py # hardcoded fmw2.elf in local path
...
 [AS]=> 16117043252740620650229748150501002654140000745959 (50)
$ echo "16117043252740620650229748150501002654140000745959" | ./fmw2.elf
  Deactivated.

$ ./main.py # hardcoded fmw3.elf in local path
...
 [AS]=> 10897182910046280801014144672632255690013573956266 (50)
$ echo "10897182910046280801014144672632255690013573956266" | ./fmw3.elf
  Deactivated.
```
