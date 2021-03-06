SYS ?= $(shell gcc -dumpmachine)
export SYS

export DEBUG

GLIBC_SRC = glibc-2.19
GLIBC_CHECKSUM = 18ad6db70724699d264add80b1f813630d0141cf3a3558b4e1a7c15f6beac796
SHIM_DIR = shim
BUILD_DIR = glibc-build
RUNTIME_DIR = $(CURDIR)/../Runtime
GLIBC_LIBS = \
	csu/crt1.o \
	csu/crti.o \
	csu/crtn.o \
	dlfcn/libdl.so.2 \
	elf/ld-linux-x86-64.so.2 \
	libc.so \
	libc.so.6 \
	libos/liblibos.so.1 \
	login/libutil.so.1 \
	math/libm.so.6 \
	nptl/libpthread.so.0 \
	nptl_db/libthread_db.so.1 \
	resolv/libnss_dns.so.2 \
	resolv/libresolv.so.2 \
	rt/librt.so.1
GLIBC_TARGET = $(addprefix $(BUILD_DIR)/, $(GLIBC_LIBS))
GLIBC_RUNTIME = $(addprefix $(RUNTIME_DIR)/, $(notdir $(GLIBC_TARGET)))

.PHONY: all regression
all regression: $(GLIBC_TARGET) $(GLIBC_RUNTIME)
	$(MAKE) -C $(SHIM_DIR) $@

include ../Makefile.rules

.PHONY: format
format:
	$(MAKE) -C $(SHIM_DIR) format

ifeq ($(findstring x86_64,$(SYS))$(findstring linux,$(SYS)),x86_64linux)

.SECONDARY: $(BUILD_DIR)/Build.success

$(BUILD_DIR)/Build.success: $(BUILD_DIR)/Makefile
	@echo "Building glibc, may take a while to finish. Warning messages may show up. If this process terminates with failures, see \"$(BUILD_DIR)/build.log\" for more information."
	($(MAKE) -C $(BUILD_DIR) 2>&1 >> build.log) && touch $@
#  2>&1 | tee -a build.log)

$(GLIBC_TARGET): $(BUILD_DIR)/Build.success

$(BUILD_DIR)/Makefile: $(GLIBC_SRC)/configure
ifeq ($(DEBUG),1)
	./buildglibc.py --quiet --debug
else
	./buildglibc.py --quiet
endif

define LN_SF_TO_RUNTIME_DIR_template =
$(RUNTIME_DIR)/$(notdir $(1)): $(1)
	$$(call cmd,ln_sf)
endef

$(foreach lib,$(GLIBC_TARGET),$(eval $(call LN_SF_TO_RUNTIME_DIR_template,$(lib))))

GLIBC_MIRRORS ?= https://ftp.gnu.org/gnu/ \
		https://mirrors.kernel.org/gnu/ \
		https://mirrors.ocf.berkeley.edu/gnu/

ifeq ($(shell git ls-files $(GLIBC_SRC)/configure),)
GLIBC_PATCHES = \
	$(GLIBC_SRC).patch \
	glibc-syscalldb-api.patch \
	glibc-fix-warning.patch \
	glibc-no-pie.patch \
	glibc-liblibos.patch \
	01-glibc-gettimeofday.patch \
	02-glibc-sched-getscpu-S.patch \
	03-glibc-time-c.patch \
	04-pthread-cond-timedwait-S.patch

ifneq ($(filter %.gold,$(shell readlink -f /usr/bin/ld)),)
GLIBC_PATCHES += glibc-ld.gold.patch
endif

$(GLIBC_SRC)/configure: $(GLIBC_PATCHES) $(GLIBC_SRC).tar.gz
	[ "`sha256sum $(GLIBC_SRC).tar.gz`" = "$(GLIBC_CHECKSUM)  $(GLIBC_SRC).tar.gz" ] || \
		(echo "*** $(GLIBC_SRC).tar.gz has a wrong checksum ***"; exit 255)
	rm -rf $(GLIBC_SRC)
	tar -xzf $(GLIBC_SRC).tar.gz
	cd $(GLIBC_SRC) && \
	for p in $(GLIBC_PATCHES); do \
		echo applying $$p; \
		patch -p1 < ../$$p || exit 255; \
	done
	touch $@
endif

$(GLIBC_SRC).tar.gz:
	for MIRROR in $(GLIBC_MIRRORS); do \
		wget --timeout=10 $${MIRROR}glibc/$(GLIBC_SRC).tar.gz \
		&& break; \
	done

$(GLIBC_SRC)/elf/Versions: $(GLIBC_SRC)/configure

$(GLIBC_SRC)/nptl/Versions: $(GLIBC_SRC)/configure

$(GLIBC_SRC)/dlfcn/Versions: $(GLIBC_SRC)/configure

.PHONY: clean
clean:
	$(MAKE) -C $(SHIM_DIR) clean
	rm -rf $(BUILD_DIR)

else
.IGNORE: $(GLIBC_TARGET)
$(GLIBC_TARGET):

.PHONY: clean
clean:
	rm -rf $(BUILD_DIR)
endif
