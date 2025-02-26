email-validator: Validate Email Addresses
=========================================

A robust email address syntax and deliverability validation library for
Python by [Joshua Tauberer](https://joshdata.me).

This library validates that a string is of the form `name@example.com`. This is
the sort of validation you would want for an email-based login form on 
a website.

Key features:

* Checks that an email address has the correct syntax --- good for
  login forms or other uses related to identifying users.
* Gives friendly error messages when validation fails (appropriate to show
  to end users).
* (optionally) Checks deliverability: Does the domain name resolve? And you can override the default DNS resolver.
* Supports internationalized domain names and (optionally)
  internationalized local parts, but blocks unsafe characters.
* Normalizes email addresses (super important for internationalized
  addresses! see below).

The library is NOT for validation of the To: line in an email message
(e.g. `My Name <my@address.com>`), which
[flanker](https://github.com/mailgun/flanker) is more appropriate for.
And this library does NOT permit obsolete forms of email addresses, so
if you need strict validation against the email specs exactly, use
[pyIsEmail](https://github.com/michaelherold/pyIsEmail).

This library is tested with Python 3.6+ but should work in earlier versions:

[![Build Status](https://app.travis-ci.com/JoshData/python-email-validator.svg?branch=main)](https://app.travis-ci.com/JoshData/python-email-validator)

View the [CHANGELOG / Release Notes](CHANGELOG.md) for the version history of changes in the library. Occasionally this README is ahead of the latest published package --- see the CHANGELOG for details.

---

Installation
------------

This package [is on PyPI](https://pypi.org/project/email-validator/), so:

```sh
pip install email-validator
```

`pip3` also works.

Quick Start
-----------

If you're validating a user's email address before creating a user
account in your application, you might do this:

```python
from email_validator import validate_email, EmailNotValidError

email = "my+address@mydomain.tld"
is_new_account = True # False for login pages

try:
  # Check that the email address is valid.
  validation = validate_email(email, check_deliverability=is_new_account)

  # Take the normalized form of the email address
  # for all logic beyond this point (especially
  # before going to a database query where equality
  # may not take into account Unicode normalization).  
  email = validation.email
except EmailNotValidError as e:
  # Email is not valid.
  # The exception message is human-readable.
  print(str(e))
```

This validates the address and gives you its normalized form. You should
**put the normalized form in your database** and always normalize before
checking if an address is in your database. When using this in a login form,
set `check_deliverability` to `False` to avoid unnecessary DNS queries.

Usage
-----

### Overview

The module provides a function `validate_email(email_address)` which
takes an email address (either a `str` or `bytes`, but only non-internationalized
addresses are allowed when passing a `bytes`) and:

- Raises a `EmailNotValidError` with a helpful, human-readable error
  message explaining why the email address is not valid, or
- Returns an object with a normalized form of the email address (which
  you should use!) and other information about it.

When an email address is not valid, `validate_email` raises either an
`EmailSyntaxError` if the form of the address is invalid or an
`EmailUndeliverableError` if the domain name fails DNS checks. Both
exception classes are subclasses of `EmailNotValidError`, which in turn
is a subclass of `ValueError`.

But when an email address is valid, an object is returned containing
a normalized form of the email address (which you should use!) and
other information.

The validator doesn't permit obsoleted forms of email addresses that no
one uses anymore even though they are still valid and deliverable, since
they will probably give you grief if you're using email for login. (See
later in the document about that.)

The validator checks that the domain name in the email address has a
DNS MX record (except a NULL MX record) indicating that it can receive
email and that it does not have a reject-all SPF record (`v=spf1 -all`)
which would indicate that it cannot send email.
There is nothing to be gained by trying to actually contact an SMTP
server, so that's not done here. For privacy, security, and practicality
reasons servers are good at not giving away whether an address is
deliverable or not: email addresses that appear to accept mail at first
can bounce mail after a delay, and bounced mail may indicate a temporary
failure of a good email address (sometimes an intentional failure, like
greylisting).

### Options

The `validate_email` function also accepts the following keyword arguments
(defaults are as shown below):

`allow_smtputf8=True`: Set to `False` to prohibit internationalized addresses that would
    require the
    [SMTPUTF8](https://tools.ietf.org/html/rfc6531) extension. You can also set `email_validator.ALLOW_SMTPUTF8` to `False` to turn it off for all calls by default.

`check_deliverability=True`: If true, DNS queries check that a non-null MX (or A/AAAA record as an MX fallback) is present for the domain-part of the email address and that a reject-all SPF record is not present. Set to `False` to skip these DNS checks. DNS is slow and sometimes unavailable, so consider whether these checks are useful for your use case. It is recommended to pass `False` when performing validation for login pages (but not account creation pages) since re-validation of the domain by querying DNS at every login is probably undesirable. You can also set `email_validator.CHECK_DELIVERABILITY` to `False` to turn this off for all calls by default.

`allow_empty_local=False`: Set to `True` to allow an empty local part (i.e.
    `@example.com`), e.g. for validating Postfix aliases.
    
`dns_resolver=None`: Pass an instance of [dns.resolver.Resolver](https://dnspython.readthedocs.io/en/latest/resolver-class.html) to control the DNS resolver including setting a timeout and [a cache](https://dnspython.readthedocs.io/en/latest/resolver-caching.html). The `caching_resolver` function shown above is a helper function to construct a dns.resolver.Resolver with a [LRUCache](https://dnspython.readthedocs.io/en/latest/resolver-caching.html#dns.resolver.LRUCache). Reuse the same resolver instance across calls to `validate_email` to make use of the cache.

`test_environment=False`: DNS-based deliverability checks are disabled and  `test` and `subdomain.test` domain names are permitted (see below). You can also set `email_validator.TEST_ENVIRONMENT` to `True` to turn it on for all calls by default.

### DNS timeout and cache

When validating many email addresses or to control the timeout (the default is 15 seconds), create a caching [dns.resolver.Resolver](https://dnspython.readthedocs.io/en/latest/resolver-class.html) to reuse in each call. The `caching_resolver` function returns one easily for you:

```python
from email_validator import validate_email, caching_resolver

resolver = caching_resolver(timeout=10)

while True:
  email = validate_email(email, dns_resolver=resolver).email
```

### Test addresses

This library rejects email addresess that use the [Special Use Domain Names](https://www.iana.org/assignments/special-use-domain-names/special-use-domain-names.xhtml) `invalid`, `localhost`, `test`, and some others by raising `EmailSyntaxError`. This is to protect your system from abuse: You probably don't want a user to be able to cause an email to be sent to `localhost`. However, in your non-production test environments you may want to use `@test` or `@myname.test` email addresses. There are three ways you can allow this:

1. Add `test_environment=True` to the call to `validate_email` (see above).
2. Set `email_validator.TEST_ENVIRONMENT` to `True`.
3. Remove the special-use domain name that you want to use from `email_validator.SPECIAL_USE_DOMAIN_NAMES`:

```python
import email_validator
email_validator.SPECIAL_USE_DOMAIN_NAMES.remove("test")
```

It is tempting to use `@example.com/net/org` in tests. These domains are reserved to IANA for use in documentation so there is no risk of accidentally emailing someone at those domains. But beware that this library will reject these domain names if DNS-based deliverability checks are not disabled because these domains do not resolve to domains that accept email. In tests, consider using your own domain name or `@test` or `@myname.test` instead.

Internationalized email addresses
---------------------------------

The email protocol SMTP and the domain name system DNS have historically
only allowed English (ASCII) characters in email addresses and domain names,
respectively. Each has adapted to internationalization in a separate
way, creating two separate aspects to email address
internationalization.

### Internationalized domain names (IDN)

The first is [internationalized domain names (RFC
5891)](https://tools.ietf.org/html/rfc5891), a.k.a IDNA 2008. The DNS
system has not been updated with Unicode support. Instead, internationalized
domain names are converted into a special IDNA ASCII "[Punycode](https://www.rfc-editor.org/rfc/rfc3492.txt)"
form starting with `xn--`. When an email address has non-ASCII
characters in its domain part, the domain part is replaced with its IDNA
ASCII equivalent form in the process of mail transmission. Your mail
submission library probably does this for you transparently. Note that
most web browsers are currently in transition between IDNA 2003 (RFC
3490) and IDNA 2008 (RFC 5891) and [compliance around the web is not
very
good](http://archives.miloush.net/michkap/archive/2012/02/27/10273315.html)
in any case, so be aware that edge cases are handled differently by
different applications and libraries. This library conforms to IDNA 2008
using the [idna](https://github.com/kjd/idna) module by Kim Davies.

### Internationalized local parts

The second sort of internationalization is internationalization in the
*local* part of the address (before the @-sign). In non-internationalized
email addresses, only English letters, numbers, and some punctuation
(`._!#$%&'^``*+-=~/?{|}`) are allowed. In internationalized email address
local parts, a wider range of Unicode characters are allowed.

A surprisingly large number of Unicode characters are not safe to display,
especially when the email address is concatenated with other text, so this
library tries to protect you by not permitting resvered, non-, private use,
formatting (which can be used to alter the display order of characters),
whitespace, and control characters, and combining characters
as the first character (so that they cannot combine with something outside
of the email address string). See https://qntm.org/safe and https://trojansource.codes/
for relevant prior work. (Other than whitespace, these are checks that
you should be applying to nearly all user inputs in a security-sensitive
context.)

These character checks are performed after Unicode normalization (see below),
so you are only fully protected if you replace all user-provided email addresses
with the normalized email address string returned by this library. This does not
guard against the well known problem that many Unicode characters look alike
(or are identical), which can be used to fool humans reading displayed text.

Email addresses with these non-ASCII characters require that your mail
submission library and the mail servers along the route to the destination,
including your own outbound mail server, all support the
[SMTPUTF8 (RFC 6531)](https://tools.ietf.org/html/rfc6531) extension.
Support for SMTPUTF8 varies. See the `allow_smtputf8` parameter.

### If you know ahead of time that SMTPUTF8 is not supported by your mail submission stack

By default all internationalized forms are accepted by the validator.
But if you know ahead of time that SMTPUTF8 is not supported by your
mail submission stack, then you must filter out addresses that require
SMTPUTF8 using the `allow_smtputf8=False` keyword argument (see above).
This will cause the validation function to raise a `EmailSyntaxError` if
delivery would require SMTPUTF8. That's just in those cases where
non-ASCII characters appear before the @-sign. If you do not set
`allow_smtputf8=False`, you can also check the value of the `smtputf8`
field in the returned object.

If your mail submission library doesn't support Unicode at all --- even
in the domain part of the address --- then immediately prior to mail
submission you must replace the email address with its ASCII-ized form.
This library gives you back the ASCII-ized form in the `ascii_email`
field in the returned object, which you can get like this:

```python
valid = validate_email(email, allow_smtputf8=False)
email = valid.ascii_email
```

The local part is left alone (if it has internationalized characters
`allow_smtputf8=False` will force validation to fail) and the domain
part is converted to [IDNA ASCII](https://tools.ietf.org/html/rfc5891).
(You probably should not do this at account creation time so you don't
change the user's login information without telling them.)

### Support for Python 2.7

The last version of this library supporting Python 2.x is version 1.2.1.

When using Python 2.x, it is required that it was built with
UCS-4 support (see
[here](https://stackoverflow.com/questions/29109944/python-returns-length-of-2-for-single-unicode-character-string)).
Without UCS-4 support, unicode characters outside of the BMP (Basic
Multilingual Plane) will not validate correctly.

Normalization
-------------

The use of Unicode in email addresses introduced a normalization
problem. Different Unicode strings can look identical and have the same
semantic meaning to the user. The `email` field returned on successful
validation provides the correctly normalized form of the given email
address:

```python
valid = validate_email("me@Ｄｏｍａｉｎ.com")
email = valid.ascii_email
print(email)
# prints: me@domain.com
```

Because an end-user might type their email address in different (but
equivalent) un-normalized forms at different times, you ought to
replace what they enter with the normalized form immediately prior to
going into your database (during account creation), querying your database
(during login), or sending outbound mail. Normalization may also change
the length of an email address, and this may affect whether it is valid
and acceptable by your SMTP provider.

The normalizations include lowercasing the domain part of the email
address (domain names are case-insensitive), [Unicode "NFC"
normalization](https://en.wikipedia.org/wiki/Unicode_equivalence) of the
whole address (which turns characters plus [combining
characters](https://en.wikipedia.org/wiki/Combining_character) into
precomposed characters where possible, replacement of [fullwidth and
halfwidth
characters](https://en.wikipedia.org/wiki/Halfwidth_and_fullwidth_forms)
in the domain part, possibly other
[UTS46](http://unicode.org/reports/tr46) mappings on the domain part,
and conversion from Punycode to Unicode characters.

(See [RFC 6532 (internationalized email) section
3.1](https://tools.ietf.org/html/rfc6532#section-3.1) and [RFC 5895
(IDNA 2008) section 2](http://www.ietf.org/rfc/rfc5895.txt).)

Examples
--------

For the email address `test@joshdata.me`, the returned object is:

```python
ValidatedEmail(
  email='test@joshdata.me',
  local_part='test',
  domain='joshdata.me',
  ascii_email='test@joshdata.me',
  ascii_local_part='test',
  ascii_domain='joshdata.me',
  smtputf8=False)
```

For the fictitious address `example@ツ.life`, which has an
internationalized domain but ASCII local part, the returned object is:

```python
ValidatedEmail(
  email='example@ツ.life',
  local_part='example',
  domain='ツ.life',
  ascii_email='example@xn--bdk.life',
  ascii_local_part='example',
  ascii_domain='xn--bdk.life',
  smtputf8=False)

```

Note that `smtputf8` is `False` even though the domain part is
internationalized because
[SMTPUTF8](https://tools.ietf.org/html/rfc6531) is only needed if the
local part of the address is internationalized (the domain part can be
converted to IDNA ASCII Punycode). Also note that the `email` and `domain`
fields provide a normalized form of the email address and domain name
(casefolding and Unicode normalization as required by IDNA 2008).

Calling `validate_email` with the ASCII form of the above email address,
`example@xn--bdk.life`, returns the exact same information (i.e., the
`email` field always will contain Unicode characters, not Punycode).

For the fictitious address `ツ-test@joshdata.me`, which has an
internationalized local part, the returned object is:

```python
ValidatedEmail(
  email='ツ-test@joshdata.me',
  local_part='ツ-test',
  domain='joshdata.me',
  ascii_email=None,
  ascii_local_part=None,
  ascii_domain='joshdata.me',
  smtputf8=True)
```

Now `smtputf8` is `True` and `ascii_email` is `None` because the local
part of the address is internationalized. The `local_part` and `email` fields
return the normalized form of the address: certain Unicode characters
(such as angstrom and ohm) may be replaced by other equivalent code
points (a-with-ring and omega).

Return value
------------

When an email address passes validation, the fields in the returned object
are:

| Field | Value |
| -----:|-------|
| `email` | The normalized form of the email address that you should put in your database. This merely combines the `local_part` and `domain` fields (see below). |
| `ascii_email` | If set, an ASCII-only form of the email address by replacing the domain part with [IDNA](https://tools.ietf.org/html/rfc5891) [Punycode](https://www.rfc-editor.org/rfc/rfc3492.txt). This field will be present when an ASCII-only form of the email address exists (including if the email address is already ASCII). If the local part of the email address contains internationalized characters, `ascii_email` will be `None`. If set, it merely combines `ascii_local_part` and `ascii_domain`. |
| `local_part` | The local part of the given email address (before the @-sign) with Unicode NFC normalization applied. |
| `ascii_local_part` | If set, the local part, which is composed of ASCII characters only. |
| `domain` | The canonical internationalized Unicode form of the domain part of the email address. If the returned string contains non-ASCII characters, either the [SMTPUTF8](https://tools.ietf.org/html/rfc6531) feature of your mail relay will be required to transmit the message or else the email address's domain part must be converted to IDNA ASCII first: Use `ascii_domain` field instead. |
| `ascii_domain` | The [IDNA](https://tools.ietf.org/html/rfc5891) [Punycode](https://www.rfc-editor.org/rfc/rfc3492.txt)-encoded form of the domain part of the given email address, as it would be transmitted on the wire. |
| `smtputf8` | A boolean indicating that the [SMTPUTF8](https://tools.ietf.org/html/rfc6531) feature of your mail relay will be required to transmit messages to this address because the local part of the address has non-ASCII characters (the local part cannot be IDNA-encoded). If `allow_smtputf8=False` is passed as an argument, this flag will always be false because an exception is raised if it would have been true. |
| `mx` | A list of (priority, domain) tuples of MX records specified in the DNS for the domain (see [RFC 5321 section 5](https://tools.ietf.org/html/rfc5321#section-5)). May be `None` if the deliverability check could not be completed because of a temporary issue like a timeout. |
| `mx_fallback_type` | `None` if an `MX` record is found. If no MX records are actually specified in DNS and instead are inferred, through an obsolete mechanism, from A or AAAA records, the value is the type of DNS record used instead (`A` or `AAAA`). May be `None` if the deliverability check could not be completed because of a temporary issue like a timeout. |
| `spf` | Any SPF record found while checking deliverability. |

Assumptions
-----------

By design, this validator does not pass all email addresses that
strictly conform to the standards. Many email address forms are obsolete
or likely to cause trouble:

* The validator assumes the email address is intended to be
  usable on the public Internet. The domain part
  of the email address must be a resolvable domain name
  (without NULL MX or SPF -all DNS records) if deliverability
  checks are turned on.
  Most [Special Use Domain Names](https://www.iana.org/assignments/special-use-domain-names/special-use-domain-names.xhtml)
  and their subdomains and
  domain names without a `.` are rejected as a syntax error
  (except see the `test_environment` parameter above).
* Obsolete email syntaxes are rejected:
  The "quoted string" form of the local part of the email address (RFC
  5321 4.1.2) is not permitted.
  Quoted forms allow multiple @-signs, space characters, and other
  troublesome conditions. The unsual [(comment) syntax](https://github.com/JoshData/python-email-validator/issues/77)
  is also rejected. The "literal" form for the domain part of an email address (an
  IP address in brackets) is rejected. Other obsolete and deprecated syntaxes are
  rejected. No one uses these forms anymore.

Testing
-------

Tests can be run using

```sh
pip install -r test_requirements.txt 
make test
```

For Project Maintainers
-----------------------

The package is distributed as a universal wheel and as a source package.

To release:

* Update CHANGELOG.md.
* Update the version number in setup.cfg.
* Make a commit with the new version number.
* Follow the steps below to publish source and a universal wheel to pypi and tag the release.
* Make a release at https://github.com/JoshData/python-email-validator/releases/new.

```sh
./release_to_pypi.sh
git tag v$(grep version setup.cfg | sed "s/.*= //")
git push --tags
```

Notes: The wheel is specified as universal in the file `setup.cfg` by the `universal = 1` key in the
`[bdist_wheel]` section.
