Introduction
============

PAPPL is a simple C-based framework/library for developing CUPS Printer
Applications, which are the recommended replacement for printer drivers.  It is
sufficiently general purpose to support any kind of printer or driver that can
be used on desktops, servers, and in embedded environments.

PAPPL embeds a multi-threaded HTTP /
[IPP Everywhere™](https://www.pwg.org/ipp/everywhere.html)
server and provides callbacks for a variety of events that allows a GUI or
command-line application to interact with both the local user that is running
the Printer Application and any network clients that are submitting print jobs,
querying printer status and capabilities, and so forth.

PAPPL provides a simple driver interface for raster graphics printing, and
developers of CUPS Raster drivers will readily adapt to it.  PAPPL can also be
used with printers that support vector graphics printing although you'll have
to develop more code to support them.  Drivers provide configuration and
capability information to PAPPL, and PAPPL then calls the driver to print things
as needed.  PAPPL automatically supports printing of JPEG, PNG, PWG Raster,
Apple Raster, and "raw" files to printers connected via USB and network
(AppSocket/JetDirect) connections.  Other formats can be supported through
"filter" callbacks you register.

PAPPL is Copyright © 2019-2020 by Michael R Sweet and is licensed under the
Apache License Version 2.0 with an (optional) exception to allow linking against
GPL2/LGPL2 software (like older versions of CUPS), so it can be used *freely* in
any project you'd like.  See the files "LICENSE" and "NOTICE" in the source
distribution for more information.


Requirements
------------

PAPPL requires a POSIX-compliant host operating system such as Linux®, macOS®,
QNX®, or VxWorks®, a C99 compiler like Clang or GCC, a `make` program that
supports the `include` directive, and the following support libraries:

- CUPS 2.2 or later for the CUPS libraries (libcups2/libcupsimage2)
- GNU TLS 2.8 or later (except on macOS) for TLS support
- JPEGLIB 9 or later for JPEG image support
- LIBPNG 1.6 or later for PNG image support
- PAM for authentication support
- ZLIB 1.1 or later for compression support

Run the following commands to install the prerequisites on CentOS 7, Fedora 22,
and Red Hat Enterprise Linux 7:

    sudo yum groupinstall 'Development Tools'
    sudo yum install avahi-devel cups-devel gnutls-devel libjpeg-turbo-devel \
        libpam-devel libpng-devel libusbx-devel pam-devel zlib-devel

Run the following commands to install the prerequisites on CentOS 8, Fedora 23
or later, and Red Hat Enterprise Linux 8:

    sudo dnf groupinstall 'Development Tools'
    sudo dnf install avahi-devel cups-devel gnutls-devel libjpeg-turbo-devel \
        libpam-devel libpng-devel libusbx-devel pam-devel zlib-devel

Run the following commands to install the prerequisites on Debian GNU/Linux,
Raspbian, and Ubuntu:

    sudo apt-get install build-essential libavahi-client-dev libcups2-dev \
        libcupsimage2-dev libgnutls28-dev libjpeg-dev libpam-dev libpng-dev \
        libusb-1.0-0-dev zlib1g-dev

Finally, after installing Xcode from the AppStore run the following commands to
install the prerequisites on macOS:

    (install brew if necessary)
    brew install libjpeg
    brew install libpng
    brew install libusb

or download, build, and install libjpeg, libpng, and libusb from source.


Building PAPPL
--------------

PAPPL uses the usual `configure` script to generate a `make` file:

    ./configure [options]
    make

Use `./configure --help` to see a full list of options.

There is also an Xcode project under the `xcode` directory that can be used on
macOS:

    open xcode/pappl.xcodeproj

You can test the build by running the PAPPL test program:

    testsuite/testpappl


Installing PAPPL
----------------

Once you have successfully built PAPPL, install it using:

    sudo make install

By default everything will be installed under `/usr/local`.  Use the `--prefix`
configure option to override the base installation directory.  Set the
`DESTDIR`, `DSTROOT`, or `RPM_BUILD_ROOT` environment variables to redirect the
installation to a staging area, as is typically done for most software packaging
systems (using one of those environment variables...)


Detecting PAPPL
---------------

PAPPL can be detected using the `pkg-config` command, for example:

    if pkg-config --exists pappl; then
        ... 
    fi

In a makefile you can add the necessary compiler and linker options with:

    CFLAGS  +=      `pkg-config --cflags pappl`
    LIBS    +=      `pkg-config --libs pappl`


Header Files
------------

PAPPL provides a top-level header file that should be used:

    #include <pappl/pappl.h>

This header includes all of the base object headers in PAPPL as well as the
CUPS header files that provide the HTTP and IPP support functions.


API Overview
============

PAPPL provides five main objects:

- [The System](@) (`pappl_system_t`): The main object that manages the whole
  printer application;
- [Clients](@) (`pappl_client_t`): The objects that manage client connections;
- [Devices](@) (`pappl_device_t`): The objects that manage printer connections;
- [Printers](@) (`pappl_printer_t`): The objects that manage printers; and
- [Jobs](@) (`pappl_job_t`): The objects that manage print jobs.

![PAPPL Block Diagram](pappl-block.png#width100)


The System
----------

The system is an object of type `pappl_system_t` that manages client and device
connections, listeners, the log, printers, and resources.  It implements a
subset of the IPP System Service
([PWG 5100.22](https://ftp.pwg.org/pub/pwg/candidates/cs-ippsystem10-20191122-5100.22.pdf))
with each printer implementing IPP Everywhere™
([PWG 5100.14](https://ftp.pwg.org/pub/pwg/candidates/cs-ippeve11-20200515-5100.14.pdf))
and some extensions to provide compatibility with the full range of mobile and
desktop client devices.  In addition, it provides an optional embedded web
interface, raw socket printing, and USB printer gadget (Linux only).

A system object is created using the [`papplSystemCreate`](@@) function and
deleted using the [`papplSystemDelete`](@@) function:

```
pappl_system_t *
papplSystemCreate(pappl_soptions_t options, const char *name, int port,
    const char *subtypes, const char *spooldir, const char *logfile,
    pappl_loglevel_t loglevel, const char *auth_service, bool tls_only);

void
papplSystemDelete(pappl_system_t *system);
```


### Filters ###

Filters allow a printer application to support additional file formats and/or
provide optimized support for existing file formats.  Filters are added using
the [`papplSystemAddMIMEFilter`](@@) function:

```
void
papplSystemAddMIMEFilter(pappl_system_t *system, const char *srctype,
    const char *dsttype, pappl_mime_filter_cb_t cb, void *data);
```


### Listeners ###

The [`papplSystemAddListeners`](@@) function adds one or more listener sockets
for the system:

```
bool
papplSystemAddListeners(pappl_system_t *system, const char *name);
```


### Run Loops ###

PAPPL provides two functions to manage its run loops.  The first is
[`papplSystemRun`](@@) which just runs a system object that you have created
and configured:

```
void
papplSystemRun(pappl_system_t *system);
```

This function runs until it receives a Shutdown-System request or a termination
signal.

The second is a convenience layer call [`papplMainloop`](@@) which handles
processing standard command-line arguments and sub-commands for a printer
application:

```
int
papplMainloop(int argc, char *argv[],
              const char *version, const char *footer_html,
              int num_drivers, pappl_pr_driver_t *drivers,
              pappl_pr_driver_cb_t driver_cb, pappl_ml_autoadd_cb_t autoadd_cb,
              const char *subcmd_name, pappl_ml_subcmd_cb_t subcmd_cb,
              pappl_ml_system_cb_t system_cb, pappl_ml_usage_cb_t usage_cb,
              void *data);
```

You can learn more about this function in the chapter on the
[HP Printer Application Example](@).


### Resources ###

PAPPL provides several functions for adding static and dynamic resources to the
embedded HTTP server:

```
void
papplSystemAddResourceCallback(pappl_system_t *system, const char *path,
    const char *format, pappl_resource_cb_t cb, void *data);

void
papplSystemAddResourceData(pappl_system_t *system, const char *path,
    const char *format, const void *data, size_t datalen);

void
papplSystemAddResourceDirectory(pappl_system_t *system, const char *basepath,
    const char *directory);

void
papplSystemAddResourceFile(pappl_system_t *system, const char *path,
    const char *format, const char *filename);

void
papplSystemAddResourceString(pappl_system_t *system, const char *path,
    const char *format, const char *data);

void
papplSystemAddStringsData(pappl_system_t *system, const char *path,
    const char *language, const char *data);

void
papplSystemAddStringsFile(pappl_system_t *system, const char *path,
    const char *language, const char *filename);
```


```
void
papplSystemRemoveResource(pappl_system_t *system, const char *path);
```


### Navigation Links ###

```
void
papplSystemAddLink(pappl_system_t *system, const char *label,
    const char *path_or_url, bool secure);

void
papplSystemRemoveLink(pappl_system_t *system, const char *label);
```


Clients
-------

The PAPPL client functions provide access to client connections.  Client
connections and the life cycle of the `pappl_client_t` objects are managed
automatically by the system object for the printer application.

The `papplClientGet` functions get the current values for various client-
supplied request data:

- [`papplClientGetCSRFToken`](@@): Get the current CSRF token.
- [`papplClientGetCookie`](@@): Get the value of a named cookie.
- [`papplClientGetForm`](@@): Get form data from the client request.
- [`papplClientGetHTTP`](@@): Get the HTTP connection associate with a client.
- [`papplClientGetHostName`](@@): Get the host name used for the client request.
- [`papplClientGetHostPort`](@@): Get the host port used for the client request.
- [`papplClientGetJob`](@@): Get the job object associated with the client
  request.
- [`papplClientGetMethod`](@@): Get the HTTP request method.
- [`papplClientGetOperation`](@@): Get the IPP operation code.
- [`papplClientGetOptions`](@@): Get any options from the HTTP request URI.
- [`papplClientGetPrinter`](@@): Get the printer object associated with the
  client request.
- [`papplClientGetRequest`](@@): Get the IPP request.
- [`papplClientGetResponse`](@@): Get the IPP response.
- [`papplClientGetSystem`](@@): Get the system object associated with the
  client.
- [`papplClientGetURI`](@@): Get the HTTP request URI.
- [`papplClientGetUsername`](@@): Get the authenticated username, if any.


### Responding to Client Requests ###

The [`papplClientRespond`](@@) function starts a HTTP response to a client
request.  The [`papplClientHTMLHeader`](@@) and [`papplClientHTMLFooter`](@@)
functions send standard HTML headers and footers for the printer application's
configured web interface while the [`papplClientHTMLEscape`](@@),
[`papplClientHTMLPrintf`](@@), [`papplClientHTMLPuts`](@@), and
[`papplClientHTMLStartForm`](@@) functions send HTML messages or strings.  Use
the [`papplClientGetHTTP`](@@) and (CUPS) `httpWrite2` functions to send
arbitrary data in a client response.  Cookies can be included in web browser
requests using the [`papplClientSetCookie`](@@) function.

The [`papplClientRespondIPP`](@@) function starts an IPP response.  Use the
various CUPS `ippAdd` functions to add attributes to the response message.

The [`papplClientRespondRedirect`](@@) function sends a redirection response to
the client.


### HTML Forms ###

PAPPL provides the [`papplClientGetCSRFToken`](@@), [`papplClientGetForm`](@@),
[`papplClientHTMLStartForm`](@@), and [`papplClientValidateForm`](@@) functions
to securely manage HTML forms.

The [`papplClientHTMLStartForm`](@@) function starts a HTML form and inserts a
hidden variable containing a CSRF token that was generated by PAPPL from a
secure session key that is periodically updated.  Upon receipt of a follow-up
form submission request, the [`papplClientGetForm`](@@) and
[`papplClientValidateForm`](@@) functions can be used to securely read the form
data (including any file attachments) and validate the hidden CSRF token.


### Authentication and Authorization ###

PAPPL supports both user-based authentication using PAM modules and a simple
cookie-based password authentication mechanism that is used to limit
administrative access through the web interface.

The [`papplHTMLAuthorize`](@@) function authorizes access to the web interface
and handles displaying an authentication form on the client's web browser.
The return value indicates whether the client is authorized to access the web
page.

The [`papplIsAuthorized`](@@) function can be used to determine whether the
current client is authorized to perform administrative operations and is
normally only used for IPP clients.  Local users are always authorized while
remote users must provide credentials (typically a username and password) for
access.  This function will return an HTTP status code that can be provided to
the [`httpClientSendResponse`](@@) function. The value `HTTP_STATUS_CONTINUE`
indicates that authorization is granted and the request should continue.  The
[`papplGetUsername`](@@) function can be used to obtain the authenticated user
identity.


Devices
-------

The PAPPL device functions provide access to output device connections and to
list available output devices.  Output devices are accessed using Uniform
Resource Identifier (URI) strings such as "file:///path/to/file-or-directory",
"socket://11.22.33.44", and "usb://make/model?serial=number".  The follow URI
schemes are supported by PAPPL:

- "dnssd": Network (AppSocket) printers discovered via DNS-SD/mDNS (Bonjour),
- "file": Local files and directories,
- "snmp": Network (AppSocket) printers discovered via SNMPv1,
- "socket": Network (AppSocket) printers using a numeric IP address or hostname
  and optional port number, and
- "usb": Local USB printer.

The [`papplDeviceList`](@@) function lists available output devices, providing
each available output device to the supplied callback function.  The list only
contains devices whose URI scheme supports discovery, at present USB printers
and network printers that advertise themselves using DNS-SD/mDNS and/or SNMPv1.

The [`papplDeviceOpen`](@@) function opens a connection to an output device
using its URI.  The [`papplDeviceClose`](@@) function closes the connection.

The [`papplDevicePrintf`](@@), [`papplDevicePuts`](@@), and
[`papplDeviceWrite`](@@) functions send data to the device, while the
[`papplDeviceRead`](@@) function reads data from the device.

The [`papplDeviceGetMetrics`](@@) function gets statistical information about
all communications with the device while it has been open, while the
[`papplDeviceGetStatus`](@@) function gets the hardware status of a device and
maps it to the [`pappl_preason_t`](@@) bitfield.


### Custom Devices ###

PAPPL supports custom device URI schemes which are registered using the
[`papplDeviceAddScheme'](@@) function:

```
void
papplDeviceAddScheme(const char *scheme, pappl_dtype_t dtype,
    pappl_devlist_cb_t list_cb, pappl_devopen_cb_t open_cb,
    pappl_devclose_cb_t close_cb, pappl_devread_cb_t read_cb,
    pappl_devwrite_cb_t write_cb, pappl_devstatus_cb_t status_cb);
```

The "scheme" parameter specifies the URI scheme and must consist of lowercase
letters, digits, "-", "_", and/or ".", for example "x-foo" or "com.example.bar".

The "dtype" parameter specifies the device type and should be
`PAPPL_DTYPE_CUSTOM_LOCAL` for locally connected printers and
`PAPPL_DTYPE_CUSTOM_NETWORK` for network printers.

Each of the callbacks corresponds to one of the `papplDevice` functions.  The
"open\_cb" callback typically calls [`papplDeviceSetData`](@@) to store a pointer
to contextual information for the connection while the "close\_cb", "read\_cb",
"write\_cb", and "status\_cb" callbacks typically call
[`papplDeviceGetData`](@@) to retrieve it.


Printers
--------

Printers are managed by the system and are represented by the
[`pappl_printer_t`](#pappl_printer_t) type.  Each printer is connected to a
device and uses a driver to process document data and produce output.  PAPPL
supports raster printers out-of-the-box and provides filter callbacks to support
other kinds of printers.


### Navigation Links ###

```
void
papplPrinterAddLink(pappl_printer_t *printer, const char *label,
    const char *path_or_url, bool secure);

void
papplPrinterRemoveLink(pappl_printer_t *printer, const char *label);
```



Jobs
----




The HP Printer Application Example
==================================

This chapter will guide you through creating a simple PCL printer application
based on the old CUPS "rastertohp" filter.  The complete code can be found in
the [hp-printer-app](https://github.com/michaelrsweet/hp-printer-app) project
and serves as an overgrown "Hello, World!" program for PAPPL.


The Main Loop
-------------

All printer applications require some sort of a main loop for processing IPP
requests, printing files, and so forth.  PAPPL provides the
[`papplMainloop`](#papplMainLoop) convenience function that provides a standard
command-line interface, and in the "hp-printer-app" project the `main` function
just calls `papplMainloop` to do all of the work:

```
int
main(int  argc, char *argv[])
{
  return (papplMainloop(argc, argv,
                        /*version*/"1.0",
                        /*footer_html*/NULL,
                        (int)(sizeof(pcl_drivers) / sizeof(pcl_drivers[0])),
                        pcl_drivers, pcl_callback, pcl_autoadd,
                        /*subcmd_name*/NULL, /*subcmd_cb*/NULL,
                        /*system_cb*/NULL,
                        /*usage_cb*/NULL,
                        /*data*/NULL));
}
```

As you can see, we pass in the command-line arguments, a version number ("1.0")
for our printer application, a list of drivers supported by the printer
application, a callback for the driver that will configure a printer for a
named driver, and a callback for automatically adding printers.

The "footer_html" argument can be provided to override the default footer text
that appears at the bottom of the web interface.  In this case we are passing
`NULL` to use the default.

The drivers list is a collection of names, descriptions, IEEE-1284 device IDs,
and extension pointers.  Ours looks like this:

```
static pappl_pr_driver_t pcl_drivers[] =// Driver information
{   /* name */          /* description */       /* device ID */ /* extension */
  { "hp_deskjet",       "HP Deskjet",           NULL,           NULL },
  { "hp_generic",       "Generic PCL",          "CMD:PCL;",     NULL },
  { "hp_laserjet",      "HP LaserJet",          NULL,           NULL }
};
```

[The driver callback](@) is responsible for providing the data associated with
each driver, while [the auto-add callback](@) tells PAPPL which driver to use
for the printers it finds.

The "subcmd\_name" and "subcmd\_cb" arguments specify a custom sub-command for
the printer application along with a function to call.  Since this printer
application has no custom commands, we pass `NULL` for both.

The "system_cb" argument specifies a callback for creating the
[system object](#the-system).  We pass `NULL` because we want the default
system object for this printer application, which supports multiple printers
and a web interface.

The "usage_cb" argument specifies a callback from displaying the program usage
message.  Passing `NULL` yields the default usage message.

The "data" argument specifies a pointer that will be passed to any of the
callback functions.  We pass `NULL` since we do not have any extra contextual
information to provide to the callbacks.

The `papplMainloop` function runs until all processing for the current
sub-command is complete, returning the exit status for the program.


The Driver Callback
-------------------

The PAPPL driver callback is called when the system is creating a new printer
object.  It receives pointers to the system, driver name, device URI, a
driver data structure, an IPP attributes pointer, and the callback data, and it
returns a boolean indicating whether the driver callback was successful:

```
typedef bool (*pappl_pr_driver_cb_t)(pappl_system_t *system,
    const char *driver_name, const char *device_uri,
    pappl_pr_driver_data_t *driver_data, ipp_t **driver_attrs, void *data);
```

A driver callback can communicate with the printer via its device URI as needed
to configure the driver, however our printer application doesn't need to do
that.

The first thing our `pcl_callback` function does is to set the printer
callbacks in the driver data structure:

```
driver_data->printfile_cb    = pcl_print;
driver_data->rendjob_cb      = pcl_rendjob;
driver_data->rendpage_cb     = pcl_rendpage;
driver_data->rstartjob_cb    = pcl_rstartjob;
driver_data->rstartpage_cb   = pcl_rstartpage;
driver_data->rwriteline_cb   = pcl_rwriteline;
driver_data->status_cb       = pcl_status;
```

The `pcl_print` function prints a raw PCL file while the `pcl_r` functions print
raster graphics.  The `pcl_status` updates the printer status.

Next is the printer's native print format as a MIME media type, in this case HP
PCL:

```
driver_data->format          = "application/vnd.hp-pcl";
```

The default orientation and print quality follow:

```
driver_data->orient_default  = IPP_ORIENT_NONE;
driver_data->quality_default = IPP_QUALITY_NORMAL;
```

Then the values for the HP DeskJet driver:

```
if (!strcmp(driver_name, "hp_deskjet"))
{
  strncpy(driver_data->make_and_model, "HP DeskJet", sizeof(driver_data->make_and_model) - 1);

  driver_data->num_resolution  = 3;
  driver_data->x_resolution[0] = 150;
  driver_data->y_resolution[0] = 150;
  driver_data->x_resolution[1] = 300;
  driver_data->y_resolution[1] = 300;
  driver_data->x_resolution[2] = 600;
  driver_data->y_resolution[2] = 600;
  driver_data->x_default = driver_data->y_default = 300;

  driver_data->raster_types = PAPPL_PWG_RASTER_TYPE_BLACK_1 | PAPPL_PWG_RASTER_TYPE_BLACK_8 | PAPPL_PWG_RASTER_TYPE_SGRAY_8 | PAPPL_PWG_RASTER_TYPE_SRGB_8;

  driver_data->color_supported = PAPPL_COLOR_MODE_AUTO | PAPPL_COLOR_MODE_AUTO_MONOCHROME | PAPPL_COLOR_MODE_COLOR | PAPPL_COLOR_MODE_MONOCHROME;
  driver_data->color_default   = PAPPL_COLOR_MODE_AUTO;

  driver_data->num_media = (int)(sizeof(pcl_hp_deskjet_media) / sizeof(pcl_hp_deskjet_media[0]));
  memcpy(driver_data->media, pcl_hp_deskjet_media, sizeof(pcl_hp_deskjet_media));

  driver_data->sides_supported = PAPPL_SIDES_ONE_SIDED;
  driver_data->sides_default   = PAPPL_SIDES_ONE_SIDED;

  driver_data->num_source = 3;
  driver_data->source[0]  = "tray-1";
  driver_data->source[1]  = "manual";
  driver_data->source[2]  = "envelope";

  driver_data->num_type = 5;
  driver_data->type[0] = "stationery";
  driver_data->type[1] = "bond";
  driver_data->type[2] = "special";
  driver_data->type[3] = "transparency";
  driver_data->type[4] = "photographic-glossy";

  driver_data->left_right = 635;       // 1/4" left and right
  driver_data->bottom_top = 1270;      // 1/2" top and bottom

  for (i = 0; i < driver_data->num_source; i ++)
  {
    if (strcmp(driver_data->source[i], "envelope"))
      snprintf(driver_data->media_ready[i].size_name, sizeof(driver_data->media_ready[i].size_name), "na_letter_8.5x11in");
    else
      snprintf(driver_data->media_ready[i].size_name, sizeof(driver_data->media_ready[i].size_name), "env_10_4.125x9.5in");
  }
}
```

and the generic HP driver:

```
else if (!strcmp(driver_name, "hp_generic"))
{
  strncpy(driver_data->make_and_model, "Generic PCL Laser Printer", sizeof(driver_data->make_and_model) - 1);

  driver_data->num_resolution  = 2;
  driver_data->x_resolution[0] = 300;
  driver_data->y_resolution[0] = 300;
  driver_data->x_resolution[1] = 600;
  driver_data->y_resolution[1] = 600;
  driver_data->x_default = driver_data->y_default = 300;

  driver_data->raster_types = PAPPL_PWG_RASTER_TYPE_BLACK_1 | PAPPL_PWG_RASTER_TYPE_BLACK_8 | PAPPL_PWG_RASTER_TYPE_SGRAY_8;
  driver_data->force_raster_type = PAPPL_PWG_RASTER_TYPE_BLACK_1;

  driver_data->color_supported = PAPPL_COLOR_MODE_MONOCHROME;
  driver_data->color_default   = PAPPL_COLOR_MODE_MONOCHROME;

  driver_data->num_media = (int)(sizeof(pcl_generic_pcl_media) / sizeof(pcl_generic_pcl_media[0]));
  memcpy(driver_data->media, pcl_generic_pcl_media, sizeof(pcl_generic_pcl_media));

  driver_data->sides_supported = PAPPL_SIDES_ONE_SIDED | PAPPL_SIDES_TWO_SIDED_LONG_EDGE | PAPPL_SIDES_TWO_SIDED_SHORT_EDGE;
  driver_data->sides_default   = PAPPL_SIDES_ONE_SIDED;

  driver_data->num_source = 7;
  driver_data->source[0]  = "default";
  driver_data->source[1]  = "tray-1";
  driver_data->source[2]  = "tray-2";
  driver_data->source[3]  = "tray-3";
  driver_data->source[4]  = "tray-4";
  driver_data->source[5]  = "manual";
  driver_data->source[6]  = "envelope";

  driver_data->left_right = 635;      // 1/4" left and right
  driver_data->bottom_top = 423;      // 1/6" top and bottom

  for (i = 0; i < driver_data->num_source; i ++)
  {
    if (strcmp(driver_data->source[i], "envelope"))
      snprintf(driver_data->media_ready[i].size_name, sizeof(driver_data->media_ready[i].size_name), "na_letter_8.5x11in");
    else
      snprintf(driver_data->media_ready[i].size_name, sizeof(driver_data->media_ready[i].size_name), "env_10_4.125x9.5in");
  }
}
```

and the HP LaserJet driver:

```
else if (!strcmp(driver_name, "hp_laserjet"))
{
 strncpy(driver_data->make_and_model, "HP LaserJet", sizeof(driver_data->make_and_model) - 1);

  driver_data->num_resolution  = 3;
  driver_data->x_resolution[0] = 150;
  driver_data->y_resolution[0] = 150;
  driver_data->x_resolution[1] = 300;
  driver_data->y_resolution[1] = 300;
  driver_data->x_resolution[2] = 600;
  driver_data->y_resolution[2] = 600;
  driver_data->x_default = driver_data->y_default = 300;

  driver_data->raster_types = PAPPL_PWG_RASTER_TYPE_BLACK_1 | PAPPL_PWG_RASTER_TYPE_BLACK_8 | PAPPL_PWG_RASTER_TYPE_SGRAY_8;
  driver_data->force_raster_type = PAPPL_PWG_RASTER_TYPE_BLACK_1;

  driver_data->color_supported = PAPPL_COLOR_MODE_MONOCHROME;
  driver_data->color_default   = PAPPL_COLOR_MODE_MONOCHROME;

  driver_data->num_media = (int)(sizeof(pcl_hp_laserjet_media) / sizeof(pcl_hp_laserjet_media[0]));
  memcpy(driver_data->media, pcl_hp_laserjet_media, sizeof(pcl_hp_laserjet_media));

  driver_data->sides_supported = PAPPL_SIDES_ONE_SIDED | PAPPL_SIDES_TWO_SIDED_LONG_EDGE | PAPPL_SIDES_TWO_SIDED_SHORT_EDGE;
  driver_data->sides_default   = PAPPL_SIDES_ONE_SIDED;

  driver_data->num_source = 7;
  driver_data->source[0]  = "default";
  driver_data->source[1]  = "tray-1";
  driver_data->source[2]  = "tray-2";
  driver_data->source[3]  = "tray-3";
  driver_data->source[4]  = "tray-4";
  driver_data->source[5]  = "manual";
  driver_data->source[6]  = "envelope";

  driver_data->left_right = 635;       // 1/4" left and right
  driver_data->bottom_top = 1270;      // 1/2" top and bottom

  for (i = 0; i < driver_data->num_source; i ++)
  {
    if (strcmp(driver_data->source[i], "envelope"))
      snprintf(driver_data->media_ready[i].size_name, sizeof(driver_data->media_ready[i].size_name), "na_letter_8.5x11in");
    else
      snprintf(driver_data->media_ready[i].size_name, sizeof(driver_data->media_ready[i].size_name), "env_10_4.125x9.5in");
  }
}
else
{
  papplLog(system, PAPPL_LOGLEVEL_ERROR, "No dimension information in driver name '%s'.", driver_name);
  return (false);
}
```

Finally, we fill out the ready and default media for each media source (tray):

```
// Fill out ready and default media (default == ready media from the first source)
for (i = 0; i < driver_data->num_source; i ++)
{
  pwg_media_t *pwg = pwgMediaForPWG(driver_data->media_ready[i].size_name);

  if (pwg)
  {
    driver_data->media_ready[i].bottom_margin = driver_data->bottom_top;
    driver_data->media_ready[i].left_margin   = driver_data->left_right;
    driver_data->media_ready[i].right_margin  = driver_data->left_right;
    driver_data->media_ready[i].size_width    = pwg->width;
    driver_data->media_ready[i].size_length   = pwg->length;
    driver_data->media_ready[i].top_margin    = driver_data->bottom_top;
    snprintf(driver_data->media_ready[i].source, sizeof(driver_data->media_ready[i].source), "%s", driver_data->source[i]);
    snprintf(driver_data->media_ready[i].type, sizeof(driver_data->media_ready[i].type), "%s", driver_data->type[0]);
  }
}

driver_data->media_default = driver_data->media_ready[0];

return (true);
```


The Auto-Add Callback
---------------------

The PAPPL auto-add callback is called when processing the "autoadd" sub-command.
It is called for each new device and is responsible for returning the name of
the driver to be used for the device or `NULL` if no driver is available:

```
typedef const char *(*pappl_ml_autoadd_cb_t)(const char *device_info,
    const char *device_uri, const char *device_id, void *data);
```

Our `pcl_autoadd` function uses the IEEE-1284 device ID string to determine
whether one of the drivers will work.  The [`papplDeviceParse1284ID`](@@)
function splits the string into key/value pairs that can be looked up using the
`cupsGetOption` function:

```
const char      *ret = NULL;            // Return value
int             num_did;                // Number of device ID key/value pairs
cups_option_t   *did;                   // Device ID key/value pairs
const char      *cmd,                   // Command set value
                *pcl;                   // PCL command set pointer


// Parse the IEEE-1284 device ID to see if this is a printer we support...
num_did = papplDeviceParse1284ID(device_id, &did);
```

The two keys we care about are the "COMMAND SET" (also abbreviated as "CMD") for
the list of document formats the printer supports and "MODEL"/"MDL" for the
model name.  We are looking for the "PCL" format and one of the common model
names for HP printers:

```
// Look at the COMMAND SET (CMD) key for the list of printer languages,,,
if ((cmd = cupsGetOption("COMMAND SET", num_did, did)) == NULL)
  cmd = cupsGetOption("CMD", num_did, did);

if (cmd && (pcl = strstr(cmd, "PCL")) != NULL && (pcl[3] == ',' || !pcl[3]))
{
  // Printer supports HP PCL, now look at the MODEL (MDL) string to see if
  // it is one of the HP models or a generic PCL printer...
  const char *mdl;                      // Model name string

  if ((mdl = cupsGetOption("MODEL", num_did, did)) == NULL)
    mdl = cupsGetOption("MDL", num_did, did);

  if (mdl && (strstr(mdl, "DeskJet") || strstr(mdl, "Photosmart")))
    ret = "hp_deskjet";                 // HP DeskJet/Photosmart printer
  else if (mdl && strstr(mdl, "LaserJet"))
    ret = "hp_laserjet";                // HP LaserJet printer
  else
    ret = "hp_generic";                 // Some other PCL laser printer
}

cupsFreeOptions(num_did, did);

return (ret);
```


The File Printing Callback
--------------------------

The file printing callback is used when printing a "raw" (printer-ready) file
from a client:

```
typedef bool (*pappl_pr_printfile_cb_t)(pappl_job_t *job,
    pappl_pr_options_t *options, pappl_device_t *device);
```

This callback will sometimes send some printer initialization commands followed
by the job file and then any cleanup commands.  It may also be able to count the
number of pages (impressions) in the file, although that is not a requirement.
For the HP Printer Application our `pcl_print` function just copies the file
from the job to the device and assumes that the file contains only one page:

```
int     fd;                     // Job file
ssize_t bytes;                  // Bytes read/written
char    buffer[65536];          // Read/write buffer


papplJobSetImpressions(job, 1);

fd = open(papplJobGetFilename(job), O_RDONLY);

while ((bytes = read(fd, buffer, sizeof(buffer))) > 0)
{
  if (papplDeviceWrite(device, buffer, (size_t)bytes) < 0)
  {
    papplLogJob(job, PAPPL_LOGLEVEL_ERROR,
                "Unable to send %d bytes to printer.", (int)bytes);
    close(fd);
    return (false);
  }
}

close(fd);

papplJobSetImpressionsCompleted(job, 1);

return (true);
```


The Raster Printing Callbacks
-----------------------------

The PAPPL raster printing callbacks are used for printing PWG and Apple raster
documents, JPEG and PNG images, and other formats that end up as raster data:

```
typedef bool (*pappl_pr_rstartjob_cb_t)(pappl_job_t *job,
    pappl_pr_options_t *options, pappl_device_t *device);

typedef bool (*pappl_pr_rstartpage_cb_t)(pappl_job_t *job,
    pappl_pr_options_t *options, pappl_device_t *device, unsigned page);

typedef bool (*pappl_pr_rwriteline_cb_t)(pappl_job_t *job,
    pappl_pr_options_t *options, pappl_device_t *device, unsigned y,
    const unsigned char *line);

typedef bool (*pappl_pr_rendpage_cb_t)(pappl_job_t *job,
    pappl_pr_options_t *options, pappl_device_t *device, unsigned page);

typedef bool (*pappl_pr_rendjob_cb_t)(pappl_job_t *job,
    pappl_pr_options_t *options, pappl_device_t *device);
```

Each of the raster printing callbacks is expected to send data to the printer
using the provided "device" pointer.  The "job" argument provides the current
job object and the "options" pointer provides the current
[print job options](#pappl_pr_options_s).  The "page" argument specifies the
current page number staring at `0`.

The `pappl_pr_rstartjob_cb_t` function is called at the beginning of a job to
allow the driver to initialize the printer for the current job.

The `pappl_pr_rstartpage_cb_t` function is called at the beginning of each page
to allow the driver to do any per-page initialization and/or memory allocations
and send any printer commands that are necessary to start a new page.

The `pappl_pr_rwriteline_cb_t` function is called for each raster line on the
page and is typically responsible for dithering and compressing the raster data
for the printer.

The `pappl_pr_rendpage_cb_t` function is called at the end of each page where
the driver will typically eject the current page.

The `pappl_pr_rendjob_cb_t` function is called at the end of a job to allow the
driver to send any cleanup commands to the printer.


The Identification Callback
---------------------------

The PAPPL identification callback is used to audibly or visibly identify the
printer being used:

```
typedef void (*pappl_pr_identify_cb_t)(pappl_printer_t *printer,
    pappl_identify_actions_t actions, const char *message);
```

The most common identification method is `PAPPL_IDENTIFY_ACTIONS_SOUND` which
should have the printer make a sound.  The `PAPPL_IDENTIFY_ACTIONS_DISPLAY`
and `PAPPL_IDENTIFY_ACTIONS_SPEAK` methods use the "message" argument to display
or speak a message.  Finally, the `PAPPL_IDENTIFY_ACTIONS_FLASH` method flashes
a light on the printer.

The HP Printer Application does not have an identification callback since most
PCL printers lack a buzzer or light that can be remotely activated.

> *Note:* IPP Everywhere™ requires all printers to support identification.
> A warning message is logged by PAPPL whenever a driver does not set the
> `identify_cb` member of the printer driver data structure.


The Status Callback
-------------------

The PAPPL status callback is used to update the printer state, supply levels,
and/or ready media for the printer:

```
typedef bool (*pappl_pr_status_cb_t)(pappl_printer_t *printer);
```

The callback can open a connection to the printer using the
[`papplPrinterOpenDevice`](@@) function.


The Self-Test Page Callback
---------------------------

The PAPPL self-test page callback is used to generate a self-test page for the
printer:

```
typedef const char *(*pappl_printer_testpage_cb_t)(pappl_printer_t *printer,
    char *buffer, size_t bufsize);
```

When the callback returns a filename (copied to the specified buffer), that
file will be queued as a job for the printer.  The callback can also try opening
the device using the [`papplPrinterOpenDevice`](@@) function to send a printer
self-test command instead - in this case the callback must return `NULL` to
indicate there is no file to be printed.


Writing Printer Drivers
=======================

