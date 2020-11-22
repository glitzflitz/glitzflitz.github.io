+++
title = "Memory registration in Qemu"
date = 2020-11-22
in_search_index = true
+++

So finally my ADHD brain decided to finish this post which I have been working
on since a month now. In this post we analyze memory registration process in Qemu.

## Memory registration interface
After allocating memory to guest, the **GPA**(Guest Physical Address) and
**HVA**(Host Virtual Address) mappping can be determined through Memory Region.
This mapping needs to be notified to KVM, so that it can use
`GPA->HVA->HPA(Host Physical Address)` mappings to be filled in
**EPT**(Extended Page Table) when processing guest page faults. Qemu Memory
Region represents the segment of the guest memory.

As KVM is a kernel module so it can directly go through host's page tables when
dealing with page faults and then fill obtained HPA into EPT. Why go through <br>
`GPA->HVA->HPA` cycle for prossing page faults? If the pages are filled direcly,
the page table of Qemu process won't be filled and Qemu won't know the memory
allocated for guest.

So after Qemu allocates the memory for guest at the begining, it needs to know
mapping between GPA and HVA in order to let it know the memory usage of guest.
This is where memory registration mechanism of Qemu comes in. KVM provides a
`KVM_SET_USER_MEMORY_REGION` ioctl for memory registration. It takes a
`kvm_userspace_memory_region` struct as input argument.

```
Capability: KVM_CAP_USER_MEM						/* 1 */
Architectures: all
Type: vm ioctl
Parameters: struct kvm_userspace_memory_region (in)
Returns: 0 on success, -1 on error

struct kvm_userspace_memory_region {
    __u32 slot;								/* 2 */
    __u32 flags;
    __u64 guest_phys_addr;						/* 3 */
    __u64 memory_size; /* in bytes */					/* 4 */
    __u64 userspace_addr; /* start of the userspace allocated memory */	/* 5 */
};
```
```
1. KVM_CAP_USER_MEM capability denotes whether kvm supports user memory
   registration or not
2. On which slot to register the memory range
3. Start of guest physical address range
4. Size of the physical address range
5. The backing HVA corresponding to memory region of the guest
```
Qemu executes this iotcl through `kvm_set_user_memory_region`
function. The GPA,HVA and `memory_size` are set after it allocates memory for
the guest but `slot` number is obtained by querying KVM because Qemu registered
memory must be placed in an empty slot.

## Memory Slots
Qemu will query the maximum number of guest memory slots supported by KVM
through KVM_CHECK_EXTENSION ioctl when initializing the guest.
```
kvm_init
	s->nr_slots = kvm_check_extension(s, KVM_CAP_NR_MEMSLOTS) /*Query the maximum number of slots supported by KVM */
```
Then it goes into following case of switch statement
```
kvm_vm_ioctl_check_extension		/ * kvm returns the maximum number of slots defined in the code */
	case KVM_CAP_NR_MEMSLOTS:
       	r = KVM_USER_MEM_SLOTS;
```
After obtaining the number of slots, Qemu registers listener to the memory address space
`address_space_memory` and allocates corresponding number of slots. All slots
are assigned an unique id. At this time these slots are free.
```
kvm_init
    kvm_memory_listener_register(s, &s->memory_listener, &address_space_memory, 0);
    	kml->slots = g_malloc0(s->nr_slots * sizeof(KVMSlot));	/* Allocate the memory for maximum number of slots */
      		for (i = 0; i < s->nr_slots; i++) {
        		kml->slots[i].slot = i;
    		}
    	kml->listener.region_add = kvm_region_add; /* Assign listener's region_add callback */
		......
    	memory_listener_register(&kml->listener, as);   /* Add this listener to global memory_listeners list */
```
After Qemu allocates memory for the guest, it will update address space topopogy,
call the `region_add` callback for each listener in global `memory_listeners` list
and finally call the `kvm_region_add` from KVM
```
kvm_region_add
	kvm_set_phys_mem
	 	/* register the new slot */
    	mem = kvm_alloc_slot(kml);							/* Find the free slot. `memory_size` of slot(struct KVMSlot) is 0 for the free slot  */
    	mem->memory_size = size;							/* The size of the guest memory taken from kvm_userspace_memory_region */
    	mem->start_addr = start_addr;						/* Starting physical address of guest taken from kvm_userspace_memory_region */
		......
		kvm_set_user_memory_region							/* Register the obatined slot with KVM */
```
Finally lets take a look at how slots are allocated
```
kvm_alloc_slot
	kvm_get_free_slot
	    for (i = 0; i < s->nr_slots; i++) {
        	if (kml->slots[i].memory_size == 0) {		/* The size of the slot is 0 which means it is free */
            	return &kml->slots[i];
        	}
    	}
```

## KVM side
What would KVM do after it receives ioctl from Qemu to register the memory?
For the purpose of memory registration, KVM needs to maintain mapping between
GPA and HVA. When guest is out of pages, it uses this mapping to determine
HVA from GPA. `kvm_memory_slot` data structure is used to maintin this GPA to
HVA mapping.

```
struct kvm_memory_slot {
    gfn_t base_gfn;						/* 1 */
    unsigned long npages;   			/* 2 */
    unsigned long *dirty_bitmap;		/* 3 */
    struct kvm_arch_memory_slot arch;	/* 4 */
    unsigned long userspace_addr;		/* 5 */
    u32 flags;
    short id;
};
```
```
1. The physical page frame number of memory area of the guest. After adding piece of memory to the slot, that slot is associated with a segment of guest's memory. `base_gfn` keeps track of starting page frame number of this area.
2. Convert the size of memory region into number of pages
3. This data structure also provides a dirty page recording mechanism for each page in the memory of the slot. When this mechanism is enabled, it will record dirty pages through `dirty_bitmap` which is used in memory migration.
4. Reverse mapping related field
5. HVA corresponding to given memory region
```
The `arch` member is used to record reverse mapping. When the page needs to be
reclaimed on the host, if that page is used by the guest, the status of page
recorded in EPT needs to be changed from existing/present to non-existent/absent.<br>

We need to locate all EPT page entries that point to the same physical page
frame number. So how KVM does it? Well, the kernel has a page frame recycling
algorithm.The reverse mapping can be used to locate all page table entries that
point to the same page frame, that is, through host physical page frame number
you can get all the HVA that use the page. The `kvm_memory_slot` can check the
corresponding GPA. At this time, there are two ways to find which EPT entry the
GPA falls on. <br>
The first is to simply traverse the EPT, looking down one by one, until you get
the page table entry that contains desired GPA. The other is to maintain a
reverse mapping and record the address of page table entry corresponding to the
GPA when EPT page table entry is created when KVM processes the page fault.
The `arch.rmap` member maintains this mapping
```
struct kvm_arch_memory_slot {
    struct kvm_rmap_head *rmap[KVM_NR_PAGE_SIZES];		/* 6 */
	......
};
```
```
6. KVM_NR_PAGE_SIZES is the type of pages of different sizes allocated by Qemu for guests that is 8KB, 2M, 1G for x86. KVM needs to maintain pages of different page sizes and its corresponding EPT entry. Each rmap[i] is an array and it contains address of EPT entry corresponding to the page. The entire array is allocated at the time of memory registration.
```
`kvm_memory_slot` describes a guest memory range. There are many such memory
segments in a guest. All slots are put together and managed by KVM. The slots
are independent of each other. `kvm->memslots` maintains array of address spaces,
each containing multiple slots:
```
struct kvm {
	......
    struct kvm_memslots *memslots[KVM_ADDRESS_SPACE_NUM];	/* 7 */
	......
```
```
7. The current number of address space KVM_ADDRESS_SPACE_NUM is defined to be 2, hence `memslots` has 2 members corresponding to 2 address spaces
```
```
struct kvm_memslots {
	......
    struct kvm_memory_slot memslots[KVM_MEM_SLOTS_NUM];		/* An array of all slots maintained by corresponding address space */
	......
};
```

The relation between memory slot, address space and kvm struct is something like this  ->
![memory-registration.jpg](https://raw.githubusercontent.com/glitzflitz/glitzflitz.github.io/main/static/images/memory_registration.jpg)
(please excuse my below average krita drawing skills)

## Memory registration process
We looked at how kvm does memory registration. The process starts from calling ioctl and finally calls `__kvm_set_memory_region`
function which does the main work of memory registration
```
kvm_vm_ioctl
	case KVM_SET_USER_MEMORY_REGION:
		kvm_vm_ioctl_set_memory_region(kvm, &kvm_userspace_mem);
			kvm_set_memory_region
				__kvm_set_memory_region
int __kvm_set_memory_region(struct kvm *kvm,
                const struct kvm_userspace_memory_region *mem)
{
	as_id = mem->slot >> 16;					/* 1 */
    id = (u16)mem->slot;
   	slot = id_to_memslot(__kvm_memslots(kvm, as_id), id);		/* 2 */
    base_gfn = mem->guest_phys_addr >> PAGE_SHIFT;			/* 3 */
    npages = mem->memory_size >> PAGE_SHIFT;				/* 4 */
    new.id = id;							/* 5 */
    new.base_gfn = base_gfn;
    new.npages = npages;
    if (change == KVM_MR_CREATE) {
        new.userspace_addr = mem->userspace_addr;
        if (kvm_arch_create_memslot(kvm, &new, npages))
            goto out_free;
    }
	......
```

```
1. The lower 16 bits of the slot denote slot number and upper part denote id of the address space.
2. Find the corresponding address space's kvm_memslots from kvm->memslots[] array according address space id and then find corresponding address from kvm_memslots->memslots[] array according to slot id
3,4,5. Get the newly registered kvm_memory_slot struct members and put them into new slot
```
The slot struct was created and not initiized when kvm was initiized so it doesn't know
how many pages it needs to put. The `arch` member of reverse mapping is also not initiized.
`kvm_arch_create_memslot` initilizes this member according to number of registered pages:
```
int kvm_arch_create_memslot(struct kvm *kvm, struct kvm_memory_slot *slot,
                unsigned long npages)
{
	......
    for (i = 0; i < KVM_NR_PAGE_SIZES; ++i) {				/* 1 */
    	lpages = gfn_to_index(slot->base_gfn + npages - 1,		/* 2 */
                      		slot->base_gfn, level) + 1;
      	slot->arch.rmap[i] = kvzalloc(lpages * sizeof(*slot->arch.rmap[i]), GFP_KERNEL); /* 3 */
   	}
```
```
1. For pages of different sizes, initialize the arch member of the slot. Pages with different sizes have seprate data structure
2. Calculate the number of pages based on size of the memory region
3. To allocate the memory of reverse mapping rmap[i], we can see that lpages number of elements are allocated, so each page registered by QEMU has a corresponding EPT entry address stored in the 2d rmap array.
 When kvm processes page faults and goes through the host page fault process, after filling in the EPT, the page table address will also be written to the ramp
 ```
 Note that the "slot" here has nothing to do with physical memory slot. KVM just acts as a memory allocator.
