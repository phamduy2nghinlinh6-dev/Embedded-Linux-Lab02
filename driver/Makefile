ccflags-y := -march=armv7-a
obj-m += lab2_driver.o

KDIR ?= /home/duy/embedded_lab1/kernel/linux-5.15
CROSS_COMPILE ?= arm-linux-gnueabihf-
ARCH ?= arm

all:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) clean
