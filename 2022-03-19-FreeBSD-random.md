We should borrow good ideas from Linux where we can.  In that spirit, here's a short enumeration of top-level changes from [Jason's recent article](https://www.zx2c4.com/projects/linux-rng-5.17-5.18/) and how they relate to FreeBSD.  (Caveat: I am not a cryptographer.)

* Readability
    - Existing code in FreeBSD is reasonably well documented.  Comments referance relevant sections of FS&K (Fortuna) or Ferguson (Win10Rng).

* /dev/[u]random unification
    - These devices have been the same on FreeBSD for a long time.

* Jitter entropy initial seeding
    - FreeBSD should adopt this, but hasn't implemented it yet.

* Reinitialization on VM fork (VM generation ID)
    - FreeBSD implemented this in late 2019 ([ffac39deae0a](https://github.com/freebsd/freebsd-src/commit/ffac39deae0a), [767991d2bed3](https://github.com/freebsd/freebsd-src/commit/767991d2bed3)).

* Extension of VM fork notification to other subsystems, e.g., Wireguard
    - FreeBSD should probably adopt this, too.

* CPU hotplug
    - FreeBSD does not support this, as far as I know.

* Fast pool mixing deferred to worker
    - FreeBSD already asynchronously processes entropy input via [`random_harvest_queue()`](http://fxr.watson.org/fxr/ident?i=random_harvest_queue_) and [`random_harvest_fast()`](http://fxr.watson.org/fxr/ident?i=random_harvest_fast) in the "rand_harvestq" kthread.

* Blake2s instead of SHA1 in extraction
    - In FreeBSD, we use SHA256 (or Blake2) for our pool primitives, instead of a non-cryptographic mixing function.  When we want output from the pools, we just take the finalized digest from one or more pools and digest them together with SHA256 (or Blake2).

* Chacha-based CRNG
    - FreeBSD uses a Chacha-based generator since 2019 ([ab69c4858cb7](https://github.com/freebsd/freebsd-src/commit/ab69c4858cb7), [68b97d40fbe8](https://github.com/freebsd/freebsd-src/commit/68b97d40fbe8)).

* Entropy crediting simplification
    - FreeBSD's implementation of Fortuna already makes the same simplification, that entropy is essentially additive via the cryptographic accumulator (SHA256 or Blake2).

* Use of Siphash as an interrupt entropy accumulator
    - FreeBSD initially accumulates interrupt entropy with non-cryptographic Jenkins hash.  This may be less desireable than SipHash; I'm not really familiar.  The output from Jenkins hash is collected asynchronously by the "rand_harvestq" worker thread every 100 ms into the SHA256 pools.

* Use of RDSEED instead of RDRAND in extraction
    - FreeBSD does this since 2019 ([2cb54a800c3a](https://github.com/freebsd/freebsd-src/commit/2cb54a800c3a)).

* RDSEED is just another entropy input
    - FreeBSD has always treated the various entropy sources as basically the same, with the exception of the CPU timecounter (which is mixed in with all other entropy sources).

* CRNG: simpler fast key erasure flow on per-cpu keys
    - FreeBSD adds a fast key-erasure version of Fortuna in [179f62805cf0](https://github.com/freebsd/freebsd-src/commit/179f62805cf0) (2019) (default in [548dca90ae2f](https://github.com/freebsd/freebsd-src/commit/548dca90ae2f), 2019), which only holds the shared lock long enough to copy the current state and bump the shared key (see [`random_fortuna_read_concurrent()`](https://github.com/freebsd/freebsd-src/blob/4312ebfe0bbf314a0d5d1b6d14d003673255dd0d/sys/dev/random/fortuna.c#L681)).
    - FreeBSD adds per-CPU generators using essentially the same base -> per-cpu generator versioning scheme based on the Windows 10 RNG design in 2020 ([a3c41f8bfbc7](https://github.com/freebsd/freebsd-src/commit/a3c41f8bfbc7)).

* Rapid boot-time reseeding, slowing to 5 minutes as the uptime grows
    - Similar design added to FreeBSD in 2020 ([a3c41f8bfbc7](https://github.com/freebsd/freebsd-src/commit/a3c41f8bfbc7)), based on Windows 10 design (powers of three seconds, up to an hour).  However, this algorithm is not the default.  The default Fortuna algorithm will reseed every 100ms if there is sufficient estimated new entropy.

* Premature next
    - Fortuna enforces 64 bytes of input entropy (512 bits) in the zero pool between reseeds.  (By default -- it is admin-configurable with a "minpoolsize" tuneable/sysctl).  The rough heuristic here is that input entropy sources are anticipated to have at least 4 bits-per-byte of min entropy, so this should be 256 bits of "real" entropy.
    - The Win10-based algorithm in FreeBSD is just timer based, and the timer interval quickly grows such that there is a long time between reseeds, like Linux.
    - My main concern around "premature next" is the initial seeding step, where it's hard to tell if we have enough entropy.  Many kernel random values are initialized once from this initial seed, and if this is predictable, it would break the designs using these numbers.
