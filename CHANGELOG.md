Version 1.3.0 (September 18, 2022)
----------------------------------

* Deliverability checks now check for 'v=spf1 -all' SPF records as a way to reject more bad domains.
* Special use domain names now raise EmailSyntaxError instead of EmailUndeliverableError since they are performed even if check_deliverability is off.
* New module-level attributes are added to override the default values of the keyword arguments and the special-use domains list.
* The keyword arguments of the public methods are now marked as keyword-only, ending support for Python 2.x.
* [pyIsEmail](https://github.com/michaelherold/pyIsEmail)'s test cases are added to the tests.
* Recommend that check_deliverability be set to False for validation on login pages.
* Added an undocumented globally_deliverable option.

Version 1.2.1 (May 1, 2022)
---------------------------

* example.com/net/org are removed from the special-use reserved domain names list so that they do not raise exceptions if check_deliverability is off.
* Improved README.

Verison 1.2.0 (April 24, 2022)
------------------------------

* Reject domains with NULL MX records (when deliverability checks
  are turned on).
* Reject unsafe unicode characters. (Some of these checks you should
  be doing on all of your user inputs already!)
* Reject most special-use reserved domain names with EmailUndeliverableError. A new `test_environment` option is added for using `@*.test` domains.
* Improved safety of exception text by not repeating an unsafe input character in the message.
* Minor fixes in tests.
* Invoking the module as a standalone program now caches DNS queries.
* Improved README.

Version 1.1.3 (June 12, 2021)
-----------------------------

* Allow passing a custom dns_resolver so that a DNS cache and a custom timeout can be set.

Version 1.1.2 (Nov 5, 2020)
---------------------------

* Fix invoking the module as a standalone program.
* Fix deprecation warning in Python 3.8.
* Code improvements.
* Improved README.

Version 1.1.1 (May 19, 2020)
----------------------------

* Fix exception when DNS queries time-out.
* Improved README.

Version 1.1.0 (Spril 30, 2020)
------------------------------

* The main function now returns an object with attributes rather than a dict with keys, but accessing the object in the old way is still supported.
* Added overall email address length checks.
* Minor tweak to regular expressions.
* Improved error messages.
* Added tests.
* Linted source code files; changed README to Markdown.

Version 1.0.5 (Oct 18, 2019)
----------------------------

* Prevent resolving domain names as if they were not fully qualified using a local search domain settings.

Version 1.0.4 (May 2, 2019)
---------------------------

* Added a timeout argument for DNS queries.
* The wheel distribution is now a universal wheel.
* Improved README.

Version 1.0.3 (Sept 12, 2017)
-----------------------------

* Added a wheel distribution for easier installation.

Version 1.0.2 (Dec 30, 2016)
----------------------------

* Fix dnspython package name in Python 3.
* Improved README.

Version 1.0.1 (March 6, 2016)
-----------------------------

* Fixed minor errors.

Version 1.0.0 (Sept 5, 2015)
----------------------------

* Fail domains with a leading period.
* Improved error messages.
* Added tests.

Version 0.5.0 (June 15, 2015)
-----------------------------

* Use IDNA 2008 instead of IDNA 2003 and use the idna package's UTS46 normalization instead of our own.
* Fixes for Python 2.
* Improved error messages.
* Improved README.

Version 0.1.0 (April 21, 2015)
------------------------------

Initial release!
