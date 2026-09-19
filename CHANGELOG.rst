==========================
tudat-resources Change Log
==========================

.. current developments

**Added:**

* add a trimmed set of datasets used by the ``tudatpy`` test suite
  in CI.  This is not required outside of CI.

**Removed:**

* All C++ code; the tudat-resources package has been removed in favour
  of managing the datasets outside of packaging.  The package used to
  behave this way anyway besides providing the header ``resource.h``,
  and implicitly downloading the datasets from GH - both functions are
  now merged into the main ``tudatpy``.

* All files that can be downloaded from the original source, and used
  by tudat as-is has been removed.

v1.1.2
====================



v1.1.1
====================



v1.1.0
====================



v1.0.16
====================

**Added:**

* Added rever support to the repo


v1.0.10
====================


