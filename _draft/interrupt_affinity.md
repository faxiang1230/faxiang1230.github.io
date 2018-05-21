I need one help.
I am using i.MX7 Sabre board with kernel version 4.1.15

Let's say I am interested in GPIO number: 21
I wanted to set CPU affinity for particular GPIO->IRQ number, so I tried
the below steps:
root@10:~# echo 21 > /sys/class/gpio/export
root@10:~# echo "rising" > /sys/class/gpio/gpio21/edge
root@10:~# cat /proc/interrupts | grep 21
  47: 0 0 gpio-mxc 21 Edge gpiolib
root@10:~# cat /sys/class/gpio/gpio21/direction
in
root@10:~# cat /proc/irq/47/smp_affinity
3
root@10:~# echo 2 > /proc/irq/47/smp_affinity
-bash: echo: write error: Input/output error

But I get input/output error.
When I debug further, found that irq_can_set_affinity is returning 0:
[    0.000000] genirq: irq_can_set_affinity (0): balance: 1, irq_data.chip:
a81b7e48, irq_set_affinity:   (null)
[    0.000000] write_irq_affinity: FAIL

I also tried first setting /proc/irq/default_smp_affinity to 2 (from 3).
This change is working, but the smp_affinity setting for the new IRQ is not
working.

When I try to set smp_affinity for mmc0, then it works.
# cat /proc/interrupts | grep mmc
295:         55          0     GPCV2  22 Edge      mmc0
296:          0          0     GPCV2  23 Edge      mmc1
297:         52          0     GPCV2  24 Edge      mmc2

root@10:~# echo 2 > /proc/irq/295/smp_affinity
root@10:~#


So, I wanted to know what are the conditions for which setting smp_affinity
for an IRQ will work ?

Is there any way by which I can set CPU affinity to a GPIO -> IRQ ?
Whether, irq_set_affinity_hint() will work in this case ?

>> IRQ affinity is only supported where interrupts are _directly_ wired to
>> the GIC.  It's the GIC which does the interrupt steering to the CPU
>> cores.
>>
>> Interrupts on downstream interrupt controllers (such as GPCV2) have no
>> ability to be directed independently to other CPUs - the only possible
>> way to change the mapping is to move _all_ interrupts on that controller,
>> and any downstream chained interrupts at GIC level.
>>
>> Hence why Interrupt 295 has no irq_set_affinity function: there is no way
>> for the interrupt controller itself to change the affinity of the input
>> interrupt.
>
> The GPCv2 though is a secondary IRQ controller which has a 1:1 mapping
> of its input IRQs to the upstream GIC IRQ lines. Affinity can thus be
> handled by forwarding the request to the GIC by
> irq_chip_set_affinity_parent().
>
> As this is handled correctly in the upstream kernel since the first
> commit introducing support for the GPCv2, it seems the issue is only
> present in some downstream kernel.
>

OK. Thanks so much for your reply.

I saw some of the drivers using irq_set_affinity_hint() to force the
IRQ affinity to a particular CPU.
This is the sample:
{
cpumask_clear(mask);
cpumask_set_cpu(cpu, mask);
irq_set_affinity_hint(irq, mask);
}

Whether this logic will work for a particular GPIO pin ?
