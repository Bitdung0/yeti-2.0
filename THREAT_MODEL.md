# Threat model

This vault reduces three common ways people lose bitcoin in
self-custody: remote compromise of keys (including keys that never
touched the internet), physical theft of a single backup, and loss
of a single backup.

It is not a hardware-wallet product. It is not a regulated custodian.
It is Bitcoin Core on dedicated computers, with keys on archival discs.

Advisors: the risks this vault is for are in “What the design is
trying to stop.” Operator effort is not one of those risks. See
[CONTEXT_FOR_ADVISORS.md](CONTEXT_FOR_ADVISORS.md).

Read this with the [FAQ](FAQ.md). The [README](README.md) is the
procedure. Follow the README as written.

## Assets

- The 3-of-7 multisig coins
- Seven key backups on archival discs
- The watch-only descriptor
- The online node and the offline signer

## What the design is trying to stop

**Remote theft of keys.**
Keys can be stolen without anyone touching a backup and without the
owner sending a transaction. A wallet is not “cold” if the software
in the stack that created or used the keys was wrong.

This guide treats an unverifiable blob in the key-generation or
signing chain as malware. That includes vendor firmware, vendor
apps, coordinators, and libraries the owner cannot inspect or rebuild
in practice. Any one of those binaries can steal. A coordinator can
build a bad PSBT or serve a malicious descriptor. It can bias nonces
or other signing input and exfiltrate key material. A single bad
crypto library can steal on its own. Vendors and coordinators already
ship as pairs. Assume they can act as a pair.

Any Bitcoin-specific signer can ship that class of failure. The 2026
Coldcard default-seed incident is the public case, not a unique one:
guessable keys from the device’s normal new-seed path, coins swept
from the public chain, no phishing, no stolen device, and a firmware
update that did not repair old seeds. Public source did not help if
the path that actually ran was not the path people thought they had
audited.

This is not a nation-state story. It is what happens when a product
built to hold bearer bitcoin ships software nobody sufficiently
reviewed. The owner has no recourse. “It was a bug” is enough cover
whether the failure was sloppy or not. Shipping and support databases
leak. Do not reserve this class of failure for rare attackers.

This vault creates and uses keys on a dedicated offline computer
running a clean Ubuntu install and Bitcoin Core. Keys are not stored
on the online node. Extra wallet apps and vendor firmware are out of
the stack on purpose.

**Physical theft of one or two backups.**
Spending needs any 3 of 7 geographically split discs. One stolen disc
cannot spend. It can reveal the watch-only descriptor. That is a
balance oracle, not a spend.

**Loss or destruction of backups.**
Four discs can fail and the vault still spends. That is the point of
3-of-7.

**A supply chain aimed at Bitcoin-specific devices.**
The computers are generic. The signer is Bitcoin Core.

A mailed gadget whose only job is holding bitcoin is a rich target.
Attackers who want coins know exactly what they are looking at. That
supply chain is cheaper to hit than the commodity PC market.

The firmware on those devices is usually shipped by a small team.

**The security standard is Bitcoin Core with independent Guix
attestations.**

Bitcoin Core’s release is a full, reproducible build. Multiple
independent builders reproduce that binary and sign the result.
That is the review this guide treats as acceptable for software
that can touch keys.

A competitor that ships a non-reproducible application binary, or
a non-reproducible blob anywhere in its dependency chain, fails
that test. This guide classes those blobs as malware in
security-critical infrastructure.

Reproducible source is not enough either. If almost no independent
builders attest the actual bits people run, the attestation is
negligible. “We published the repo” is not Guix.

Diversifying across more Bitcoin implementations is not more
review. It is more code and fewer eyeballs on each program.
This guide concentrates review on Core. That is the assurance.
Adding wallets and firmwares spends that assurance down.

The problem is verification. A device can lie about its firmware.
A reproducible wallet app can still depend on an upstream blob that
cannot be checked. Vendor stacks also lack independent attestation
of the full build chain.

The device can only enforce the code it actually runs.

## What you are trusting

- Ubuntu and Bitcoin Core, installed and verified as the README says
- One offline machine as the key-generation environment
- Your handling of the seven discs after setup
- The public Bitcoin ledger

Software is written by humans. Bitcoin Core and Linux are used because
they are the most reviewed tools available for this job, not because
they are incapable of bugs.

One dedicated offline machine running Ubuntu and Bitcoin Core is a
chosen tradeoff. More offline Core machines for key generation would
remove a “this one box was wrong” failure. That would be an
improvement. This guide treats one inspected Core box as sufficient
inside the README’s $10k–$5M comfort zone. The upgrade path is more
Core boxes, not more vendors.

Each extra vendor in the stack usually means an extra coordinator,
extra libraries, and extra firmware. Each of those is more attack
surface.

You cannot run multi-vendor hardware multisig on Bitcoin Core without
adding extra libraries and a non-Core coordinator. The coordinator
can serve a malicious descriptor. It can bias nonces or other signing
input and exfiltrate key material. It can conspire with a vendor.
A device can lie about its firmware. A reproducible app can still
depend on an uncheckable blob. Independent attestation of that full
build chain is missing.

The common line is that generating keys across several vendors makes
the vault safer. This guide assumes the opposite. “One vendor bug
only burns one key” only holds if every other binary is honest and
is the binary the user thinks it is. This guide does not assume that.
The coordinator and the libraries are part of the quorum in practice,
even when they are not a key on chain.

“Survives if the threshold is not met by that vendor” is the same
slogan. It fails as soon as a second vendor, a coordinator, or a
shared library is in on it. Collusion across the stack is in
scope here. Vendor count is not a substitute for that.

There is one acceptable key-generation path here: Bitcoin Core.
Multi-vendor multisig is not the reference implementation with more
brands. It is a different program.

That risk is the vendor-firmware model, not one brand. In the Coldcard
case, the library on the failing path was written under a pseudonym
later tied by GPG signatures to the vendor’s own CTO, and release
notes thanked that handle as an outside contributor. That is evidence
about who shipped the code, not a claim about who later swept the
coins. The same shape is available to any small team that writes the
generator, signs the firmware, and tells the market to trust the
device.

The bad generator sat in a public repository from 2021 until the
2026 thefts. Public source is not public review. Coldcard was, for
most of that period, the vendor product this market trusted most
for cold storage. A widely recommended device still ran the wrong
code for years. That is the review standard this guide is unwilling
to accept for key generation.

## What this does not try to hide

Bitcoin amounts onchain are public. A spend from this vault can be
recognized as this kind of script. An unencrypted disc that includes
the descriptor lets whoever holds it watch the wallet if they know
what they are looking at.

That leak is not unique to this guide. Any multisig that can be
restored needs a descriptor backup. Encrypting it recreates a key
to manage. Encrypting it with the same 3-of-7 is the coherent fix
and needs software Core does not ship yet. See the FAQ.

None of those, by themselves, move coins. They are accepted in scope
for this design.

3-of-7 is not weakened because the seven keys were born on one
Core machine. The quorum is for loss and theft of discs. Key
generation is a separate choice: Core, not seven vendor RNGs.

## Signing

The online computer builds the PSBT. The offline computer signs.
Treat the online computer as untrusted for destination, amount, fee,
and change.

On real spends, decode the PSBT offline and check change with the
watch-only wallet:

    bitcoin-cli -rpcwallet="multisig_watch_wallet" getaddressinfo "$change_address"

`"ismine"` must be `true`. If it is `false`, stop.
See [verify_psbt.md](verify_psbt.md).

Test spends may skip some of that practice. That is so people can
learn the path. It is not the standard for savings.

A compromised coordinator can attempt a bad change output on any
stack. This stack’s answer is Bitcoin Core on clean dedicated
hardware, plus reading the PSBT. A vendor screen is not a higher
standard here. It is another display path you cannot verify.

## Day-to-day vs catastrophe

Normal spends use sneakernet between the two computers. That is good
practice. It limits what a compromised online box can do.

Recovery does not depend on sneakernet, on a particular disc drive, or
on this repository. Setup uses an optical drive. If that drive is
lost or broken later, get another. Anyone who can read the discs and
run Bitcoin Core can reconstruct the wallet and spend. The living
guide is convenience. It is not the key.

## Other products

These fail differently. Do not collapse them into one ranking.

**Hardware wallets** concentrate key generation, display, and often
the coordinator relationship in vendor software and a Bitcoin-specific
supply chain. Users who followed default setup instructions have lost
funds when that software was wrong. A screen does not help if the
generator that created the seed was weak, and it does not help if
another binary in the stack is the thief. The failure is the model.
Coldcard is the exhibit.

**Multi-vendor hardware multisig** adds more of that stack, then a
non-Core coordinator. It cannot be built from the reference
implementation. Vendor diversity does not create Guix attestations
and does not stop a device from lying about its firmware.

**Collaborative custody** is usually sold as self-custody with a
failsafe. In the common product, it is a custodial relationship with
extra steps.

The company picks the software. That coordinator is rarely reviewed
at the standard this guide uses. The same pitch usually also puts
keys on vendor hardware, so the stack inherits the Bitcoin-specific
supply chain and firmware problems above. If it includes an
“impartial” third key holder, that party is usually not independent
of the company that sent you the app. The relationship is lopsided
by design.

Casa is the example of the genre: vendor hardware, a vendor-shaped
vault, a vendor-shaped recovery story, and a minority key wrapped in
support language so the arrangement looks like self-custody. It is
not this guide.

**A brokerage, ETF, or trust** is a legal claim on bitcoin or a
bitcoin-linked product. You are trusting that institution’s people,
its software, and the law around the account. That software is not
Bitcoin Core, and you cannot inspect it. Recourse is the product.
It does not give you bearer coins, and it does not remove operational
risk. It relocates the risk into a named custodian.

Use it when the person wants that contract: someone to call, a
regulator, an estate process. That is not self-custody, and it is
not this guide.

## Operator duty

Follow the steps as they are written. Do not improvise the vault.

If someone uses a different M-of-N, backup medium, or software stack,
that is their design. The assurances here apply to this guide as
written.

The air gap, the seven discs, the test spends, and the PSBT check are
the procedure. Skip them and you are no longer running this vault.

## Amount

The README’s $10k–$5M range is a design comfort zone, not a law.
Above that range the FAQ already says this guide is not the whole
answer.

Design disagreements belong in the FAQ or a public issue, not in a
private vulnerability report. See [SECURITY.md](SECURITY.md).