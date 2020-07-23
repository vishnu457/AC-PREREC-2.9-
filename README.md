# AC-PREREC-2.9-
AC_PREREQ(2.59)

dnl Get the version of Subversion, using m4's esyscmd() command to do this
dnl at m4-time, since AC_INIT() requires it then.
AC_INIT([subversion],
     [esyscmd($PYTHON build/getversion.py SVN subversion/include/svn_version.h)],
     [http://subversion.apache.org/])

AC_CONFIG_SRCDIR(subversion/include/svn_types.h)
AC_CONFIG_AUX_DIR([build])

AC_MSG_NOTICE([Configuring Subversion ]AC_PACKAGE_VERSION)

AC_SUBST([abs_srcdir], ["`cd $srcdir && pwd`"])
AC_SUBST([abs_builddir], ["`pwd`"])
if test "$abs_srcdir" = "$abs_builddir"; then
  canonicalized_srcdir=""
else
  canonicalized_srcdir="$srcdir/"
fi
AC_SUBST([canonicalized_srcdir])

SWIG_LDFLAGS="$LDFLAGS"
AC_SUBST([SWIG_LDFLAGS])

# Generate config.nice early (before the arguments are munged)
SVN_CONFIG_NICE(config.nice)

# ==== Check for programs ====================================================

# Look for a C compiler (before anything can set CFLAGS)
CUSERFLAGS="$CFLAGS"
AC_PROG_CC
SVN_CC_MODE_SETUP

# Look for a C++ compiler (before anything can set CXXFLAGS)
CXXUSERFLAGS="$CXXFLAGS"
AC_PROG_CXX
SVN_CXX_MODE_SETUP

# Look for a C pre-processor
AC_PROG_CPP

# Look for a good sed
# AC_PROG_SED was introduced in Autoconf 2.59b
m4_ifdef([AC_PROG_SED], [AC_PROG_SED], [SED="${SED:-sed}"])

# Grab target_cpu, so we can use it in the Solaris pkginfo file
AC_CANONICAL_TARGET

# Look for an extended grep
AC_PROG_EGREP

AC_PROG_LN_S

AC_PROG_INSTALL
# If $INSTALL is relative path to our fallback install-sh, then convert
# to an absolute path, as in some cases (e.g. Solaris VPATH build), libtool
# may try to use it from a changed working directory.
if test "$INSTALL" = "build/install-sh -c"; then
  INSTALL="$abs_srcdir/$INSTALL"
fi

if test -z "$MKDIR"; then
  MKDIR="$INSTALL -d"
fi
AC_SUBST([MKDIR])

# ==== Libraries, for which we may have source to build ======================

dnl verify apr version and set apr flags
dnl These regular expressions should not contain "\(" and "\)".

APR_VER_REGEXES=["1\.[5-9]\. 2\."]

SVN_LIB_APR($APR_VER_REGEXES)

if test `expr $apr_version : 2` -ne 0; then
  dnl Bump the library so-version to 2 if using APR-2
  dnl (Debian uses so-version 1 for APR-1-with-largefile)
  svn_lib_ver=2
  dnl APR-2 provides APRUTIL
  apu_config=$apr_config
  AC_SUBST(SVN_APRUTIL_INCLUDES)
  AC_SUBST(SVN_APRUTIL_CONFIG, ["$apu_config"])
  AC_SUBST(SVN_APRUTIL_LIBS)
  SVN_APR_MAJOR_VERSION=2
else
  svn_lib_ver=0
  APU_VER_REGEXES=["1\.[3-9]\."]
  SVN_LIB_APRUTIL($APU_VER_REGEXES)
  SVN_APR_MAJOR_VERSION=1
fi
AC_SUBST(SVN_APR_MAJOR_VERSION)
SVN_LT_SOVERSION="-version-info $svn_lib_ver"
AC_SUBST(SVN_LT_SOVERSION)
AC_DEFINE_UNQUOTED(SVN_SOVERSION, $svn_lib_ver,
                   [Subversion library major version])

dnl Search for pkg-config
AC_PATH_PROG(PKG_CONFIG, pkg-config)

dnl Search for serf
SVN_LIB_SERF(1,3,4)

if test "$svn_lib_serf" = "yes"; then
  AC_DEFINE([SVN_HAVE_SERF], 1,
            [Defined if support for Serf is enabled])
fi

dnl Search for apr_memcache (only affects fs_fs)
SVN_LIB_APR_MEMCACHE

if test "$svn_lib_apr_memcache" = "yes"; then
  AC_DEFINE(SVN_HAVE_MEMCACHE, 1,
            [Defined if apr_memcache (standalone or in apr-util) is present])
fi

AC_ARG_ENABLE(apache-whitelist,
  AS_HELP_STRING([--enable-apache-whitelist=VER],
                 [Whitelist a particular Apache version number,
                  typically used to enable the use of a old version
                  patched by a distribution.]),
                 [apache_whitelist_ver=$enableval],
                 [apache_whitelist_ver=no])
HTTPD_WHITELIST="$apache_whitelist_ver"
AC_SUBST(HTTPD_WHITELIST)

dnl Find Apache with a recent-enough magic module number
SVN_FIND_APACHE(20051115, $apache_whitelist_ver)

dnl Search for SQLite.  If you change SQLITE_URL from a .zip to
dnl something else also update build/ac-macros/sqlite.m4 to reflect
dnl the correct command to unpack the downloaded file.
SQLITE_MINIMUM_VER="3.8.2"
SQLITE_RECOMMENDED_VER="3.8.11.1"
dnl Used to construct the SQLite download URL.
SQLITE_RECOMMENDED_VER_REL_YEAR="2015"
SQLITE_URL="https://www.sqlite.org/$SQLITE_RECOMMENDED_VER_REL_YEAR/sqlite-amalgamation-$(printf %d%02d%02d%02d $(echo ${SQLITE_RECOMMENDED_VER} | sed -e 's/\./ /g')).zip"

SVN_LIB_SQLITE(${SQLITE_MINIMUM_VER}, ${SQLITE_RECOMMENDED_VER},
               ${SQLITE_URL})

AC_ARG_ENABLE(sqlite-compatibility-version,
  AS_HELP_STRING([--enable-sqlite-compatibility-version=X.Y.Z],
                 [Allow binary to run against SQLite as old as ARG]),
  [sqlite_compat_ver=$enableval],[sqlite_compat_ver=no])

if test -n "$sqlite_compat_ver" && test "$sqlite_compat_ver" != no; then
  SVN_SQLITE_VERNUM_PARSE([$sqlite_compat_ver],
                          [sqlite_compat_ver_num])
  CFLAGS="-DSVN_SQLITE_MIN_VERSION='\"$sqlite_compat_ver\"' $CFLAGS"
  CFLAGS="-DSVN_SQLITE_MIN_VERSION_NUMBER=$sqlite_compat_ver_num $CFLAGS"
fi

SVN_CHECK_FOR_ATOMIC_BUILTINS
if test "$svn_cv_atomic_builtins" = "yes"; then
    AC_DEFINE(SVN_HAS_ATOMIC_BUILTINS, 1, [Define if compiler provides atomic builtins])
fi

dnl Set up a number of directories ---------------------

dnl Create SVN_BINDIR for proper substitution
if test "${bindir}" = '${exec_prefix}/bin'; then
        if test "${exec_prefix}" = "NONE"; then
                if test "${prefix}" = "NONE"; then
                        SVN_BINDIR="${ac_default_prefix}/bin"
                else
                        SVN_BINDIR="${prefix}/bin"
                fi
        else
                SVN_BINDIR="${exec_prefix}/bin"
        fi
else
        SVN_BINDIR="${bindir}"
fi

dnl fully evaluate this value. when we substitute it into our tool scripts,
dnl they will not have things such as ${bindir} available
SVN_BINDIR="`eval echo ${SVN_BINDIR}`"
AC_SUBST(SVN_BINDIR)

dnl provide ${bindir} in svn_private_config.h for use in compiled code
AC_DEFINE_UNQUOTED(SVN_BINDIR, "${SVN_BINDIR}",
        [Defined to be the path to the installed binaries])

dnl This purposely does *not* allow for multiple parallel installs.
dnl However, it is compatible with most gettext usages.
localedir='${datadir}/locale'
AC_SUBST(localedir)

dnl For SVN_LOCALE_DIR, we have to expand it to something.  See SVN_BINDIR.
if test "${prefix}" = "NONE" \
  && ( test "${datadir}" = '${prefix}/share' \
       || ( test "${datadir}" = '${datarootdir}' \
            && test "${datarootdir}" = '${prefix}/share' ) ); then
  exp_localedir='${ac_default_prefix}/share/locale'
else
  exp_localedir=$localedir
fi
SVN_EXPAND_VAR(svn_localedir, "${exp_localedir}")
AC_DEFINE_UNQUOTED(SVN_LOCALE_DIR, "${svn_localedir}",
                   [Defined to be the path to the installed locale dirs])

dnl Check for libtool -- we'll definitely need it for all our shared libs!
AC_MSG_NOTICE([configuring libtool now])
ifdef([LT_INIT], [LT_INIT], [AC_PROG_LIBTOOL])
AC_ARG_ENABLE(experimental-libtool,
  AS_HELP_STRING([--enable-experimental-libtool],[Use APR's libtool]),
  [experimental_libtool=$enableval],[experimental_libtool=no])

if test "$experimental_libtool" = "yes"; then
  echo "using APR's libtool"
  sh_libtool="`$apr_config --apr-libtool`"
  LIBTOOL="$sh_libtool"
  SVN_LIBTOOL="$sh_libtool"
else
  sh_libtool="$abs_builddir/libtool"
  SVN_LIBTOOL="\$(SHELL) \"$sh_libtool\""
fi
AC_SUBST(SVN_LIBTOOL)

dnl Determine the libtool version
changequote(, )dnl
lt_pversion=`$LIBTOOL --version 2>/dev/null|$SED -e 's/([^)]*)//g;s/^[^0-9]*//;s/[- ].*//g;q'`
lt_version=`echo $lt_pversion|$SED -e 's/\([a-z]*\)$/.\1/'`
lt_major_version=`echo $lt_version | cut -d'.' -f 1`
changequote([, ])dnl

dnl set the default parameters
svn_enable_static=yes
svn_enable_shared=yes

dnl check for --enable-static option
AC_ARG_ENABLE(static,
  AS_HELP_STRING([--enable-static],
                 [Build static libraries]),
  [svn_enable_static="$enableval"], [svn_enable_static="yes"])

dnl check for --enable-shared option
AC_ARG_ENABLE(shared,
  AS_HELP_STRING([--enable-shared],
                 [Build shared libraries]),
  [svn_enable_shared="$enableval"], [svn_enable_shared="yes"])

if test "$svn_enable_static" = "yes" && test "$svn_enable_shared" = "yes" ; then
  AC_MSG_NOTICE([building both shared and static libraries])
elif test "$svn_enable_static" = "yes" ; then
  AC_MSG_NOTICE([building static libraries only])
  LT_CFLAGS="-static $LT_CFLAGS"
  LT_LDFLAGS="-static $LT_LDFLAGS"
elif test "$svn_enable_shared" = "yes" ; then
  AC_MSG_NOTICE([building shared libraries only])
  if test "$lt_major_version" = "1" ; then
    LT_CFLAGS="-prefer-pic $LT_CFLAGS"
  elif test "$lt_major_version" = "2" ; then
    LT_CFLAGS="-shared $LT_CFLAGS"
  fi
  LT_LDFLAGS="-shared $LT_LDFLAGS"
else
  AC_MSG_ERROR([cannot disable both shared and static libraries])
fi

dnl Check for --enable-all-static option
AC_ARG_ENABLE(all-static,
  AS_HELP_STRING([--enable-all-static],
                 [Build completely static (standalone) binaries.]),
  [
    if test "$enableval" = "yes" ; then
      LT_LDFLAGS="-all-static $LT_LDFLAGS"
    elif test "$enableval" != "no" ; then
      AC_MSG_ERROR([--enable-all-static doesn't accept argument])
    fi
])

AC_SUBST(LT_CFLAGS)
AC_SUBST(LT_LDFLAGS)

AC_ARG_ENABLE(local-library-preloading,
  AS_HELP_STRING([--enable-local-library-preloading], 
                 [Enable preloading of locally built libraries in locally
                  built executables.  This may be necessary for testing
                  prior to installation on some platforms.  It does not
                  work on some platforms (Darwin, OpenBSD, ...).]),
  [
  if test "$enableval" != "no"; then
    if test "$svn_enable_shared" = "yes"; then
      TRANSFORM_LIBTOOL_SCRIPTS="transform-libtool-scripts"
    else
      AC_MSG_ERROR([--enable-local-library-preloading conflicts with --disable-shared])
    fi
  else
    TRANSFORM_LIBTOOL_SCRIPTS=""
  fi
  ], [
  TRANSFORM_LIBTOOL_SCRIPTS=""
])
AC_SUBST(TRANSFORM_LIBTOOL_SCRIPTS)

dnl Check if -no-undefined is needed for the platform.
dnl It should always work but with libtool 1.4.3 on OS X it breaks the build.
dnl So we only turn it on for platforms where we know we really need it.
AC_MSG_CHECKING([whether libtool needs -no-undefined])
case $host in
  *-*-cygwin*)
    AC_MSG_RESULT([yes])
    LT_NO_UNDEFINED="-no-undefined"
    ;;
  *)
    AC_MSG_RESULT([no])
    LT_NO_UNDEFINED=""
    ;;
esac
AC_SUBST(LT_NO_UNDEFINED)

dnl Check for trang.
trang=yes
AC_ARG_WITH(trang,
AS_HELP_STRING([--with-trang=PATH],
               [Specify the command to run the trang schema converter]),
[
    trang="$withval"
])
if test "$trang" = "yes"; then
    AC_PATH_PROG(TRANG, trang, none)
else
    TRANG="$trang"
    AC_SUBST(TRANG)
fi

dnl Check for doxygen
doxygen=yes
AC_ARG_WITH(doxygen,
AC_HELP_STRING([--with-doxygen=PATH],
               [Specify the command to run doxygen]),
[
    doxygen="$withval"
])
if test "$doxygen" = "yes"; then
    AC_PATH_PROG(DOXYGEN, doxygen, none)
else
    DOXYGEN="$doxygen"
    AC_SUBST(DOXYGEN)
fi


dnl Check for libraries --------------------

dnl Expat -------------------

AC_ARG_WITH(expat,
  AS_HELP_STRING([--with-expat=INCLUDES:LIB_SEARCH_DIRS:LIBS], 
                 [Specify location of Expat]),
                 [svn_lib_expat="$withval"],
                 [svn_lib_expat="::expat"])

# APR-util accepts "builtin" as an argument to this option so if the user
# passed "builtin" pretend the user didn't specify the --with-expat option
# at all. Expat will (hopefully) be found in apr-util.
test "_$svn_lib_expat" = "_builtin" && svn_lib_expat="::expat"

AC_MSG_CHECKING([for Expat])
if test -n "`echo "$svn_lib_expat" | $EGREP ":.*:"`"; then
  SVN_XML_INCLUDES=""
  for i in [`echo "$svn_lib_expat" | $SED -e "s/\([^:]*\):.*/\1/"`]; do
    SVN_XML_INCLUDES="$SVN_XML_INCLUDES -I$i"
  done
  SVN_XML_INCLUDES="${SVN_XML_INCLUDES## }"
  for l in [`echo "$svn_lib_expat" | $SED -e "s/.*:\([^:]*\):.*/\1/"`]; do
    LDFLAGS="$LDFLAGS -L$l"
  done
  for l in [`echo "$svn_lib_expat" | $SED -e "s/.*:\([^:]*\)/\1/"`]; do
    SVN_XML_LIBS="$SVN_XML_LIBS -l$l"
  done
  SVN_XML_LIBS="${SVN_XML_LIBS## }"
  old_CPPFLAGS="$CPPFLAGS"
  old_LIBS="$LIBS"
  CPPFLAGS="$CPPFLAGS $SVN_XML_INCLUDES"
  LIBS="$LIBS $SVN_XML_LIBS"
  AC_LINK_IFELSE([AC_LANG_SOURCE([[
#include <expat.h>
int main()
{XML_ParserCreate(NULL);}]])], svn_lib_expat="yes", svn_lib_expat="no")
  LIBS="$old_LIBS"
  if test "$svn_lib_expat" = "yes"; then
    AC_MSG_RESULT([yes])
  else
    SVN_XML_INCLUDES=""
    SVN_XML_LIBS=""
    CPPFLAGS="$CPPFLAGS $SVN_APRUTIL_INCLUDES"
    if test "$enable_all_static" != "yes"; then
      SVN_APRUTIL_LIBS="$SVN_APRUTIL_LIBS `$apu_config --libs`"
    fi
    AC_COMPILE_IFELSE([AC_LANG_SOURCE([[
#include <expat.h>
int main()
{XML_ParserCreate(NULL);}]])], svn_lib_expat="yes", svn_lib_expat="no")
    if test "$svn_lib_expat" = "yes"; then
      AC_MSG_RESULT([yes])
      AC_MSG_WARN([Expat found amongst libraries used by APR-Util, but Subversion libraries might be needlessly linked against additional unused libraries. It can be avoided by specifying exact location of Expat in argument of --with-expat option.])
    else
      AC_MSG_RESULT([no])
      AC_MSG_ERROR([Expat not found])
    fi
  fi
  CPPFLAGS="$old_CPPFLAGS"
else
  AC_MSG_RESULT([no])
  if test "$svn_lib_expat" = "yes"; then
    AC_MSG_ERROR([--with-expat option requires argument])
  elif test "$svn_lib_expat" = "no"; then
    AC_MSG_ERROR([Expat is required])
  else
    AC_MSG_ERROR([Invalid syntax of argument of --with-expat option])
  fi
fi
AC_SUBST(SVN_XML_INCLUDES)
AC_SUBST(SVN_XML_LIBS)

dnl Berkeley DB -------------------

# Berkeley DB on SCO OpenServer needs -lsocket
AC_CHECK_LIB(socket, socket)

# Build the BDB filesystem library only if we have an appropriate
# version of Berkeley DB.
case "$host" in
powerpc-apple-darwin*)
    # Berkeley DB 4.0 does not work on OS X.
    SVN_FS_WANT_DB_MAJOR=4
    SVN_FS_WANT_DB_MINOR=1
    SVN_FS_WANT_DB_PATCH=25
    ;;
*)
