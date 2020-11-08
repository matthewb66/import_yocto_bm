# Synopsys Import Yocto Build Manifest - import_yocto_bm.py

# OVERVIEW
This script is provided under an OSS license as an example of how to use the Black Duck APIs to import components from a Yocto project manifest.

It does not represent any extension of licensed functionality of Synopsys software itself and is provided as-is, without warranty or liability.

# DESCRIPTION

The `import_yocto_bm.py` script is designed to import a Yocto project build manifest created by Bitbake. It replaces previous scripts (including https://github.com/matthewb66/import_yocto_build_manifest).

The script must be executed on a Linux workstation where Yocto has been installed and after a successful Bitbake build.

If invoked in the Yocto build folder (or the build folder is manually specified using the -y option), then it will locate the build-manifest file automatically in the tmp/deploy hierarchy.

If the Bitbake build was performed with the Yocto `cve_check` class configured, then the script will also optionally locate the CVE log exported by CVE check, extract patched CVEs and set the remediation status of matching CVEs in the Black Duck project.

It requires access to the Black Duck server via the API (see Prerequisites below) unless the -o option is used to create the output scan file for manual upload.

# SUPPORTED YOCTO PROJECTS

This script is designed to support Yocto versions 2.0 up to 3.1 for building projects.

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
				[--cve_check_only] [--no_cve_check]
				[--cve_check_file CVE_CHECK_FILE]

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
				determined from conf files)
	  -m BUILDCONF, --buildconf BUILDCONF
				Build config file (if not specified 
				poky/meta/conf/bitbake.conf will be used)
	  -l LOCALCONF, --localconf LOCALCONF
				Local config file (if not specified 
				poky/build/conf/local.conf will be used)
	  --arch ARCH           Architecture (if not specified then will be determined
				from conf files)
	  --cve_check_only      Only check for patched CVEs from cve_check and update
				existing project
	  --no_cve_check        Skip check for and update of patched CVEs
	  --cve_check_file CVE_CHECK_FILE
				CVE check output file (if not specified will be
				determined from conf files)

The script will use the invocation folder as the Yocto build folder (e.g. yocto_zeus/poky/build) by default (if there is a `build` sub-folder then it will be used instead). The `--yocto_folder` option can be used to specify the Yocto build folder as opposed to the invocation folder.

The `--project` and `--version` options are required to define the Black Duck project and version names for inclusion in the json output file (and update CVE patch status).

The `--output_json` option can be used to specify an output file for the project scan. If specified, then the scan will not be uploaded automatically and CVE patch checking will be skipped.

The Yocto target and architecture values are required to locate the manifest and cve\_check log files and will be extracted from the Bitbake config files automatically, but the `--target` and `--arch` options can be used to specify these manually.

The most recent Bitbake output manifest file (located in the `build/tmp/deploy/licenses/<image>-<target>-<datetime>/package.manifest` file) will be located automatically. Use the `--manifest` option to specify the manifest file manually.

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

# CVEs from cve_check Versus Black Duck

The Yocto `cve_check` class works on the Bitbake dependencies within the dev environment, and produces a list of CVEs identified from the NVD for ALL packages in the development environment.

This script extracts the packages from the build manifest (which will be a subset of those in the full Bitbake dependencies for build environment) and creates a Black Duck project.

The list of CVEs reported by `cve_check` will therefore be considerably larger than seen in the Black Duck project (whcih is the expected situation).

# OUTSTANDING ISSUES

The identification of the Linux Kernel version from the Bitbake recipes and association with the upstream component in the KB has not been completed yet. Until an automatic identification is possible, the required Linux Kernel component can be added manually to the Black Duck project.
