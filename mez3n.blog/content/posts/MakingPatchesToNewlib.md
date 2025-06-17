+++
title = 'Submitting Patches To Newlib'
date = 2025-06-17T16:12:01+03:00
draft = false
tags = ['programming', 'newlib', 'cygwin', 'git', 'open-source']
categories = ['Development']
+++

# Contributing to Newlib: A Step-by-Step Guide

If you're interested in contributing to Newlib, the C standard library used by many embedded systems (including RTEMS and Cygwin), this guide will walk you through the process of submitting patches.

## What is Newlib?

Newlib is a C standard library implementation intended for use on embedded systems. It's widely used in projects like RTEMS, Cygwin, and various bare-metal environments.

## Part 1: Creating and Submitting Patches

### 1. Clone the Repository

First, you need to clone the newlib-cygwin repository:

```bash
git clone https://sourceware.org/git/newlib-cygwin.git
cd newlib-cygwin
```

### 2. Make Your Changes

After cloning, make your desired changes to the codebase. Ensure your modifications follow the project's coding standards and conventions.

### 3. Stage Your Changes

Add all modified and new files to the staging area:

```bash
git add .
```

Alternatively, you can add specific files:

```bash
git add path/to/modified/file.c
```

### 4. Commit Your Changes

Create a commit with a descriptive message:

```bash
git commit -m "Brief description of changes" -m "Detailed explanation of what was changed and why"
```

**Example:**
```bash
git commit -m "Fix pthread_cond_clockwait header declaration" -m "Added missing clockid_t parameter and corrected function signature in pthread.h to match POSIX specification"
```

### 5. Generate a Patch File

Create a patch file for your last commit:

```bash
git format-patch HEAD~1
```

This generates a `.patch` file containing your changes.

### 6. Submit Your Patch

Submit your patch to the Newlib mailing list via email:

```bash
git send-email --to=newlib@sourceware.org your-patch-file.patch
```

**Note:** You may need to configure git send-email first:

```bash
git config --global sendemail.smtpserver your-smtp-server
git config --global sendemail.smtpuser your-email@example.com
```

## Part 2: Testing Patches in RTEMS

To test your patch in RTEMS, you'll need the RTEMS Source Builder (RSB). This tutorial assumes you've already followed the setup steps in the [RTEMS User Manual](https://docs.rtems.org/docs/6.1/user/start/index.html).

### 1. Clone the RTEMS Source Builder

```bash
git clone git@gitlab.rtems.org:rtems/rtems-source-builder.git RSB
cd RSB
```

### 2. Prepare Your Patch

Copy your patch file to the RSB patches directory:

```bash
cp /path/to/your/patch.patch rtems/patches/
```

Generate the SHA512 hash of your patch file:

```bash
sha512sum rtems/patches/0001-fix-pthread-cond-clockwait-header.patch
```

### 3. Find Your BSP Configuration

List available BSP configurations:

```bash
ls rtems/config/7/
```

**Expected output:**
```
rtems-aarch64.bset  rtems-arm.bset   rtems-default.bset  rtems-kernel.bset  
rtems-llvm.bset     rtems-m68k.bset  rtems-mips.bset    rtems-powerpc.bset  
rtems-sparc.bset    rtems-x86_64.bset rtems-tools.bset
```

For this example, we'll use SPARC. Check the SPARC configuration:

```bash
cat rtems/config/7/rtems-sparc.bset
```

**Output:**
```
%define release 1
%define rtems_arch sparc
%define with_libgomp
%define with_newlib_tls
%define gdb-disable-sim 1
%include 7/rtems-default.bset 
devel/sis-2-1
```

### 4. Trace the Configuration Chain

The `%include 7/rtems-default.bset` line tells us to check the default configuration:

```bash
cat rtems/config/7/rtems-default.bset
```

Look for the GCC configuration line:
```
%defineifnot with_rtems_gcc      tools/rtems-gcc-13.3-newlib-head
```

This points to the configuration file: `rtems/config/tools/rtems-gcc-13.3-newlib-head.cfg`

### 5. Add Your Patch to the Configuration

Edit the GCC configuration file and add your patch:

```bash
nano rtems/config/tools/rtems-gcc-13.3-newlib-head.cfg
```

Add these lines to the file:

```
%patch add newlib -p1 file://0001-fix-pthread-cond-clockwait-header.patch
%hash sha512 0001-fix-pthread-cond-clockwait-header.patch \
    YOUR_CALCULATED_SHA512_HASH_HERE
```

Replace `YOUR_CALCULATED_SHA512_HASH_HERE` with the hash you calculated in step 2.

### 6. Rebuild RTEMS Tools

Navigate to the RTEMS directory and rebuild with your patch:

```bash
cd rtems
../source-builder/sb-set-builder --prefix=/path/to/your/rtems/installation 7/rtems-sparc
```

Replace `/path/to/your/rtems/installation` with your actual RTEMS installation directory.

## Conclusion

Contributing to open-source projects like Newlib is a great way to improve your programming skills and give back to the community.

