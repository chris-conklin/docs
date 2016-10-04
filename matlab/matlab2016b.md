## Installing and Packaging Matlab 2016b

### Build Machine Requirements

- Create or locate a virtual machine with enough storage to store the deb file, install files and files for packaging (80 to 100 GB of space is sufficient)
- The sofware and utilities used throughout this process run best with 4 to 8 GB of ram and at least 4 cores.


### Getting the Software


- Create an iso of the install media and move it to the build machine.
- mount the iso.
```
  # mount -o loop /path/to/iso /mnt
```
- Optionally, use some other soft copy of the installer files (extracted archive etc.) saved in a directory on the build machine (i.e. /opt/sw_name/).

### Installation on Build Machine

Assuming that the installation software has been made available in **/opt/matlab/**:

- Use a sensible umask (ex 022)

- Install the software on the build machine by executing the following as a *privledged user*:
```
# /opt/matlab/install
```
**note: The matlab installer used here, will require Xauthority configuration. You can copy
the .Xauthority file from your afs home directory into ~/root .

- When prompted, select the offline method of installation. You will need to provide an 
installation key in order to use this method. A sample key can be provided by software aquisition. 
Copy and paste the key into the textbox and continue.

- Agree to the terms of service.

- Review the install directory entry - we are using **/usr/local/MATLAB/R2016b**. 

- Review the software list that is displayed and ensure that all entries are selected for install.

- When prompted for link creation, leave the checkbox unchecked. The links will be created later manually.

- Complete the installation.


### Post-installation Review
- Review the permissions and ownership of the install directory: **/usr/local/MATLAB/**

- Copy the contents of **/etc/MATLAB/network.lic** from an existing matlab client to the the corresponding location on the build machine. 

- Launch the application on the build machine as an unprivledged user:
```
$ /usr/local/MATLAB/R2016b/bin/matlab
```

- When prompted enter the path to the network license file in **/etc/MATLAB**.

- Close the application and remove the license file entry that was created by the installer in the **/usr/local/MATLAB/licenses/** directory.

- Remove the **/etc/MATLAB/network.lic** file.

## Packaging the Installed Software

### Software Package Naming Convention

Packages are named according to the following format:

```<package name>_<package version>-<build version>_<arch>.deb```

Ex: matlab-2016b_matlab-2016b-1_amd64.deb

Where
   - The **package name** is the name provided by the software providor. This can and often does include a vendor version number. 
In our case, matlab-2016b is the package name.

   - The **package version** is the version of the software being installed. This can be identical to the package name
on initial install but can change once package updates or patches are issued and the software is repackaged. 
In our case matlab-2016b is the package number

   - The **build version** is an incremental value, starting with 1, that is incremented once structural changes are made
to an otherwise identical package. For example if a package is distributed to hosts, but an issue is found with the control
file. The package can be repackaged with a corrected version of the control file, and the build version would be 
incremented. The rebuilt package would be added to the repository and upon a host's package update run, the package would 
be identified as an update candidate.

   - The **arch** reflects the architecture that is targeted by the package.

### Package the Software

- Create a work area for package creation, typically located within **/tmp** or **/opt** - using **/opt** here.

- Move the contents of the local install to your work area, preserving file and directory attributes:

    ```
   # tar -cpf - /usr/local/MATLAB | ( cd /opt/matlab-2016b ; tar -xvpf - ) 

   ```

Check the result paying attention to file permissions and the directory structure.

```
/opt/matlab-2016b/
   usr/
      local/
         MATLAB/
            R2016b/
```

We will now be working exclusively inside of the new working directory /opt/matlab-2016b/

- Create directory /opt/matlab-2016b/etc/MATLAB

- Create a symbolic link in **R2016b/licenses/**
```
lrwxrwxrwx 1 root root 23 Apr 26  2016 network.lic -> /etc/MATLAB/network.lic
```
** note: this link will be broken but will work on the installation target once CFEngine copies the
license information into place.


- Create directory **/opt/matlab-2016b/usr/local/bin/**

- Create symbolic links in the new directory to the matlab executables

  **(ln -s ../MATLAB/R2016b/bin/matlab matlab2016b)**

```
lrwxrwxrwx 1 root root 27 Sep 27 13:35 matlab2016b -> ../MATLAB/R2016b/bin/matlab
lrwxrwxrwx 1 root root 27 Sep 27 13:36 matlab-2016b -> ../MATLAB/R2016b/bin/matlab
lrwxrwxrwx 1 root root 27 Sep 27 13:66 mbuild2016b -> ../MATLAB/R2016b/bin/mbuild
lrwxrwxrwx 1 root root 27 Sep 27 13:46 mbuild-2016b -> ../MATLAB/R2016b/bin/mbuild
lrwxrwxrwx 1 root root 24 Sep 27 13:47 mcc2016b -> ../MATLAB/R2016b/bin/mcc
lrwxrwxrwx 1 root root 24 Sep 27 13:71 mcc-2016b -> ../MATLAB/R2016b/bin/mcc
lrwxrwxrwx 1 root root 24 Sep 27 13:47 mex2016b -> ../MATLAB/R2016b/bin/mex
lrwxrwxrwx 1 root root 24 Sep 27 13:47 mex-2016b -> ../MATLAB/R2016b/bin/mex
```


### The Control File 

- Details about the control file can be found on the [Debian Policy Manual Website]([https://www.debian.org/doc/debian-policy/ch-controlfields.htm]()
- Create directory **/opt/matlab-2016b/DEBIAN**
- Create a control file for the package in **/opt/matlab-2016b/DEBIAN/control** with the followig content:
```
  Package: matlab-2016b
  Version: 2016b-1
  Priority: extra
  Section: main
  Architecture: amd64
  Maintainer: linux@unc.edu
  Description: Matlab from The Mathworks version 2016b.
```

** note : If the Description spans more than a single line, indent the additional lines with two spaces.

The working directory structure should have the following form:
```
/opt/matlab-2016b/
   DEBIAN/
      control
   etc/
      MATLAB/
   usr/
      local/
         bin/
         MATLAB/
            R2016b/
```

### Creating the .deb Installer Package

- Ensure that all of the files in the working directory are owned by root.
- Create a deb package by executing the following command outside of the working directory:

```	  
# dpkg -b matlab-2016b_2016b-1_amd64.deb /opt/matlab-2016b
```


### The Matlab meta package

In addition to the matlab application package there is also an Ubuntu meta package that must be updated. The meta packages do not contain actual software, they simply depend on other packages to be installed. 

In particular, check the symbolic links that are created in the meta package and ensure that all links are present, and create new links where necessary. 

Update the control file contained in the meta package to reflect the updated matlab version information.

Create a file structure similar to the one created to build the matlab deb package
