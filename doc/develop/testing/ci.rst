.. _ttzp_ci_tests:

Continuous Integration Tests
============================

GitHub Actions builds and tests this repository. This page is the map of those
workflows: what each one runs, which boards it covers, and what causes it to
start. :ref:`Hardware Testing <ttzp_testing>` explains how to run the hardware
suites on a developer machine.

Board names come from :file:`.github/ci_boards.json` and
:file:`.github/boards.json`. Jobs that build a matrix read those files, so
adding a board there changes CI without editing the workflow.

Pull request checks
-------------------

The workflows in this section run on pull requests. Most of them also run for
the merge queue and for manual ``workflow_dispatch`` runs. The ``on:`` block
in each workflow file is the full trigger list.

Build
*****

:file:`.github/workflows/build-fw.yml` (workflow name ``Build``).

Runs on pull requests and the merge queue targeting ``main``, ``v*-branch``,
and ``collab*``, and on pushes to ``v*-branch`` and ``collab*``. HW Soak and
Metal call this workflow to produce the firmware bundle they flash.

The workflow does the following:

* Sysbuild of ``app/smc`` for every revision in :file:`.github/boards.json`:
  p100a, p150a, p150b, p150c, p300a, p300b, p300c, galaxy, galaxy_2,
  galaxy_revc, galaxy_cf, and orion_slt. Compiler warnings are errors. When a
  signing key is present, images are signed and checked with ``imgtool``.
  ``galaxy_2`` is built and uploaded unsigned.
* Recovery image for ``tt_blackhole@p100a/tt_blackhole/smc``, using
  ``sysbuild/recovery.conf``.
* Assembly-test DMC firmware, with ``CONFIG_TT_ASSEMBLY_TEST=y``, for the p150
  and p300 families (``tt_blackhole@p150a`` and ``tt_blackhole@p300a`` DMC
  boards).
* Preflash images for p150a and p300a, via ``scripts/build-preflash.sh``.
* A combined firmware bundle packed from the per-board bundles.
* The Blackhole flash loader module
  (``scripts/tooling/blackhole_recovery/data/bh_flm/build-flm.sh``).
* Build-only Twister runs for SMC scenarios tagged ``build-ci``, on p100a,
  p150a, and p300a. ``scripts/discover_build_ci_tests.py`` selects the
  scenarios from ``app/smc/sample.yaml``. Those tags are on ``app.tracing``
  and ``app.tt_shell``.
* Build-only Twister runs for DMC scenarios tagged ``build-ci``, on
  ``native_sim``. The same script reads ``app/dmc/sample.yaml``. That tag is
  on ``sample.app.native_sim_build``.
* Twister build of tests tagged ``smoke`` for ``tt_mimir/tt_mimir/smc``,
  including Zephyr tests and samples plus this repository's tests and samples,
  with the overlays in ``test-conf``.
* Build of ``app/dm_test_app`` for ``tt_grendel_mk_bu_dmc``, with compiler
  warnings as errors.

Unit
****

:file:`.github/workflows/run-unit-tests.yml` (workflow name ``Unit``).

Runs on pull requests and the merge queue targeting ``main`` and ``v*-branch``,
and on pushes to ``v*-branch``.

* ``unit-test`` builds, then runs, Twister on the ``unit_testing`` and
  ``native_sim`` platforms. The test roots are ``tests/`` and ``samples/``.
* ``tt-sim-test`` builds and runs Twister on ``native_sim/native/64`` for the
  ``tt_sim`` tag. The test root is the Zephyr tree, with
  ``test-conf`` as an alternate config root.

HW Smoke
********

:file:`.github/workflows/hardware-smoke.yml` (workflow name ``HW Smoke``).

Runs on pull requests and the merge queue targeting ``main``, ``v*-branch``,
and ``collab*``, and on pushes to ``v*-branch`` and ``collab*``. Pull request
``labeled`` events are included so the Wormhole job can start when the
``wh-tests`` label is added.

Blackhole smoke and end-to-end jobs run on every Blackhole entry in
:file:`.github/ci_boards.json` with ``supports_smoke`` set. That is p100a,
p150a, and p300a. Each job uses the CI container and the board's ``kmd_build``.

* DMC smoke: ``scripts/ci/run-smoke.sh -t dmc``. Twister flashes the DMC and
  runs tests tagged ``smoke`` on the DMC board, using ``scripts/dmc_rtt.py``
  as the serial PTY. Roots are Zephyr tests and samples plus this repository's
  tests and samples, with ``test-conf`` overlays.
* SMC smoke: ``scripts/ci/run-smoke.sh -t smc``. The script first flashes the
  DMC end-to-end image (tag ``e2e`` under ``app/``) so the flash mux points at
  the SMC, then runs tests tagged ``smoke`` on the SMC board through
  ``scripts/smc_console.py``.
* Blackhole end-to-end: ``scripts/ci/run-e2e.sh``. Twister runs the
  ``e2e-flash`` tag, which is ``app.e2e-smoke-flash`` in ``app/smc/sample.yaml``
  and executes ``app/smc/pytest/e2e_smoke.py``. The job builds ``bh-mod`` from
  the luwen repository for the ccfgovr cases. A signing key from the
  repository secret is used when one is configured; otherwise the script
  generates a temporary key.

Wormhole end-to-end runs on Wormhole entries with ``supports_smoke`` set
(n150). The job runs when the pull request has the ``wh-tests`` label, or when
the pull request changes ``zephyr/blobs/fw_pack-wormhole.tar.gz``. It runs
``pytest app/wh/pytest/e2e_smoke.py`` and skips ``test_upgrade_from_19_0``.

Static checks
*************

Compliance
~~~~~~~~~~

:file:`.github/workflows/compliance.yml` (workflow name ``Compliance Checks``).

Runs on every pull request (including edits to the description) and on manual
dispatch. The job rebases the series onto the target branch, rejects merge
commits, and runs ``scripts/check_sysfw_compliance.py``. That script wraps
Zephyr's compliance checks. ``BinaryFiles``, ``KconfigBasicNoModules``,
``SysbuildKconfigBasic``, and ``LicenseAndCopyrightCheck`` are excluded.
``Checkpatch`` and ``CheckpatchFile`` keep Zephyr's severity.
``ClangFormat`` failures are always errors. Other checks promote warnings to
errors. The job is ``continue-on-error``, and it uploads ``compliance.xml``.

Copyright
~~~~~~~~~

:file:`.github/workflows/copyright-check.yml` runs
``scripts/check-copyright.py``. It runs on pull requests and the merge queue
targeting ``main``, ``v*-branch``, and ``collab*``, and on pushes to
``v*-branch`` and ``collab*``.

License scan
~~~~~~~~~~~~

:file:`.github/workflows/license-check.yml` (workflow name ``Scancode``) scans
the ``scan/`` directory with ``zephyrproject-rtos/action_scancode``. A
non-empty ``artifacts/report.txt`` fails the job. Triggers match the copyright
check.

Blob checksums
~~~~~~~~~~~~~~

:file:`.github/workflows/verify-blob.yml` runs ``scripts/verify_blob.py`` on
pull requests to ``main``, ``v*-branch``, and ``collab*``.

Native tooling
~~~~~~~~~~~~~~

:file:`.github/workflows/build-native.yml` (workflow name ``Build tt-native``)
runs ``make`` in ``scripts/tooling`` and uploads ``tt-fw-terminal`` and
``tt-tracing``. It runs on pull requests and the merge queue targeting
``main`` and ``v*-branch``, and on pushes to ``v*-branch``.

Scheduled and label-gated tests
-------------------------------

These workflows stay idle on an ordinary pull request. They run on their
schedule, when someone applies the named label, or on manual dispatch.

HW Soak
*******

:file:`.github/workflows/hardware-long.yml` (workflow name ``HW Soak``).

Runs daily at 05:00 UTC, when a pull request receives the ``long-tests``
label, or on manual dispatch. It builds a firmware bundle by calling the Build
workflow, then runs three hardware jobs.

* End-to-end stress on every Blackhole board in :file:`.github/ci_boards.json`:
  p100a, p150a, p300a, quietbox2, loudbox, and bh-galaxy-revc.
  ``scripts/ci/run-stress.sh`` runs Twister with the ``e2e-stress`` tag. That
  tag covers ``app.stress-test`` (``app/smc/pytest/e2e_stress.py``) and
  ``app.e2e-prod-assert`` (``app/smc/pytest/e2e_smoke.py`` built with
  ``CONFIG_ASSERT=y``).
* Tensix column check on the p150a runner. The job installs
  ``tt-update-tensix-disable-count``, patches the combined bundle, flashes it
  with ``tt-flash``, and reads telemetry. The check passes when the
  tensix-enabled column count is 140.
* Manufacturing bring-up on p100a, p150a, and p300a.
  ``scripts/ci/run-manufacturing-test.sh`` consumes the preflash image,
  assembly MCUBoot image, assembly firmware, combined bundle, and Blackhole
  flash loader produced by the Build workflow. See
  :ref:`Manufacturing Test <ttzp_manufacturing>` for the local equivalent.
  That flow erases SPI on the device under test.

Metal
*****

:file:`.github/workflows/metal.yml` (workflow name ``Metal``).

Runs daily at 06:00 UTC, when a pull request receives the ``metal-tests``
label, or on manual dispatch. It builds a firmware bundle by calling the Build
workflow. Both hardware jobs run on every board in
:file:`.github/ci_boards.json` that has a ``metal-image``: p100a, p150a,
p300a, quietbox2, loudbox, and bh-galaxy-revc. Each job flashes the combined
bundle with ``tt-flash`` and resets the card with ``tt-smi``.

* Upstream tt-metal tests. The job runs
  ``dockerfile/upstream_test_images/run_upstream_tests_vanilla.sh`` inside the
  board's ``metal-image`` container, passing that board's ``metal-target``
  (``blackhole``, ``blackhole_p300``, ``blackhole_qb_ge``,
  ``blackhole_loudbox``, or ``blackhole_glx``).
* Deploy tests, in the same container. DRAM coverage comes from
  ``tests/tt_metal/tt_metal/deployment/dram/test_runner.py``. PCIe coverage is
  a host-read and a host-write ``mem_bench`` run. Ethernet coverage
  (``tests/tt_metal/tt_metal/deployment/eth/test_runner.py``) runs on boards
  with ``metal-deploy-run-eth`` set, which is bh-galaxy-revc.

Publishing and maintenance
--------------------------

These workflows package artifacts or maintain the repository.

* ``Documentation`` (:file:`.github/workflows/docs.yml`) builds this Sphinx
  site on pull requests and publishes it from ``main``. Per-board CI history
  pages are generated on pushes to ``main`` and on pull requests labeled
  ``ci-docs``.
* ``Create a Release`` (:file:`.github/workflows/release.yml`) packages a
  release when a ``v*`` or ``v*rc*`` tag is pushed.
* ``Create and publish Docker images``
  (:file:`.github/workflows/build-containers.yml`) publishes container images
  on those same tags, or on manual dispatch.
* ``Produce Data`` (:file:`.github/workflows/produce-data.yml`) writes pipeline
  JSON after HW Smoke, HW Soak, Metal, Unit, or Create a Release finishes.
* ``Publish Update Tensix Disable Count Package``
  (:file:`.github/workflows/publish-tensix-disable.yml`) publishes the
  ``tt-update-tensix-disable-count`` package on manual dispatch. Before
  publishing, it patches a firmware bundle with the package, flashes that
  bundle on p150a, and expects 140 tensix-enabled columns.
* ``Pull Request Assigner``, ``Auto Rebase Pull Requests``,
  ``Close stale pull requests/issues``, and ``Nightly Branch Cleanup`` assign
  reviewers, rebase labeled pull requests, close stale items, and delete
  obsolete branches.
