[![Latest Version](https://img.shields.io/crates/v/neli.svg)](https://crates.io/crates/neli) [![Documentation](https://docs.rs/neli/badge.svg)](https://docs.rs/neli)

# neli
Type safe netlink library for Rust

As of version 0.4.0, completeness of autogenerated documentation
and examples will be a focus. Please open issues if something is
missing or unclear!

## API documentation
API documentation can be found [here](https://docs.rs/neli/)

## Goals

This library aims to cover as many of the netlink subsystems as
possible and provide ways to extend neli for anything that is not
within the scope of support for this library.

This is also a pure Rust implementation and aims to make use of
idiomatic Rust features.

## Examples using `neli`

Examples of working code exist in the `examples/` subdirectory on
Github. They have a separate `Cargo.toml` file to provide easy
testing and use.  

Workflows seem to usually follow a pattern of socket creation,and
then either sending and receiving messages in request/response
formats:

```rust
use std::error::Error;

use neli::{
    consts::{genl::*, nl::*, socket::*},
    err::NlError,
    genl::{Genlmsghdr, Nlattr},
    nl::{Nlmsghdr, NlPayload},
    socket::NlSocketHandle,
    types::{Buffer, GenlBuffer},
};

const GENL_VERSION: u8 = 1;

fn request_response() -> Result<(), Box<dyn Error>> {
    let mut socket = NlSocketHandle::connect(
        NlFamily::Generic,
        None,
        &[],
    )?;

    let attrs: GenlBuffer<Index, Buffer> = GenlBuffer::new();
    let genlhdr = Genlmsghdr::new(
        CtrlCmd::Getfamily,
        GENL_VERSION,
        attrs,
    );
    let nlhdr = {
        let len = None;
        let nl_type = GenlId::Ctrl;
        let flags = NlmFFlags::new(&[NlmF::Request, NlmF::Dump]);
        let seq = None;
        let pid = None;
        let payload = NlPayload::Payload(genlhdr);
        Nlmsghdr::new(len, nl_type, flags, seq, pid, payload)
    };
    socket.send(nlhdr)?;
    
    // Do things with multi-message response to request...
    let mut iter = socket.iter::<NlTypeWrapper, Genlmsghdr<CtrlCmd, CtrlAttr>>(false);
    while let Some(Ok(response)) = iter.next() {
        // Do things with response here...
    }
    
    // Or get single message back...
    let msg = socket.recv::<Nlmsg, Genlmsghdr<CtrlCmd, CtrlAttr>>()?;

    Ok(())
}
```

or a subscriptions to a stream of event notifications from netlink:

```rust
use std::error::Error;

use neli::{
    consts::{genl::*, nl::*, socket::*},
    err::NlError,
    genl::Genlmsghdr,
    socket,
};

fn subscribe_to_mcast() -> Result<(), Box<dyn Error>> {
    let mut s = socket::NlSocketHandle::connect(
        NlFamily::Generic,
        None,
        &[],
    )?;
    let id = s.resolve_nl_mcast_group(
        "my_family_name",
        "my_multicast_group_name",
    )?;
    s.add_mcast_membership(&[id])?;
    for next in s.iter::<NlTypeWrapper, Genlmsghdr<u8, u16>>(true) {
        // Do stuff here with parsed packets...
    
        // like printing a debug representation of them:
        println!("{:?}", next?);
    }

    Ok(())
}
```

I plan to support both of these using a higher level API eventually.

## Contributing

Your contribution will be licensed under neli's [license](LICENSE).
I want to keep this aspect as simple as possible so please read over
the license file prior to contributing to make sure that you feel
comfortable with your contributions being released under the BSD
3-Clause License.

CI is awesome - please add tests for new features wherever possible.
I may request this prior to merge if I see it is possible and missing.

Please document new features not just at a lower level but also with
`//!` comments at the module for high level documentation and
overview of the feature.

Before submitting PRs, take a look at the module's documentation that
you are changing. I am currently in the process of adding a "Design
decision" section to each module. If you are wondering why I did
something the way I did, it should be there. That way, if you have a
better way to do it, please let me know! I'm always happy to learn.
My hope is that this will also clarify some questions beforehand
about why I did things the way I did and will make your life as a
contributer easier.

### PR target branch

Steps for a PR:
* For bug fixes and improvements that are not breaking changes,
please target `master`
* For breaking changes, please target the branch for the next version
release - this will look like `v[NEXT_VERSION]-dev`
* Please include a brief description of your change in the CHANGELOG
file
* Once a PR has been reviewed and approved, please rebase onto the
target branch
  * For those less familiar with git, it should look something like
this
    * `git rebase [TARGET_BRANCH] [YOUR_BRANCH]`
    * _This is a destructive operation so make sure you check carefully before doing this_: `git push -f origin [YOUR_BRANCH]`
