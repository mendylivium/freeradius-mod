# rlm_securid
## Metadata
<dl>
  <dt>category</dt><dd>authentication</dd>
</dl>

## Summary

Supports authentication against an RSA SecurID ACE instance.

## Description

This module implements SecurID token checking. It should be listed
in the "authenticate" section.

The module configuration is in the "securid" file. You will need
to create a symlink from raddb/mods-enabled/securid to
../mods-available/securid.

There is a configure script, but you will most likely need to pass
--with-securid-dir=<securid_path> argument.

The files it requires are acexport.h and libaceclnt.so which
should be located in the include and lib folders beneath
<secured_path>. EMC/RSA do not distribute these files on their
public website. You will need to request the AuthSDK from your
EMC/RSA representative.

Many people will wonder about the license issues involved in
distributing this module. The short answer is that the source can
be distributed, the binaries cannot be distributed. The
explanation is given below.

This module falls under the GPLv2 license. The primary goal of
this license is largely to ensure that you have access to the
source code, which is included here. A secondary goal of this
license is to ensure that binary distributions can be re-built
from the existing source code. This is done by requiring binary
distributions to offer source code for the GPL'd binary, and to
distribute ALL DEPENDENT LIBRARIES.

The RSA libraries are proprietary to RSA, and cannot be
distributed. Therefore, any library (rlm_securid.a,
rlm_securid.so, etc.) CANNOT be distributed by ANYONE.

The module is still useful, of course. The GPL restriction on
distribution apply only on distribution to third parties. This
means that building the module, and using it within your
organization is allowed under the GPL.
