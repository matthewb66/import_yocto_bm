# Synopsys Import Yocto Build Manifest - import_yocto_bm.py

Please use the main repository for this utility https://github.com/blackducksoftware/import_yocto_bm.
This is a mirror which may not reflect the latest updates.

# INTRODUCTION
This script is provided under an OSS license as an example of how to use the Black Duck APIs to import components from a Yocto project manifest.

It does not represent any extension of licensed functionality of Synopsys software itself and is provided as-is, without warranty or liability.

# OVERVIEW

The `import_yocto_bm.py` script is designed to import a Yocto project build manifest created by Bitbake. It replaces previous scripts (including https://github.com/matthewb66/import_yocto_build_manifest).

It can be used as an alternative to the standard Yocto scan process for Black Duck provided within Synopsys Detect, and is mainly focussed on identifying the recipes within the built image as opposed to all recipes in the build environment, but provides additional capabilities including checking against the Black Duck KB, replacing recipe specifications and propagating locally patched CVEs to the Black Duck project.

# LATEST UPDATES - VERSION 1.11

Version 1.11 fixed a minor issue with AUTOINC component versions.

Versions 1.10 and 1.9 added fixes for the KB recipe lookup, and a new report output showing the list of recipes matched, modified or not matchable (use `--report reort.txt` option) as well as updating the list of KB recipes located in the data file.

Version 1.8 will automatically download a list of all Yocto projects stored in the Black Duck KB and checks the recipes in the local project, remapping recipes to origin layers or revisions to reduce the number of non matched components in the resulting Black Duck project. This functionality can be skipped using the --no_kb_check option, or if the download of the KB data from Github is not possible within the script, the --kb_recipe_file option allows a local copy to be used.

The DEPLOY_DIR and TMP_DIR values extracted from conf files now support the use of environment variables. Other fixes have been applied especially to the processing of the recipeinfo files and extraction of recipe revisions.

# DEFAULT YOCTO SCANNING IN BLACK DUCK

The default Yocto scan process for Black Duck is to determine Bitbake dependencies using Synopsys Detect (see [Synopsys Detect - scanning Yocto](https://synopsys.atlassian.net/wiki/spaces/INTDOCS/pages/631276245/Package+Managers+Supported+by+Detect)).

You should NOT use Black Duck Signature scanning for whole Yocto projects (except for custom recipes which may contain OSS which you want to identify). Signature scanning will not function accurately on Yocto projects because the code is modified locally by change/diff files and no reference copy of the changed code is available, meaning that Signature matching will not be able to determine the correct Yocto modified OSS components). Yocto projects are also very large and Signature scanning can consume large volumes of server resources without providing accurate matching.

In this mode, Synopsys Detect calls Bitbake to report the project layers and recipes, checking against the recipes reported at OpenEmbedded.org (refer to [layers.openembedded.org](http://layers.openembedded.org/) to identify layers and recipes which will be mapped) and creating a Black Duck project containing the mapped components. Recipes in layers not reported at OpenEmbedded.org will not be imported into Black Duck, and you should consider using other Black Duck scan techniques to identify OSS within specific custom recipes referenced in the build. Note also that moving original OSS recipes to new layers will also stop them being reported.

The resulting project will contain mapped recipes/layers including those used to create the image but which may not delivered in the built image. Recipe matching also requires that OSS components are used within standard recipes/revisions and within the original layer. If you copy a recipe to a new or different layer, it will not be matched in the Black Duck project.

To perform a Yocto scan using Synopsys Detect:
- Change to the poky folder of a Yocto project
- Run Synopsys Detect adding the options `--detect.tools=DETECTOR --detect.bitbake.package.names=core-image-sato`  (where `core-image-sato` is the package name).

# WHY THIS SCRIPT

This script operates on a built Yocto project, identifying the build manifest containing __only the recipes which are within the built image__ as opposed to the superset of all recipes used to build the distribution (including build tools etc.) produced by Synopsys Detect. This script also optionally supports extracting a list of locally patched CVEs from Bitbake and marking them as patched in the Black Duck project.

Recipes/layers are checked against OpenEmbedded.org and those which match are added to the created Black Duck project. See the note above about custom recipes and origin recipes moved to new layers which will not be detected.

The script must be executed on a Linux workstation where Yocto has been installed and after a successful Bitbake build.

If invoked in the Yocto build folder (or the build folder is manually specified using the -y option), then it will locate the license-manifest file automatically in the tmp/deploy hierarchy.

If the Bitbake build was performed with the Yocto `cve_check` class configured, then the script will also optionally locate the CVE log exported by CVE check, extract patched CVEs and set the remediation status of matching CVEs in the Black Duck project.

It requires access to the Black Duck server via the API (see Prerequisites below) unless the -o option is used to create the output scan file for manual upload.

# OVERALL SCAN PROCESS USING IMPORT_YOCTO_BM

Using this script will allow standard OSS recipes to be imported to the Black Duck project. It will also identify where standard OSS recipes have been moved to new or different layers, and match close revisions, but the version must be the same.

The proposed process to scan a Yocto project using this script is:

1. Use the import_yocto_bm to identify original OSS recipes within the project and create a Black Duck project version.

1. Add the `--report rep.txt` option to export a report of the matched recipes. This will include reports categorised as follows:

- OK = recipe matched the Black Duck KB and will be included in the Black Duck project
- REPLACED = recipe has been moved to a new layer and the script has referenced the original layer in the KB - will be included in the Black Duck project
- REPLACED_NOREVISION = the script has replaced the revision to match the KB - will be included in project
- REPLACED_NOLAYER+REVISION = recipe has been moved to a new layer with a new revision and the script has referenced the original layer and revision in the KB - will be included in the Black Duck project
- NOTREPLACED_NOVERSION = the layer and recipe exist in the KB but the version does not - will not be included in the Black Duck project
- NOTREPLACED_NOLAYER+VERSION = the recipe exists in the KB but the layer and version do not - will not be included in the Black Duck project
- MISSING = recipe does not exist in the KB and will not be included in the Black Duck project

3. For the NOTREPLACED_NOVERSION and NOTREPLACED_NOLAYER+VERSION recipes, you could consider using the `--replacefile repfile` option to map to layers/recipes and version/revisions which exist in the KB, rerunning the script to import them. See the section REPLACING LAYER AND RECIPE NAMES below.

4. For the MISSING recipes, these are almost certainly custom recipes either containing new OSS or a mix of custom application code and OSS. Identify the layers which contain mainly custom recipes, and use standard Black Duck signature scanning (and optionally snippet scanning) to search for modified OSS within these sub-folders only. __Do not run a signature or snippet scan on the entire Yocto project__ - make sure you only use this for the custom layers and recipes. You can optionally combine the dependency and signature scans in the same Black Duck project.

# SUPPORTED YOCTO PROJECTS

This script is designed to support Yocto versions 2.0 up to 3.3 for building projects.

OSS components from OpenEmbedded recipes maintained at layers.openbedded.org should be detected by the scan. Additional OSS components managed by custom recipes will not be detected.

# PREREQUISITES

1. Must be run on Linux

1. Python 3 must be installed.

1. Black Duck API package must be installed using `pip3 install blackduck`.

1. An API key for the Black Duck server must be configured in the `.restconfig.json` file in the script invocation folder.

1. A supported Yocto environment (version 2.0 to 3.1) must be loaded to the current shell (see [Preconfiguration](#PRECONFIGURATION) section below).

1. The Yocto project must have been pre-built.

1. For patched CVE remediation in the Black Duck project, you will need to add the `cve_check` bbclass to the Yocto build configuration to generate the CVE check log output. Add the following line to the `build/conf/local.conf` file:

       INHERIT += "cve-check"

Then use the Yocto build command (e.g. `bitbake core-image-sato` which will incrementally build without needing to rerun the full build, but will add the CVE check action to generate the log files.

# INSTALLATION

To download the script, change to a chosen location and use Git to download a copy of the project:

    git clone https://github.com/matthewb66/import_yocto_bm
    export YOCTO_BM_LOC=`pwd`/import_yocto_bm

# STANDARD USAGE

The `import_yocto_bm.py` usage is shown below:

	usage: import_yocto_bm [-h] [-p PROJECT] [-v VERSION] [-y YOCTO_FOLDER]
				[-t TARGET] [-o OUTPUT_JSON] [-m MANIFEST]
				[-b BUILDCONF] [-l LOCALCONF] [--arch ARCH]
				[-r repfile] [--cve_check_only] [--no_cve_check]
				[--cve_check_file CVE_CHECK_FILE] [--report rep.txt]

	Import Yocto build manifest to BD project version

	optional arguments:
	  -h, --help            show this help message and exit
	  -p PROJECT, --project PROJECT
				Black Duck project to create (REQUIRED)
	  -v VERSION, --version VERSION
				Black Duck project version to create (REQUIRED)
	  -y YOCTO_FOLDER, --yocto_build_folder YOCTO_FOLDER
	  			Yocto build folder (required if CVE check required or
				manifest file not specified)
	  -o OUTPUT_JSON, --output_json OUTPUT_JSON
				Output JSON bom file for manual import to Black Duck
				(instead of uploading the scan automatically)
	  -t TARGET, --target TARGET
				Yocto target (default core-poky-sato)
	  -m MANIFEST, --manifest MANIFEST
				Input build manifest file (if not specified will be
				determined from conf files) - must be the 
				license.manifest file (not build.manifest)
	  -b BUILDCONF, --buildconf BUILDCONF
				Build config file (if not specified 
				poky/meta/conf/bitbake.conf will be used)
	  -l LOCALCONF, --localconf LOCALCONF
				Local config file (if not specified 
				poky/build/conf/local.conf will be used)
	  -r REPFILE, --replacefile REPFILE
	  			Replace file used to replace layer and recipe names
	  --arch ARCH           Architecture (if not specified then will be determined
				from conf files)
	  --cve_check_only      Only check for patched CVEs from cve_check and update
				existing project
	  --no_cve_check        Skip check for and update of patched CVEs
	  --cve_check_file CVE_CHECK_FILE
				CVE check output file (if not specified will be
				determined from conf files)
	  --no_kb_check         Do not check recipes against KB
	  --kb_recipe_file KB_RECIPE_FILE
                        	KB recipe file local copy
	  --report rep.txt	If KB check is performed, produce a list of matched. modified and unmatched recipes.


The script will use the invocation folder as the Yocto build folder (e.g. yocto_zeus/poky/build) by default (if there is a `build` sub-folder then it will be used instead). The `--yocto_folder` option can be used to specify the Yocto build folder as opposed to the invocation folder.

The `--project` and `--version` options are required to define the Black Duck project and version names for inclusion in the json output file (and update CVE patch status).

The `--output_json` option can be used to specify an output file for the project scan. If specified, then the scan will not be uploaded automatically and CVE patch checking will be skipped.

The `--replacefile` option can be used to specify a layer/recipe/version replacement file (see REPLACING LAYER AND RECIPE NAMES below).

The Yocto target and architecture values are required to locate the manifest and cve\_check log files and will be extracted from the Bitbake config files automatically, but the `--target` and `--arch` options can be used to specify these manually.

The most recent Bitbake output manifest file (located in the `build/tmp/deploy/licenses/<image>-<target>-<datetime>/license.manifest` file) will be located automatically. Use the `--manifest` option to specify the manifest file manually.

The most recent cve\_check log file `build/tmp/deploy/images/<arch>/<image>-<target>-<datetime>.rootfs.cve` will be located automatically if it exists. Use the `--cve_check_file` option to specify the cve\_check log file location manually (for example to use an older copy).

Use the `--cve_check_only` option to skip the scanning of the project and creation of a project, only looking for a CVE check output log file to identify and patching matched CVEs within an existing Black Duck project (which must have been created previously).

Use the `--no_cve_check` option to skip the patched CVE identification and update of CVE status in the Black Duck project. 

# PRECONFIGURATION

You will need to run the following commands (change the location as required):

    cd /home/users/myuser/yocto_zeus/poky
    source oe-init-build-env

The `oe-init-build-env` script will change directory into the Yocto build sub-folder.

A `.restconfig.json` file must be created within the build folder: example `.restconfig.json` file:

    {
      "baseurl": "https://SERVER_URL",
      "api_token": "TOKEN",
      "insecure": true,
      "debug": false
    }

Where `SERVER_URL` is the Black Duck server URL and `TOKEN` is the Black Duck API token.

# REPLACING LAYER AND RECIPE NAMES

Layers and recipes extracted from the project are combined by Black Duck and used to lookup original OSS components at https://layers.openembedded.org. If OSS components are moved from original layers to a new (custom) or different layer which is not shown at https://layers.openembedded.org then they will not be mapped in the resulting Black Duck project.

To reference the original component, you can use the `--replacefile REPFILE` option to map OSS components back to original layers or original names.

The replacefile option can also be used to remap new OSS component versions (not listed at https://layers.openembedded.org) to previous versions which are listed.

Example REPFILE content is shown below:

	LAYER meta-customlayer meta-oe
	RECIPE alsa-lib2 alsa-lib
	RECIPE alsa-lib/1.2.1.2-r5 alsa-lib/1.2.1.2-r0
	RECIPE meta-customlayer/alsa-lib meta/alsa-lib
	RECIPE meta-customlayer/alsa-lib/1.2.1.2-r0 meta/alsa-lib/1.2.1.2-r0

The `LAYER` line will remap all recipes from the `meta-customlayer` layer to `meta-oe`.
The 1st `RECIPE` line will remap all recipe versions called `alsa-lib2` to `alsa-lib` across all layers .
The 2nd `RECIPE` line will remap the recipe and version `alsa-lib/1.2.1.2-r5` across all layers to `alsa-lib/1.2.1.2-r0`.
The 3rd `RECIPE` line will remap all versions of the recipe `alsa-lib` in the `meta-customlayer` layer to `meta/alsa-lib`.
The 4th `RECIPE` line will remap recipe and version `alsa-lib/1.2.1.2-r5` in the `meta-customlayer` to `meta/alsa-lib/1.2.1.2-r0`.

# EXAMPLE USAGE

Check the [Preconfiguration](#PRECONFIGURATION) section above before running the script.

Use the following command to scan a Yocto build, create Black Duck project `myproject` and version `v1.0`, then update CVE patch status for identified CVEs:

    (If script installed) python3 $YOCTO_BM_LOC/import_yocto_bm.py -p myproject -v v1.0

To scan a Yocto project specifying a different build manifest as opposed to the most recent one:

    (Script installed) python3 $YOCTO_BM_LOC/import_yocto_bm.py -p myproject -v v1.0 -m tmp/deploy/licenses/core-image-sato-qemux86-64-20200728105751/package.manifest

To scan the most recent Yocto build in a different build folder location (not the current folder):

    (Script installed) python3 $YOCTO_BM_LOC/import_yocto_bm.py -p myproject -v v1.0 --y $HOME/newyocto/poky/build

To perform a CVE check patch analysis only use the command:

    (Script installed) python3 $YOCTO_BM_LOC/import_yocto_bm.py -p myproject -v v1.0 --cve_check_only

To create a JSON output scan without uploading (and no CVE patch update) use:

    (Script installed) python3 $YOCTO_BM_LOC/import_yocto_bm.py -p myproject -v v1.0 -o my.jsonld

To create a JSON output scan without uploading (and no CVE patch update) without checking recipes against the KB recipe list downloaded from Github:

    (Script installed) python3 $YOCTO_BM_LOC/import_yocto_bm.py -p myproject -v v1.0 -o my.jsonld --no_kb_check

# CVEs from cve_check Versus Black Duck

The Yocto `cve_check` class works on the Bitbake dependencies within the dev environment, and produces a list of CVEs identified from the NVD for ALL packages in the development environment.

This script extracts the packages from the build manifest (which will be a subset of those in the full Bitbake dependencies for build environment) and creates a Black Duck project.

The list of CVEs reported by `cve_check` will therefore be considerably larger than seen in the Black Duck project (whcih is the expected situation).

# OUTSTANDING ISSUES

The identification of the Linux Kernel version from the Bitbake recipes and association with the upstream component in the KB has not been completed yet. Until an automatic identification is possible, the required Linux Kernel component can be added manually to the Black Duck project.
