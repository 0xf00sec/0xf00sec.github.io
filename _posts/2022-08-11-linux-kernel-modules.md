---
title: The Linux Kernel Modules Programming
layout: post
tags:
  - Programming
blog_post: true
---


In this tutorial, I’m going to teach you how to write linux kernel modules, it is necessary to know C programming language. You will probably ask "So, what the hell is that linux kernel module?"


it is a piece of code that can be dynamically loaded and unloaded from the kernel, "maybe you don’t know what kernel is" It is the main part of each operating system. It is "program" that is loaded and executed by bootloader at the boot time The kernel manages all system resources. It’s responsible for communication between software and hardware, manages all user’s processes and many, many more.

Kernel and user mode processes run at different privilege levels. Modern processors, such as Intel processors, support multiple privilege levels, including ``ring0 (the most powerful)`` and ``ring3 (the least powerful)``. Linux, however, utilizes only two of these levels: ring0 for the kernel and ring3 for user-mode processes. You might ask, "Why are these privilege levels useful?" The reason is security. If user-mode processes ran in ``ring0``, they would have the ability to execute potentially harmful commands, such as "cli," which would halt all interrupts and potentially crash the kernel. To prevent such issues, only the kernel runs in ``ring0``, while user-mode processes run in ring3, where they can't harm the kernel.

The key advantage of Linux kernel modules is that they run in ``ring0 (kernel mode)``, unlike regular processes that run in ``ring3 (user mode)``. Keep in mind that only the root user can load them. So, why are they useful? Consider a scenario where you have hardware for which there are no drivers compiled into the kernel. Kernel modules come to the rescue. A kernel module can serve as a driver for that hardware. By loading such a kernel module, you can use your hardware without the need to recompile the entire kernel to include support for it, which can take a long time... 

### Loading Modules

Modules typically have the ".ko" extension. To load a module, you simply execute the following command as the root user:

```shell
insmod module.ko
```

Once the module is loaded, you can unload it with the following command:

```shell
rmmod module
```
or

```shell
rmmod module.ko
```

Now that we understand what Linux kernel modules are, why they are useful, and how to load and unload them, let's write a simple module - the classic "Hello, World!" module.

### Hello World Module

```cpp
/* These are standard module include files */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/init.h>

/* This is function that will be executed when we load the module */
int __init mod_init(void)
{
	/* Now we see printk function. This is something like
	standard printf in C. But we are working in kernel mode
	so we can't use functions used by user mode programs.
	I will give more information about this function right
	after the code of this module */
	printk(KERN_ALERT "Hello world!\n");

	return 0;
}

/* This function will be executed when we unload the module */
void __exit mod_exit(void)
{
	printk(KERN_ALERT "Bye world\n");
}


/* Here we register mod_init and mod_exit */

/* mod_init and mod_exit functions can have different names they only have to be registered by module_init and module_exit macros */

/* As a parameter of module_init we give function that has to be executed during loading of the module */
module_init(mod_init);

/* As a parameter of module_exit we give function that has to be executed during unloading of the module */
module_exit(mod_exit);
```


I will explain `printk` function, First thing if you want to "normally" see what `printk` writes you must load and unload module not in “X” but from standard tty console, So `printk` just writes given text to the screen if you load module in “X” mode you won’t see what `printk` wrote. However you can still see it  
by executing command `dmesg` However, `printk` was not meant to communicate with user it is rather used as a logging mechanism. `KERN_ALERT` is a priority of message to be logged, There are 8 priorities levels each level has its own macro if value of used priority is lower(the lower value it has, the more important is the message)  than `console_loglevel` the message is written to the screen you can see all macros in `linux/kernel.h` file(this is relative path from root kernel’s source code directory)

### Compiling a Module

To compile such a module, you need a special Makefile:

```bash
obj-m += hello.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=${PWD} modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=${PWD} clean


````

Assuming your source code file is named "hello.c," execute `make` to generate the "hello.ko" file: 

```shell
insmod hello.ko
```

You should see "Hello, world!" To unload the module, use:

```sh
rmmod hello
```

You should see "Goodbye, world!"

### PROCFS

Linux features a helpful mechanism that enables communication between the kernel and modules with processes: **procfs**. In most, you can find it in the "/proc" directory, with subdirectories for all processes and various files, such as "/proc/version," which provides information about the kernel's version. In this section, I will demonstrate how to create "files" or "entries" in **procfs** and explain the necessary functions and structures for our module.

- `struct proc_dir_entry`: Each entry in **procfs** is represented by its own `proc_dir_entry` structure. Let's examine its definition:

```cpp
struct proc_dir_entry {
        unsigned int low_ino;
        unsigned short namelen;
        const char *name;
        mode_t mode;
        nlink_t nlink;
        uid_t uid;
        gid_t gid;
        loff_t size;
        const struct inode_operations *proc_iops;
        /*
         * NULL proc_fops means PDE is going away RSN or
         * PDE is just created. In either case, e.g. read_proc won't be
         * called because it's too late or too early, respectively.
         *
         * If you're allocating proc_fops dynamically, save a pointer
         * somewhere.
         */
        const struct file_operations *proc_fops;
        struct proc_dir_entry *next, *parent, *subdir;
        void *data;
        read_proc_t *read_proc;
        write_proc_t *write_proc;
        atomic_t count;         /* use count */
        int pde_users;  /* number of callers into module in progress */
        spinlock_t pde_unload_lock; /* proc_fops checks and pde_users bumps */
        struct completion *pde_unload_completion;
        struct list_head pde_openers;   /* who did open, but not release */
};
```


1. `name`: The name of the entry in procfs.
2. `mode`: Permissions for accessing the entry (e.g., 777).
3. `read_proc`: A pointer to the function responsible for reading from this file.
4. `count`: The number of bytes that can be written to this file.

Let's take a look at the prototype for `read_proc_t`:

```cpp
typedef int (read_proc_t)(char *page, char **start, off_t off,
                          int count, int *eof, void *data);
```

Here we are interested only in page and count arguments page is pointer to user mode buffer where we have to write data

The `write_proc` function is a pointer to the function responsible for writing to this file. Its prototype is:

```cpp
typedef int (write_proc_t)(struct file *file, const char __user *buffer,
                           unsigned long count, void *data);
````

We are interested in the `buffer` and `count` parameters. `buffer` is a pointer to the user-mode buffer containing the data to be written, and `count` is the size of this buffer.

Additional functions required for our module:

1. `sprintf` and `snprintf` (used similarly to standard `sprintf` and `snprintf` in user-mode programs).
2. `copy_from_user(void *dst, void *src, int count)`: This function copies data from a user-mode buffer (`src`) to our kernel-mode buffer (`dst`) for the specified number of bytes (`count`).
3. `remove_proc_entry(char *name, struct proc_dir_entry *parent)`: This function removes an entry from procfs, where `name` is the entry's name, and `parent` specifies the sub-directory of `/proc` where the entry is located (use 0 for the root directory).

Now, let's take a look at the code for our module:

```cpp
/* Standard includes for modules */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/init.h>

/* for proc_dir_entry and create_proc_entry */
#include <linux/proc_fs.h>

/* For sprintf and snprintf */
#include <linux/string.h>

/* For copy_from_user */
#include <linux/uaccess.h>

MODULE_LICENSE("GPL");

static char our_buf[256];

int buf_read(char *buf, char **start, off_t offset, int count, int *eof, void *data)
{
	int len;
	/* For example - when content of our_buf is hello when user executes command cat /proc/test_proc;
	he will see content of our_buf(in our example hello */
	len = snprintf(buf, count, "%s", our_buf);
	return len;
}

/* When user writes to our entry. For example echo aa > /proc/test_ptoc. aa will be stored in our_buf.
Then, when user reads from our entry(cat /proc/test_proc) he will see aa */
static int buf_write(struct file *file, const char *buf, unsigned long count, void *data)
{
	/* If count is bigger than 255, data which user wants to write is too big to fit in our_buf. We don't want
	any buffer overflows, so we read only 255 bytes */	
	if(count > 255)
		count = 255;
	/* Here we read from buf to our_buf */
	copy_from_user(our_buf, buf, count);
	/* we write NULL to end the string */
	our_buf[count] = '\0';
	return count;
}

int __init start_module(void)
{

	/* We create our entry */	
	struct proc_dir_entry *de = create_proc_entry("test_proc", 0666, 0);

	/* Set pointers to our functions reading and writing */
	de->read_proc = buf_read;
	de->write_proc = buf_write;

	/* We initialize our_buf with some text. */
	sprintf(our_buf, "hello");

	return 0 ;
}

void __exit exit_module(void)
{
	/* We delete our entry */
	remove_proc_entry("test_proc", NULL);
}

module_init(start_module);
module_exit(exit_module);
```

To compile and test the module, modify the Makefile as follows:


```sh
obj-m += proc.o all:     make -C /lib/modules/$(shell uname -r)/build M=${PWD} modules clean:     make -C /lib/modules/$(shell uname -r)/build M=${PWD} clean
```

Assuming your source code file is named "proc.c," execute `make` to generate the "proc.ko" file. To load the module, use the following command as the root user:

```shell
insmod proc.ko
```

You should now be able to test the module.

### NOTIFIERS

Next, let's discuss something called a "notify chain." Some events in the kernel are critical, and various kernel subsystems want to be informed when these events occur. Notify chains are used to facilitate this communication. We will use keyboard events as an example. When a user presses a key, the kernel reads it and uses a notify chain to inform all subsystems interested in keypress events. One of the key structures used in notifiers is the "notifier_block." Here's its definition:

```c
struct notifier_block {     int (*notifier_call)(struct notifier_block *self, unsigned long x, void *data);     struct notifier_block *next;     int priority; };
```

This structure is used to register in notify chains. The `notifier_call` function is called when an event occurs. The `priority` field indicates the priority of the function; functions with higher priorities are called first. However, this field is typically set to 0. The `next` field is a pointer to the next registered `notifier_block`.

So, how can a module register in the keyboard notify chain? First, create a `notifier_block` structure and initialize it:

```c
struct notifier_block nb; nb.notifier_call = function;
```

Here, `function` is the function to be executed when an event occurs. Register it with `register_keyboard_notifier(&nb);`.

You'll need to write a function that handles keypress events. What are the parameters? As in the prototype:

```c
int (*notifier_call)(struct notifier_block *self, unsigned long stage, void *data);
```

- `self`: A pointer to our `notifier_block`. We don't use it.
- `stage`: The stage of handling the keypress. We only take action when the stage is `KBD_KEYSYM`.
- `data`: A pointer to a `keyboard_notifier_param` structure. We are interested in the `value` field, which stores the value of the pressed key.

Our module will be a simple random number generator, primarily for educational purposes. Please note that this module should not be used in a production environment.

```c
```cpp
/* Standard includes for modules */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
/* For keyboard_notifier param etc. */
#include <linux/keyboard.h>
/* notifier_block etc. */
#include <linux/notifier.h>
/* create_proc_entry etc. */
#include <linux/proc_fs.h>
/* sprintf */
#include <linux/string.h>

MODULE_LICENSE("GPL") ;

static unsigned long long random_num ;

/* Our function that will be executed when a key is pressed */
static int kbd_notify(struct notifier_block *self, unsigned long stage, void *data)
{
	struct keyboard_notifier_param *param = data; /* Pointer to keyboard_notifier_param */
	int value = param->value - 0xf000; /* Value can't be to big */
	/* We calculate our random number by random_num*key_value
	This is really dummy and improfessional, but this module
	only has to show how to use notifiers */	
	/* Value must fit in long long :) */
	if(random_num > 1000000000)
		random_num -= 10*value;	
	else	
		random_num  *= value;
	return NOTIFY_DONE ;
}

/* We prepare our notifier_block structure. kbd_notify is function handling events */
static struct notifier_block kbd_nb = { .notifier_call = kbd_notify, } ;

/* We register in notify chain */
static void handler_init(void)
{
	register_keyboard_notifier(&kbd_nb) ;
}

/* We write this random number to user mode buffer */
static int random_read(char *buf, char **start, off_t off, int count, int *peof, void *data)
{
	int len = sprintf(buf, "%llu", random_num);

	return len ;
}

/* We create entry in procfs */
static void proc_init(void)
{
	struct proc_dir_entry *de = create_proc_entry("random_simple", 0444, 0);
	de->read_proc = random_read;
}

static int __init random_init(void) 
{
	handler_init();
	proc_init();
	random_num = 1;
	

	return 0 ;
}

static void __exit random_exit(void)
{
	remove_proc_entry("random_simple", 0);
	unregister_keyboard_notifier(&kbd_nb);
}


module_init(random_init) ;
module_exit(random_exit) ;
/*  Now you can compile it and test. */
```

That concludes our tutorial. I hope you've learned something valuable about Linux kernel modules, and I encourage you to explore the Linux kernel's source code. If you're interested in diving deeper into the kernel, consider starting with writing kernel modules, I began my adventure with linux kernel by writing linux kernel modules one day I got an idea to write a [simple rootkit](https://0x00sec.org/t/writing-a-simple-rootkit-for-linux/29034), I wanted my rootkit to be difficult to detect to achieve this i had to know more organisation of kernel’s structures, Firstly, I analyzed how `procfs` works and how new entries are created etc. 
