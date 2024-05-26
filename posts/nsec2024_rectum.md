Code for copying a waste, unlabeled. Not much to label, it's pretty self
explanatory, but gives us insight in the structure.

```C

char * waste_copy(char *param_1) {
  char *__dest;
  
  __dest = (char *)tgc_alloc(gc,0x810);
  strcpy(__dest,param_1);
  *(undefined8 *)(__dest + 0x800) = *(undefined8 *)(param_1 + 0x800);
  *(undefined8 *)(__dest + 0x808) = *(undefined8 *)(param_1 + 0x808);
  return __dest;
}
```

Code for reading a waste:

```C
long waste_read(void) {
  long waste_struct;
  long waste_name;
  undefined8 content_size;
  
  waste_struct = tgc_alloc(gc,0x810);
  puts("[?] Waste Name:");
  fflush(stdout);
  waste_name = read_line(stdin,waste_struct,0x7ff);
  rstrip(waste_struct,waste_name,10);
  *(undefined *)(waste_name + waste_struct) = 0;
  puts("[?] Waste Size:");
  fflush(stdout);
  content_size = read_atoi(stdin);
  *(undefined8 *)(waste_struct + 0x800) = content_size;
  content_size = tgc_alloc(gc,*(undefined8 *)(waste_struct + 0x800)); // not really content_size, but the compiler used the same field
  *(undefined8 *)(waste_struct + 0x808) = content_size;
  puts("[?] Waste Content:");
  fflush(stdout);
  read_exact(stdin,*(undefined8 *)(waste_struct + 0x808),*(undefined8 *)(waste_struct + 0x800));
  return waste_struct;
}
```

As we can see; the allocation for the structure is of size `0x810`. Of those
`0x810` bytes, `0x7FF` are allocated for the name. If we take the difference,
that leaves 17 bytes. Looking at the rest of the code, we see a write at
`0x808` and one at `0x800`, of type `undefined8` meaning 8 bytes, so we can
take the leftover one as padding or a trailing 0. We can guess the structure
looks like:

```C
struct Waste {
    char name[0x7FF],
    undefined8 content_size;
    undefined8 content_ptr;
}
```

Quite a long name available!


Code for adding a waste, unlabeled

```C
void wastes_add(long param_1) {
  int iVar1;
  char *__s1;
  long lVar2;
  
  iVar1 = wastes_find_next_avail(param_1);
  if (iVar1 < 0) {
    puts("No space left for wastes\n");
    fflush(stdout);
  }
  __s1 = (char *)waste_read();
  *(char **)(param_1 + (long)iVar1 * 8) = __s1;
  tgc_set_dtor(gc,__s1,waste_dtor);
  iVar1 = strcmp(__s1,"orao");
  if (iVar1 == 0) {
    puts("[!] ORAO WASTE DETECTED [!]");
    iVar1 = wastes_find_next_avail(param_1);
    if (-1 < iVar1) {
      puts("[!] Spliting Orao in two for easier processing");
      lVar2 = waste_copy(__s1);
      *(ulong *)(lVar2 + 0x800) = *(ulong *)(__s1 + 0x800) >> 1;
      *(long *)(lVar2 + 0x808) = *(long *)(lVar2 + 0x808) + *(long *)(lVar2 + 0x800);
      *(long *)(param_1 + (long)iVar1 * 8) = lVar2;
    }
    fflush(stdout);
  }
  return;
}
```

Labelled,

```C
void wastes_add(long waste_chain) {
  int index;
  char *waste_ptr;
  long waste_cpy;
  
  index = wastes_find_next_avail(waste_chain);
  if (index < 0) {
    puts("No space left for wastes\n");
    fflush(stdout);
  }
  waste_ptr = (char *)waste_read();
  *(char **)(waste_chain + (long)index * 8) = waste_ptr;
  tgc_set_dtor(gc,waste_ptr,waste_dtor);
  index = strcmp(waste_ptr,"orao");
  if (index == 0) {
    puts("[!] ORAO WASTE DETECTED [!]");
    index = wastes_find_next_avail(waste_chain);
    if (-1 < index) {
      puts("[!] Spliting Orao in two for easier processing");
      waste_cpy = waste_copy(waste_ptr); // basically, a simple field by field copy. Gives us insight in the structure
      *(ulong *)(waste_cpy + 0x800) = *(ulong *)(waste_ptr + 0x800) >> 1;
      *(long *)(waste_cpy + 0x808) = *(long *)(waste_cpy + 0x808) + *(long *)(waste_cpy + 0x800);
      *(long *)(waste_chain + (long)index * 8) = waste_cpy; // SAM: update the next PTR in the chain
    }
    fflush(stdout);
  }
  return;
}
```

If we analyze the orao case, we see the length is chopped in half, and the
pointer to the content is set to halfway the length of the content. This is
interesting, because this pointer is **not known** to the GC. This means
eventually, the GC will reallocate **over** the area of the content.

Here is the plan:
- Allocates a big orao.
- This big orao gets split in two.
- We free the first orao: because it's big, it gives us room for reallocation
over the copy.
- The GC will not reallocate over the original orao yet because it's taken, so
we need to force the GC to run.
    - We can do this by allocating and freeing a bunch
- We allocate few wastes, hoping that the wastes ends up allocated in the area
copied orao's content.
- Find the waste that lines up in the orao copy. We'll call it our tool waste.

We now have: 1) `orao_copy` and 2) `tool_waste`. 

```
                   ^-----------------------------------------------
orao_copy: [name, sz, ptr]      [tool_waste, sz, ptr]
                        v-------^
```

We will use `orao_copy` to edit the pointer of `tool_waste`. This will
allow us to read and write EVERYWHERE in memory! We can do that because:

1. List waste gives us the content of a waste
2. Edit waste allows us to write stuff

Editing the pointer of `tool_waste`, using `orao_copy`, allows us to read using
reading `tool_waste`.

So:

1. If we edit `orao_copy`, we can change the pointer of `tool_waste`
2. We can read using list waste and look at `tool_waste`
3. We can write using edit waste on `tool_waste`

