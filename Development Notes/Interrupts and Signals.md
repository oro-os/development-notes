Interrupts and signals in Oro have been rattling around in my head for the better part of five years at this point.

Now that I've started to get some of the timekeeping bits ([[HPET Timings]]) and device enumeration bits working, I have to start thinking about how these will be implemented in practice and how they'll be used from a developer standpoint.

One of the overarching goals of this system - however it may look - is to be """safer""" than POSIX signals, which have a [well known set of footguns](