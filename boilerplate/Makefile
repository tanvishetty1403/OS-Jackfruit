# Makefile — Multi-Container Runtime
# Builds both the user-space engine and the kernel module with a single `make`

# ── User-space ─────────────────────────────────────────────────────────────────
CC      = gcc
CFLAGS  = -Wall -Wextra -g -D_GNU_SOURCE
LDFLAGS = -lpthread

ENGINE_SRC  = engine.c
ENGINE_BIN  = engine

WORKLOADS   = cpu_hog io_pulse memory_hog

# ── Kernel module ───────────────────────────────────────────────────────────────
obj-m       += monitor.o
KDIR        := /lib/modules/$(shell uname -r)/build
PWD         := $(shell pwd)

# ── Default target ──────────────────────────────────────────────────────────────
.PHONY: all clean load unload

all: $(ENGINE_BIN) workloads
	$(MAKE) -C $(KDIR) M=$(PWD) modules

# Engine binary
$(ENGINE_BIN): $(ENGINE_SRC) monitor_ioctl.h
	$(CC) $(CFLAGS) -o $@ $< $(LDFLAGS)

# Workload test programs
.PHONY: workloads
workloads: $(WORKLOADS)

cpu_hog: cpu_hog.c
	$(CC) $(CFLAGS) -o $@ $<

io_pulse: io_pulse.c
	$(CC) $(CFLAGS) -o $@ $<

memory_hog: memory_hog.c
	$(CC) $(CFLAGS) -o $@ $<

# ── Kernel module helpers ───────────────────────────────────────────────────────
load:
	sudo insmod monitor.ko
	@echo "Module loaded. Device: /dev/container_monitor"

unload:
	sudo rmmod monitor
	@echo "Module unloaded."

# ── Clean ───────────────────────────────────────────────────────────────────────
clean:
	rm -f $(ENGINE_BIN) $(WORKLOADS)
	$(MAKE) -C $(KDIR) M=$(PWD) clean
