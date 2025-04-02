# Android启动之LK

> LK是Little Kernel的简称，它是一款bootloader程序，类似的bootloader还有以前的U-Boot，vivi。现Qualcomm、MTK都使用LK来作为系统引导程序。

## 1. 概述

Bootloader是移动平台/嵌入式系统在开机上电后执行的一段或几段程序。开机上电后，在芯片内部的SRAM控制器作用下默认跳转到某一固定地址执行，该位置所在即BootROM，BootROM在完成CPU Core初始化后将加载bootloader程序执行。一般bootloader都分成几个文件，分阶段引导。BootROM加载bootloader第一阶段BL1到On-chip SRAM执行，BL1在完成SOC初步的初始化后将引导BL2到SDRAM执行。以MTK平台为例，启动加载顺序依次为BootROM->preloader->LK。

LK的代码一般在源码bootable/bootloader/lk目录下，目录结构如下：

```
├── app 　　　　LK上的应用，aboot，shell等
├── arch　　　　体系/架构相关，arm，x86
├── dev 　　　　设备相关，key，usb，pmic等
├── include　　　头文件
├── kernel　　　LK核心，thread，timer等
├── lib　　　　　C库
├── LICENSE
├── make　　　　编译mk文件
├── makefile
├── platform　　驱动相关
├── project　　　makefile文件
├── scripts　　　Jtag脚本
└── target　　　具体板子相关
```

LK的工作是引导系统启动，它主要完成以下工作：

* 硬件初始化，设置中断向量表(vector table)，初始化MMU，cache，peripherals，storage，USB，crypto等
* 加载boot.img(inux Kernel与ramdisk)，引导系统启动
* 支持fastboot/recovery模式

## 2. 初始化流程

> 在64-bit体系结构下，LK也时运行在32-bit模式，通过TrustZone安全模式跳转到64-bit kernel执行。

链接脚本arch/arm/system-onesegment.ld中定义了入口地址`ENTRY(_start)` LK将&#x4ECE;_&#x73;tart开始执行，_&#x73;tart在/arch/arm/crt0.S中定义。 1.crt0.S CPU相关初始化 首先初始化异常向量表

```
.globl _start
_start:
  b reset
  b arm_undefined
  b arm_syscall
  b arm_prefetch_abort
  b arm_data_abort
  b arm_reserved
  b arm_irq
  b arm_fiq
```

接下来执行Set Up CPU操作，调用\_\_cpu\_early\_init()进行CPU相关初始化。 初始化各模式stack

```
.Lstack_setup:
  /* set up the stack for irq, fiq, abort, undefined, system/user, and lastly supervisor mode */
  mrs     r0, cpsr
  bic     r0, r0, #0x1f
​
  ldr   r2, =abort_stack_top
  orr     r1, r0, #0x12 // irq
  msr     cpsr_c, r1
  ldr   r13, =irq_save_spot   /* save a pointer to a temporary dumping spot used during irq delivery */
​
  orr     r1, r0, #0x11 // fiq
  msr     cpsr_c, r1
  mov   sp, r2
​
  orr     r1, r0, #0x17 // abort
  msr     cpsr_c, r1
  mov   sp, r2
​
  orr     r1, r0, #0x1b // undefined
  msr     cpsr_c, r1
  mov   sp, r2
​
  orr     r1, r0, #0x1f // system
  msr     cpsr_c, r1
  mov   sp, r2
​
  orr   r1, r0, #0x13 // supervisor
  msr   cpsr_c, r1
  mov   sp, r2
```

最后跳入C函数kmain()中执行 `bl kmain`

2.kmain()初始化流程 kmain()在kernel/main.c的中定义，执行流程如下：​!\[LK Call Flow]\(https://github.com/XRobinHe/Resource/blob/master/blog/image/android/lk\_call\_flow.png?raw=true)​

* thread\_init\_early();初始化线程相关结构
* arch\_early\_init();disable cache，设置中断向量表基址，初始化MMU，enable cache
* platform\_early\_init();interrupt contr，timer block初始化
* target\_early\_init();Uart等初始化

完成上述初始化操作后，新建线程并在新线程中执行bootstrap2 `thread_resume(thread_create("bootstrap2", &bootstrap2, NULL, DEFAULT_PRIORITY, DEFAULT_STACK_SIZE));` bootstrap2()分别调用arch\_init();platform\_init();target\_init();进一步对硬件平台初始化。最后将调用apps\_init()启动LK上的app。

3.apps\_init() apps\_init的实现如下，首先遍&#x5386;**\_\_apps\_start**&#x5230;**\_\_apps\_end**位置的app\_descriptor，并调用其init函数，然后在新线程中start app。

```
extern const struct app_descriptor __apps_start;
extern const struct app_descriptor __apps_end;
​
static void start_app(const struct app_descriptor *app);
​
/* one time setup */
void apps_init(void)
{
  const struct app_descriptor *app;
​
  /* call all the init routines */
  for (app = &__apps_start; app != &__apps_end; app++) {
    if (app->init)
      app->init(app);
  }
​
  /* start any that want to start on boot */
  for (app = &__apps_start; app != &__apps_end; app++) {
    if (app->entry && (app->flags & APP_FLAG_DONT_START_ON_BOOT) == 0) {
      start_app(app);
    }
  }
}
​
static int app_thread_entry(void *arg)
{
  const struct app_descriptor *app = (const struct app_descriptor *)arg;
​
  app->entry(app, NULL);
​
  return 0;
}
​
static void start_app(const struct app_descriptor *app)
{
  thread_t *thr;
  printf("starting app %s\n", app->name);
​
  thr = thread_create(app->name, &app_thread_entry, (void *)app, DEFAULT_PRIORITY, DEFAULT_STACK_SIZE);
  if(!thr)
  {
    return;
  }
  thread_resume(thr);
}
```

其&#x4E2D;**\_\_apps\_start**&#x4E0E;**\_\_apps\_end**在链接脚本arch/arm/system-onesegment.ld中定义，**\_\_apps\_start**&#x4E0E;**\_\_apps\_end**与中间存放的是apps section。

```
.rodata : {
  *(.rodata .rodata.* .gnu.linkonce.r.*)
  . = ALIGN(4);
  __commands_start = .;
  KEEP (*(.commands))
  __commands_end = .;
  . = ALIGN(4);
  __apps_start = .;
  KEEP (*(.apps))
  __apps_end = .;
  . = ALIGN(4);
  __rodata_end = . ;    
}
```

所有的LK app都需定义app\_descriptor结构体存放到apps段，app\_descriptor在include/app.h中定义

```
/* each app needs to define one of these to define its startup conditions */
struct app_descriptor {
  const char *name;
  app_init  init;
  app_entry entry;
  unsigned int flags;
};
​
#define APP_START(appname) struct app_descriptor _app_##appname __SECTION(".apps") = { .name = #appname,
#define APP_END };
```

通过宏**APP\_START**与**APP\_END**定义app\_descriptor结构体并放到apps sections中，例如在aboot.c的定义如下：

```
APP_START(aboot)
  .init = aboot_init,
APP_END
```

即定义了aboot的app\_descriptor结构体并存放在apps section中，在apps\_init中将调用到aboot\_init() 至此，初始化流程结束。

## 3. 系统引导流程

aboot的主要目的是进行系统引导，根据reboot\_mode来决定进入Main System或Recovery或fastboot模式等。​!\[lk mode]\(https://github.com/XRobinHe/Resource/blob/master/blog/image/android/lk\_modes.jpg?raw=true)​

下面从aboot\_init()来分析整个流程： 1.首先设置EMMC/NAND读取page大小

```
if (target_is_emmc_boot())
{
  page_size = mmc_page_size();
  page_mask = page_size - 1;
}
else
{
  page_size = flash_page_size();
  page_mask = page_size - 1;
}
```

2.选择开机模式 通过keys\_get\_state()与check\_reboot\_mode()确定开机模式。 keys\_get\_state()会检测按键，并根据按键的定义确定进入对应的模式。 check\_reboot\_mode()会读取指定memory位置信息，该位置信息由kernel shutdown时写入，例如执行`adb reboot recovery`，`adb reboot bootloader`，check\_reboot\_mode()获取的分别是`RECOVERY_MODE`，`FASTBOOT_MODE`。

```
reboot_mode = check_reboot_mode();
if (reboot_mode == RECOVERY_MODE)
{
  boot_into_recovery = 1;
}
else if(reboot_mode == FASTBOOT_MODE)
{
  boot_into_fastboot = true;
}
else if(reboot_mode == ALARM_BOOT)
{
  boot_reason_alarm = true;
}
```

如果reboot\_mode不是`FASTBOOT_MODE`，将调用boot\_linux\_from\_mmc()进入系统引导流程 如果reboot\_mode等于`FASTBOOT_MODE`，将执行fastboot\_init()进入fastboot模式 3.boot\_linux\_from\_mmc() boot\_linux\_from\_mmc()解析boot.img/recovery.img的头部boot\_img\_hdr结构来获取启动加载信息(如果是recovery模式,将从recovery分区加载recovery.img),其中cmdline对应传递给内核的参数,tags\_addr对应device tree table,boot\_img\_hdr结构体在app/aboot/bootimg.h中定义,如下:

```
#define BOOT_MAGIC "ANDROID!"
#define BOOT_MAGIC_SIZE 8
#define BOOT_NAME_SIZE  16
#define BOOT_ARGS_SIZE  512
#define BOOT_IMG_MAX_PAGE_SIZE 4096
​
struct boot_img_hdr
{
    unsigned char magic[BOOT_MAGIC_SIZE];
​
    unsigned kernel_size;  /* size in bytes */
    unsigned kernel_addr;  /* physical load addr */
​
    unsigned ramdisk_size; /* size in bytes */
    unsigned ramdisk_addr; /* physical load addr */
​
    unsigned second_size;  /* size in bytes */
    unsigned second_addr;  /* physical load addr */
​
    unsigned tags_addr;    /* physical addr for kernel tags */
    unsigned page_size;    /* flash page size we assume */
    unsigned dt_size;      /* device_tree in bytes */
    unsigned unused;    /* future expansion: should be 0 */
​
    unsigned char name[BOOT_NAME_SIZE]; /* asciiz product name */
​
    unsigned char cmdline[BOOT_ARGS_SIZE];
​
    unsigned id[8]; /* timestamp / checksum / sha1 / etc */
};
```

下图是本机编译生成的boot.img示例：​!\[boot\_img\_hdr]\(https://github.com/XRobinHe/Resource/blob/master/blog/image/android/bootimg.png?raw=true)​

根据boot\_img\_hdr结构的定义可以从上图获取到以下信息： **kernel\_size:** 0x009fc707 **kernel\_addr:** 0x80008000 **ramdisk\_size:** 0x001ddfb9 **ramdisk\_addr:** 0x81000000 **second\_size:** 0x00000000 **second\_addr:** 0x80f00000 **tags\_addr:** 0x80000100 **page\_size:** 0x00000100 **cmdline:** console=ttyHSL0,115200,n8 androidboot.console=ttyHSL0 androidboot.hardware=qcom user\_debug=31 msm\_rtb.filter=0x237 ehci-hcd.park=3 lpm\_levels.sleep\_disabled=1 cma=32M@0-0xffffffff androidboot.selinux=permissive

在解析完boot\_img\_hdr后，boot\_img\_hdr的信息被传递给boot\_linux执行具体的加载启动工作。

```
boot_linux((void *)hdr->kernel_addr, (void *)hdr->tags_addr,
     (const char *)hdr->cmdline, board_machtype(),
     (void *)hdr->ramdisk_addr, hdr->ramdisk_size);
​
​
 typedef void entry_func_ptr(unsigned, unsigned, unsigned*);
 void boot_linux(void *kernel, unsigned *tags,
    const char *cmdline, unsigned machtype,
    void *ramdisk, unsigned ramdisk_size)
 {
  unsigned char *final_cmdline;
 #if DEVICE_TREE
  int ret = 0;
 #endif
​
  void (*entry)(unsigned, unsigned, unsigned*) = (entry_func_ptr*)(PA((addr_t)kernel));
  uint32_t tags_phys = PA((addr_t)tags);
  struct kernel64_hdr *kptr = (struct kernel64_hdr*)kernel;
​
  ramdisk = (void *)PA((addr_t)ramdisk);
​
  final_cmdline = update_cmdline((const char*)cmdline);
​
 #if DEVICE_TREE
  dprintf(INFO, "Updating device tree: start\n");
​
  /* Update the Device Tree */
  ret = update_device_tree((void *)tags,(const char *)final_cmdline, ramdisk, ramdisk_size);
  if(ret)
  {
    dprintf(CRITICAL, "ERROR: Updating Device Tree Failed \n");
    ASSERT(0);
  }
  dprintf(INFO, "Updating device tree: done\n");
 #else
  /* Generating the Atags */
  generate_atags(tags, final_cmdline, ramdisk, ramdisk_size);
 #endif
​
  free(final_cmdline);
​
 #if VERIFIED_BOOT
  /* Write protect the device info */
  if (!boot_into_recovery && target_build_variant_user() && devinfo_present && mmc_write_protect("devinfo", 1))
  {
    dprintf(INFO, "Failed to write protect dev info\n");
    ASSERT(0);
  }
 #endif
​
  /* Turn off splash screen if enabled */
 #if DISPLAY_SPLASH_SCREEN
  target_display_shutdown();
 #endif
​
  /* Perform target specific cleanup */
  target_uninit();
​
  dprintf(INFO, "booting linux @ %p, ramdisk @ %p (%d), tags/device tree @ %p\n",
    entry, ramdisk, ramdisk_size, (void *)tags_phys);
​
  enter_critical_section();
​
  /* Initialise wdog to catch early kernel crashes */
 #if WDOG_SUPPORT
  msm_wdog_init();
 #endif
  /* do any platform specific cleanup before kernel entry */
  platform_uninit();
​
  arch_disable_cache(UCACHE);
​
 #if ARM_WITH_MMU
  arch_disable_mmu();
 #endif
  bs_set_timestamp(BS_KERNEL_ENTRY);
​
  if (IS_ARM64(kptr))
    /* Jump to a 64bit kernel */
    scm_elexec_call((paddr_t)kernel, tags_phys);
  else
    /* Jump to a 32bit kernel */
    entry(0, machtype, (unsigned*)tags_phys);
 }
```

至此，LK引导过程完成，将进入系统启动过程。

## 4. LK fastboot模式

当LK通过检测按键或reboot\_mode是`FASTBOOT_MODE`时，将进入到fastboot模式。 1.注册fastboot Command 通过aboot\_fastboot\_register\_commands()注册fastboot支持的命令

```
void aboot_fastboot_register_commands(void)
{
  int i;
  char hw_platform_buf[MAX_RSP_SIZE];
​
  struct fastboot_cmd_desc cmd_list[] = {
            /* By default the enabled list is empty. */
            {"", NULL},
            /* move commands enclosed within the below ifndef to here
             * if they need to be enabled in user build.
             */
#ifndef DISABLE_FASTBOOT_CMDS
            /* Register the following commands only for non-user builds */
            {"flash:", cmd_flash},
            {"erase:", cmd_erase},
            {"boot", cmd_boot},
            {"continue", cmd_continue},
            {"reboot", cmd_reboot},
            {"reboot-bootloader", cmd_reboot_bootloader},
            {"oem unlock", cmd_oem_unlock},
            {"oem unlock-go", cmd_oem_unlock_go},
            {"oem lock", cmd_oem_lock},
            {"flashing unlock", cmd_oem_unlock},
            {"flashing lock", cmd_oem_lock},
            {"flashing lock_critical", cmd_flashing_lock_critical},
            {"flashing unlock_critical", cmd_flashing_unlock_critical},
            {"flashing get_unlock_ability", cmd_flashing_get_unlock_ability},
            {"oem device-info", cmd_oem_devinfo},
            {"preflash", cmd_preflash},
            {"oem enable-charger-screen", cmd_oem_enable_charger_screen},
            {"oem disable-charger-screen", cmd_oem_disable_charger_screen},
            {"oem off-mode-charge", cmd_oem_off_mode_charger},
            {"oem select-display-panel", cmd_oem_select_display_panel},
#if UNITTEST_FW_SUPPORT
            {"oem run-tests", cmd_oem_runtests},
#endif
#endif
            };
​
  int fastboot_cmds_count = sizeof(cmd_list)/sizeof(cmd_list[0]);
  for (i = 1; i < fastboot_cmds_count; i++)
    fastboot_register(cmd_list[i].name,cmd_list[i].cb);
​
  /* publish variables and their values */
  fastboot_publish("product",  TARGET(BOARD));
  fastboot_publish("kernel",   "lk");
  fastboot_publish("serialno", sn_buf);
​
  /*
   * partition info is supported only for emmc partitions
   * Calling this for NAND prints some error messages which
   * is harmless but misleading. Avoid calling this for NAND
   * devices.
   */
  if (target_is_emmc_boot())
    publish_getvar_partition_info(part_info, ARRAY_SIZE(part_info));
​
  /* Max download size supported */
  snprintf(max_download_size, MAX_RSP_SIZE, "\t0x%x",
      target_get_max_flash_size());
  fastboot_publish("max-download-size", (const char *) max_download_size);
  /* Is the charger screen check enabled */
  snprintf(charger_screen_enabled, MAX_RSP_SIZE, "%d",
      device.charger_screen_enabled);
  fastboot_publish("charger-screen-enabled",
      (const char *) charger_screen_enabled);
  fastboot_publish("off-mode-charge", (const char *) charger_screen_enabled);
  snprintf(panel_display_mode, MAX_RSP_SIZE, "%s",
      device.display_panel);
  fastboot_publish("display-panel",
      (const char *) panel_display_mode);
  fastboot_publish("version-bootloader", (const char *) device.bootloader_version);
  fastboot_publish("version-baseband", (const char *) device.radio_version);
  fastboot_publish("secure", is_secure_boot_enable()? "yes":"no");
  smem_get_hw_platform_name((unsigned char *) hw_platform_buf, sizeof(hw_platform_buf));
  snprintf(get_variant, MAX_RSP_SIZE, "%s %s", hw_platform_buf,
    target_is_emmc_boot()? "eMMC":"UFS");
  fastboot_publish("variant", (const char *) get_variant);
#if CHECK_BAT_VOLTAGE
  update_battery_status();
  fastboot_publish("battery-voltage", (const char *) battery_voltage);
  fastboot_publish("battery-soc-ok", (const char *) battery_soc_ok);
#endif
}
```

其中主要是通过fastboot\_register注册Command到cmdlist，通过fastboot\_publish注册variables到varlist。 它们的实现如下：

```
static struct fastboot_cmd *cmdlist;
​
void fastboot_register(const char *prefix,
           void (*handle)(const char *arg, void *data, unsigned sz))
{
  struct fastboot_cmd *cmd;
  cmd = malloc(sizeof(*cmd));
  if (cmd) {
    cmd->prefix = prefix;
    cmd->prefix_len = strlen(prefix);
    cmd->handle = handle;
    cmd->next = cmdlist;
    cmdlist = cmd;
  }
}
​
static struct fastboot_var *varlist;
​
void fastboot_publish(const char *name, const char *value)
{
  struct fastboot_var *var;
  var = malloc(sizeof(*var));
  if (var) {
    var->name = name;
    var->value = value;
    var->next = varlist;
    varlist = var;
  }
}
```

执行adb reboot bootloader进入fastboot模式后，可以在PC端的fastboot可执行文件(Fastboot客户端源码在system/core/fastboot下，通过系统源码编译后的科执行文件位于./out/host/linux-x86/bin/fastboot目录下)与目标按fastboot协议交互。 通过fastboot\_register注册到cmdlist的命令，都可以在pc端通过fastboot执行，常用的命令有： **`fastboot devices` 查看连接设备 `fastboot update <filename>` 更新update.zip包 `fastboot flashall` flash boot, system, vendor, and recovery `fastboot flash <partition> [ <filename> ]` flash指定分区 `fastboot erase <partition>` erase执行分区 `fastboot continue` 继续启动 `fastboot reboot bootloader` reboot到bootloader `fastboot oem unlock` oem解锁**

**通过fastboot\_publish publish到varlist的value可以通过`fastboot getvar <variable>`获取，例如 `fastboot getvar version`** **`fastboot getvar product`**

**2.启动fastboot 通过`fastboot_init(target_get_scratch_address(), target_get_max_flash_size());`初始化并启动fastboot。**

```
int fastboot_init(void *base, unsigned size)
{
  char sn_buf[13];
  thread_t *thr;
  dprintf(INFO, "fastboot_init()\n");
​
  download_base = base;
  download_max = size;
​
  /* target specific initialization before going into fastboot. */
  target_fastboot_init();
​
  /* setup serialno */
  target_serialno((unsigned char *) sn_buf);
  dprintf(SPEW,"serial number: %s\n",sn_buf);
  surf_udc_device.serialno = sn_buf;
​
  if(!strcmp(target_usb_controller(), "dwc"))
  {
#ifdef USB30_SUPPORT
    surf_udc_device.t_usb_if = target_usb30_init();
​
    /* initialize udc functions to use dwc controller */
    usb_if.udc_init            = usb30_udc_init;
    usb_if.udc_register_gadget = usb30_udc_register_gadget;
    usb_if.udc_start           = usb30_udc_start;
    usb_if.udc_stop            = usb30_udc_stop;
​
    usb_if.udc_endpoint_alloc  = usb30_udc_endpoint_alloc;
    usb_if.udc_request_alloc   = usb30_udc_request_alloc;
    usb_if.udc_request_free    = usb30_udc_request_free;
​
    usb_if.usb_read            = usb30_usb_read;
    usb_if.usb_write           = usb30_usb_write;
#else
    dprintf(CRITICAL, "USB30 needs to be enabled for this target.\n");
    ASSERT(0);
#endif
  }
  else
  {
    /* initialize udc functions to use the default chipidea controller */
    usb_if.udc_init            = udc_init;
    usb_if.udc_register_gadget = udc_register_gadget;
    usb_if.udc_start           = udc_start;
    usb_if.udc_stop            = udc_stop;
​
    usb_if.udc_endpoint_alloc  = udc_endpoint_alloc;
    usb_if.udc_request_alloc   = udc_request_alloc;
    usb_if.udc_request_free    = udc_request_free;
​
    usb_if.usb_read            = hsusb_usb_read;
    usb_if.usb_write           = hsusb_usb_write;
  }
​
  /* register udc device */
  usb_if.udc_init(&surf_udc_device);
​
  event_init(&usb_online, 0, EVENT_FLAG_AUTOUNSIGNAL);
  event_init(&txn_done, 0, EVENT_FLAG_AUTOUNSIGNAL);
​
  in = usb_if.udc_endpoint_alloc(UDC_TYPE_BULK_IN, 512);
  if (!in)
    goto fail_alloc_in;
  out = usb_if.udc_endpoint_alloc(UDC_TYPE_BULK_OUT, 512);
  if (!out)
    goto fail_alloc_out;
​
  fastboot_endpoints[0] = in;
  fastboot_endpoints[1] = out;
​
  req = usb_if.udc_request_alloc();
  if (!req)
    goto fail_alloc_req;
​
  /* register gadget */
  if (usb_if.udc_register_gadget(&fastboot_gadget))
    goto fail_udc_register;
​
  fastboot_register("getvar:", cmd_getvar);
  fastboot_register("download:", cmd_download);
  fastboot_publish("version", "0.5");
​
  thr = thread_create("fastboot", fastboot_handler, 0, DEFAULT_PRIORITY, 4096);
  if (!thr)
  {
    goto fail_alloc_in;
  }
  thread_resume(thr);
​
  usb_if.udc_start();
​
  return 0;
​
fail_udc_register:
  usb_if.udc_request_free(req);
fail_alloc_req:
  usb_if.udc_endpoint_free(out);
fail_alloc_out:
  usb_if.udc_endpoint_free(in);
fail_alloc_in:
  return -1;
}
```

**fastboot\_init()主要工作是初始化usb\_controller\_interface,然后调用usb\_if.udc\_start()监听usb event,在新建线程等待usb event并调用fastboot\_handler处理。fastboot\_init后display\_fastboot\_menu\_thread()将会在新线程中被调用,即用来在LCD上显示fastboot菜单。**

```
static int fastboot_handler(void *arg)
{
  for (;;) {
    event_wait(&usb_online);
    fastboot_command_loop();
  }
  return 0;
}
```

**fastboot\_handler中等待usb事件，通过usb\_if.usb\_read()读取usb数据，并检索出对应cmdlist里的命令调用。**

```
static void fastboot_command_loop(void)
{
  struct fastboot_cmd *cmd;
  int r;
#if CHECK_BAT_VOLTAGE
  boolean is_first_erase_flash = false;
#endif
​
  dprintf(INFO,"fastboot: processing commands\n");
​
  uint8_t *buffer = (uint8_t *)memalign(CACHE_LINE, ROUNDUP(4096, CACHE_LINE));
  if (!buffer)
  {
    dprintf(CRITICAL, "Could not allocate memory for fastboot buffer\n.");
    ASSERT(0);
  }
again:
  while (fastboot_state != STATE_ERROR) {
​
    /* Read buffer must be cleared first. If buffer is not cleared,
     * the original data in buf trailing the received command is
     * interpreted as part of the command.
     */
    memset(buffer, 0, MAX_RSP_SIZE);
    arch_clean_invalidate_cache_range((addr_t) buffer, MAX_RSP_SIZE);
​
    r = usb_if.usb_read(buffer, MAX_RSP_SIZE);
    if (r < 0) break;
    buffer[r] = 0;
    dprintf(INFO,"fastboot: %s\n", buffer);
​
#if CHECK_BAT_VOLTAGE
    /* check battery voltage before erase or flash image */
    if (!strncmp((const char*) buffer, "getvar:partition-type", 21))
      is_first_erase_flash = true;
​
    if (is_first_erase_flash) {
      if (!strncmp((const char*) buffer, "erase", 5) ||
        !strncmp((const char*) buffer, "flash", 5)) {
        if (!target_battery_soc_ok()) {
          dprintf(INFO,"fastboot: battery voltage: %d\n",
            target_get_battery_voltage());
          fastboot_fail("Warning: battery's capacity is very low\n");
          return;
        }
      }
    }
#endif
​
    fastboot_state = STATE_COMMAND;
​
    for (cmd = cmdlist; cmd; cmd = cmd->next) {
      if (memcmp(buffer, cmd->prefix, cmd->prefix_len))
        continue;
      cmd->handle((const char*) buffer + cmd->prefix_len,
            (void*) download_base, download_size);
      if (fastboot_state == STATE_COMMAND)
        fastboot_fail("unknown reason");
​
#if CHECK_BAT_VOLTAGE
      if (!strncmp((const char*) buffer, "erase", 5) ||
        !strncmp((const char*) buffer, "flash", 5)) {
        if (is_first_erase_flash) {
          is_first_erase_flash = false;
        }
      }
#endif
      goto again;
    }
​
    fastboot_fail("unknown command");
​
  }
  fastboot_state = STATE_OFFLINE;
  dprintf(INFO,"fastboot: oops!\n");
  free(buffer);
}
```

## **5. Device tree**

Device tree是描述设备硬件信息的数据结构，通过.dts源文件定义，一般位于kernel/arch/arm/boot/dts目录下, .dts编译生成.dtb文件后将LK传递给kernel执行设备初始化工作。​!\[Device Tree]\(https://github.com/XRobinHe/Resource/blob/master/blog/image/android/device\_tree.png?raw=true)​

**关于Device Tree参考宋宝华老师的博文** [**ARM Linux 3.x的设备树（Device Tree）**](http://blog.csdn.net/21cnbao/article/details/8457546)**。**
