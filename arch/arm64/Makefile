# arch/arm64/Makefile
# Optimized for Gravity-perf (Raphael)

LDFLAGS_vmlinux	:=--no-undefined -X -z norelro
CPPFLAGS_vmlinux.lds = -DTEXT_OFFSET=$(TEXT_OFFSET)
GZFLAGS		:=-9

ifeq ($(CONFIG_RELOCATABLE), y)
LDFLAGS_vmlinux		+= -shared -Bsymbolic -z notext \
			$(call ld-option, --no-apply-dynamic-relocs)
endif

ifeq ($(CONFIG_ARM64_ERRATUM_843419),y)
  ifeq ($(call ld-option, --fix-cortex-a53-843419),)
$(warning ld does not support --fix-cortex-a53-843419; kernel may be susceptible to erratum)
  else
LDFLAGS_vmlinux	+= --fix-cortex-a53-843419
  endif
endif

KBUILD_DEFCONFIG := defconfig

lseinstr := $(call as-instr,.arch_extension lse,-DCONFIG_AS_LSE=1)

ifeq ($(CONFIG_COMPAT_VDSO), y)
  CROSS_COMPILE_ARM32 ?= $(CONFIG_CROSS_COMPILE_ARM32:"%"=%)
  ifeq ($(CROSS_COMPILE_ARM32),)
    $(error CROSS_COMPILE_ARM32 not defined or empty)
  else ifeq ($(cc-name),clang)
    export CLANG_TRIPLE_ARM32 ?= $(CROSS_COMPILE_ARM32)
    export CLANG_TARGET_ARM32 := --target=$(notdir $(CLANG_TRIPLE_ARM32:%-=%))
    export CONFIG_VDSO32 := y
    vdso32 := -DCONFIG_VDSO32=1
  endif
endif

ifeq ($(cc-name),clang)
KBUILD_CFLAGS	+= -mno-implicit-float
else
KBUILD_CFLAGS	+= -mgeneral-regs-only
endif

KBUILD_CFLAGS	+= $(lseinstr) $(brokengasinst) $(vdso32)
KBUILD_CFLAGS	+= -fno-asynchronous-unwind-tables
KBUILD_CFLAGS	+= $(call cc-option, -mpc-relative-literal-loads)
KBUILD_CFLAGS	+= -fno-pic

# =========================================================
# GRAVITY-PERF OPTIMIZATION BLOCK (SD855 / Raphael)
# =========================================================
ifeq ($(cc-name),clang)
    # Enable O3 (Extreme Optimization)
    KBUILD_CFLAGS := $(subst -O2,-O3,$(KBUILD_CFLAGS))
    
    # Target Snapdragon 855 Architecture specifically
    KBUILD_CFLAGS += -march=armv8.2-a+crc+crypto -mcpu=kryo485
    
    # Advanced Loop & Math Optimizations
    KBUILD_CFLAGS += -O3 -pipe -ftree-vectorize -funroll-loops \
                     -fstrict-aliasing -fmodulo-sched -fmodulo-sched-allow-regmoves \
                     -fgraphite-identity -ffp-contract=fast
endif
# =========================================================

KBUILD_AFLAGS	+= $(lseinstr) $(brokengasinst) $(vdso32)
KBUILD_CFLAGS	+= $(call cc-option,-mabi=lp64)
KBUILD_AFLAGS	+= $(call cc-option,-mabi=lp64)

ifeq ($(CONFIG_SHADOW_CALL_STACK), y)
KBUILD_CFLAGS	+= -ffixed-x18
endif

ifeq ($(cc-name),clang)
KBUILD_CFLAGS	+= -DCONFIG_ARCH_SUPPORTS_INT128
else
KBUILD_CFLAGS	+= $(call cc-ifversion, -ge, 0500, -DCONFIG_ARCH_SUPPORTS_INT128)
endif

ifeq ($(CONFIG_CPU_BIG_ENDIAN), y)
KBUILD_CPPFLAGS	+= -mbig-endian
AS		+= -EB
LD		+= -EB
UTS_MACHINE	:= aarch64_be
else
KBUILD_CPPFLAGS	+= -mlittle-endian
AS		+= -EL
LD		+= -EL
UTS_MACHINE	:= aarch64
endif

head-y		:= arch/arm64/kernel/head.o
TEXT_OFFSET := 0x00080000

export	TEXT_OFFSET GZFLAGS

core-y		+= arch/arm64/kernel/ arch/arm64/mm/
core-$(CONFIG_NET) += arch/arm64/net/
core-$(CONFIG_CRYPTO) += arch/arm64/crypto/
libs-y		:= arch/arm64/lib/ $(libs-y)

ifeq ($(CONFIG_BUILD_ARM64_KERNEL_COMPRESSION_GZIP),y)
KBUILD_IMAGE   := Image.gz
else
KBUILD_IMAGE   := Image
endif

boot		:= arch/arm64/boot
ifeq ($(CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE),y)
KBUILD_TARGET  := $(addsuffix -dtb,$(KBUILD_IMAGE))
KBUILD_IMAGE	:= $(boot)/$(addsuffix -dtb,$(KBUILD_IMAGE))
else
KBUILD_TARGET := $(KBUILD_IMAGE)
KBUILD_IMAGE  := $(boot)/$(KBUILD_IMAGE)
endif

all:	dtbs $(KBUILD_TARGET)

Image: vmlinux
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

Image.%: Image
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

dtbs: prepare scripts
	$(Q)$(MAKE) $(build)=$(boot)/dts

Image.gz-dtb: vmlinux scripts dtbs Image.gz
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

archclean:
	$(Q)$(MAKE) $(clean)=$(boot)
	$(Q)$(MAKE) $(clean)=$(boot)/dts
