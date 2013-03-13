---
layout: post
title: Linux内核模块引用未导出的符号
---

默认情况下，在Linux内核中，只有经过用EXPORT_SYMBOL()这种宏导出过的符号，才能被其它模块引用。但有时在调试已有系统的BUG时，想要调用一些未导出的函数，如果这些未导出的函数是自己实现的，那么可以把它们导出，再编译一遍就可以了。然而有没有办法不编译，或根本就没有代码的情况下，也可以调用未导出的符号呢？先请出我们的“老朋友”—— 谁？—— 当然Google啦~~:)

第一页的结果中最好的应该是了[它了][lref-1]，下面是对应功能代码：

    #include <linux/module.h>中文
    #include <linux/kallsyms.h>
    #include <linux/string.h>
    
    MODULE_LICENSE("GPL");
    MODULE_DESCRIPTION("Access non-exported symbols");
    MODULE_AUTHOR("Stephen Zhang");
    
    static int __init lkm_init(void)
    {
        char *sym_name = "resume_file";
        unsigned long sym_addr = kallsyms_lookup_name(sym_name);
        char filename[256];
    
        strncpy(filename, (char *)sym_addr, 255);
    
        printk(KERN_INFO "[%s] %s (0x%lx): %s\n", __this_module.name, sym_name, sym_addr, filename);
    
        return 0;
    }
    
    static void __exit lkm_exit(void)
    {
    }
    
    module_init(lkm_init);
    module_exit(lkm_exit);

但在实践过程中，我所使用的内核（2.6.30)没有导出_kallsyms_lookup_name_这个符号，再研究一番kallsyms.c后，发现_kallsyms_on_each_symbol_这个函数有被导出，而且函数可以完成的功能和_kallsyms_lookup_name_是一样的，只不过需要自己实现一个回调函数来检查是不是自己想要的符号：

    #include <linux/module.h>
    #include <linux/kallsyms.h>
    #include <linux/string.h>
    
    MODULE_LICENSE("GPL");
    MODULE_DESCRIPTION("get address of the sym");
    MODULE_AUTHOR("ruizhe");
    
    
    char *sym = "resume_file";
    module_param (sym, charp, S_IRUGO);
    
    unsigned long (*kallsyms_lookup_name_f)(char *name);
    
    static int find_fn (void *data, const char *name, struct module *mod, unsigned long addr)
    {
      if (name != NULL && strcmp (name, "kallsyms_lookup_name") == 0) {
        kallsyms_lookup_name_f = addr;
        return 1;
      }
    
      return 0;
    }
    
    static void get_fn_addr (void)
    {
      kallsyms_on_each_symbol (find_fn, 0);
    }
    
    static int __init getaddr_init(void)
    {
      unsigned long sym_addr;
      char filename[128];
    
      get_fn_addr();
      if (kallsyms_lookup_name_f == NULL)
        {
          printk (KERN_INFO "can't get kallget addr\n");
          return 0;
        }
    
      sym_addr = kallsyms_lookup_name_f(sym);
      if (0 == sym_addr)
        {
          printk (KERN_INFO "can't find sym\n");
          return 0;
        }
      strncpy(filename, (char *)sym_addr, 127);
      printk (KERN_INFO "[%s] %s (0x%lx): %s\n", __this_module.name, sym, sym_addr,
    	  filename);
    
      return 0;
    }
    
    static void __exit getaddr_exit(void)
    {
    
    }
    
    module_init(getaddr_init);
    module_exit(getaddr_exit);

上面代码先用_kallsyms_on_each_symbol_找到了_kallsyms_lookup_name_函数的地址，然后就可以用kallsyms_lookup_name来查找其它符号（包括全局变量哦）的地址了。


参考：
* [Introducing Linux Kernel Symbols][lref-1]

[lref-1]: http://onebitbug.me/2011/03/04/introducing-linux-kernel-symbols/

