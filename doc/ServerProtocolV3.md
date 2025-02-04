# Omaha Client-Server Protocol V3 #

This document describes version 3 of the Omaha client-server protocol.  Omaha launched on Windows with this version of the protocol in May 2011.  (Version 2 of the protocol launched in May 2007 on Windows and May 2008 on Mac; Version 1 of the protocol was never deployed publicly.)

Version 2 is documented [here](ServerProtocolV2.md). An older description of the V3 protocol is [here](ServerProtocol.md). Version 3.1 is documented [here](https://chromium.googlesource.com/chromium/src.git/+/master/docs/updater/protocol_3_1.md).

## Introduction ##
The Omaha protocol is designed to facilitate the acquisition, delivery, and metrics of software updates over the Internet. It is an application-layer protocol on top of HTTP.

The client sends requests via HTTP POST with an XML data body. The response is an XML data body. For Omaha Client, the request and response are secured by [CUP](ClientUpdateProtocol.md) or SSL.

Diagram of an example request-response pair:

![Protocol V3 Illustration](https://cdn.rawgit.com/google/omaha/master/doc/protocol_v3.svg)

## Terminology ##
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

The following terms also have specific meaning in this document:
  * **Client**: A host with updateable software, seeking knowledge about updates. The client sends **requests** to the server.
  * **Server**: A host reachable over the Internet that has knowledge about updates. The server sends **responses** to requests.
  * **Omaha Client**: A specific instantiation of a compatible client - this project's client.
  * **Update Flow**: A sequence of update check, update attempt, and event ping (if an update was available), or simply an update check (if no update was available). See [#Update Flow](#update-flow) for more details.
  * **Product**: A piece of software that the client is responsible for keeping up to date. The client itself may be a product.

## Update Flow ##
A complete update sequence is formed by the following steps:
  1. The client sends a request that includes information about each updateable piece of software (an "app"), asking for information about whether an update is available for any of them. This is called an "update check".
  1. The server responds for each app either that an update is available, or that there is no update available, or that an error was encountered.
  1. If there are any available updates, they are downloaded and applied by the client.
  1. If there were any available updates, the client sends a request including information about the outcome of the update back to the server. This is called an "event ping".
  1. If an event ping was sent, the server responds, acknowledging receipt of the event ping.

Both request and response data are XML, carried in the body of an HTTP Post request and response.

If any attribute specified by this protocol is missing in an element, the specified default value is assumed. Compatible clients and servers MAY decide not to implement or transmit any attributes that have a defined default. Compatible clients and servers SHOULD assume the defined default value if the attribute is missing.

Compatible clients and servers MUST be able to tolerate unexpected elements. Compatible clients and servers MUST be able to tolerate unexpected attributes.

For each app in a request, there are two types of messages: an update check or an event ping. If an `<app>` contains an `<updatecheck>`, the message is an update check for that app. If an `<app>` contains an `<event>`, the message is an event ping for that app. A client MUST NOT combine the two types of messages for the same `<app>`.

## Version Numbers ##
The Omaha protocol handles the transmission of several version numbers. Servers SHOULD publicize the versioning schemes that they understand, and clients interacting with those servers SHOULD use a published versioning scheme.

Compatible clients and servers SHOULD implement at minimum the following versioning scheme: A version is a 4-tuple encoded in dot-decimal notation, e.g. `1.0.66.44`. An element (and the preceding separating dot) may be omitted if that element and every following element is zero. For example, `1.2` is equivalent to `1.2.0.0`. Leading zeroes are ignored in each element - so `1.001` and `1.1` are both equivalent to `1.1.0.0`, and `1.005` is a higher version than `1.4`.

Versions are members of an ordered set. A version `A` is greater than a version `B` if and only if there is at least one element in `A` that is greater than the corresponding element in `B`, and all elements preceding that element in `A` are equal to their corresponding elements in `B`. Two versions are equal if and only if all their elements are equal.

## GUIDs ##
The Omaha protocol deals with globally-unique identifiers in multiple places. For the purpose of the protocol, a GUID has one of the following formats:
  1. a 128-bit value, serialized as a string of hexadecimal digits as follows: "`{00000000-1111-2222-3333-444444444444}`" (e.g. "`{430FD4D0-B729-4F61-AA34-91526481799D}`").
  1. A lowercase Mac bundle ID (for example `com.google.chrome`). This format is only allowed to identify a product.

## Counting Algorithms ##
One of the goals of the Omaha protocol is to provide the server with information about the success and failure rates of updates, as well as access to other metrics such as retention and usage, without compromising the client's privacy.

The core aspect of counting unique users is being able to de-duplicate requests from the same user. The protocol supports three different de-duplication schemes, described briefly here:

### User-ID Counting ###
In User-ID counting, the server MAY de-duplicate metrics by discarding multiple requests with the same User-ID.

However, as user IDs can be used to track users across days, Omaha Client only supports them on an opt-in basis.

### Client-Regulated Counting (Days-Based) ###
In client-regulated counting, the client transmits with each request the number of days since the previous time it sent a message. If the timestamp of the request minus the number of days since the previous message is within the server's counting window, the request can be discarded as a duplicate.

#### Unreliable Client Storage ####
A major drawback to client-regulated counting is that it depends upon the client's ability to reliably track information about what transactions the client has made with the server. This ability is significantly degraded in environments where the client is frequently re-imaged to some known state. For the server to detect such re-imagings, the client should store a random value alongside the client-regulated counting data and transmit this to the server. To protect the privacy of the client, the random value should be rotated on every successfully-parsed update check response (even if the response indicates no update or an error). The server interprets duplicate values as indicative of the machine's state being reset.

### Client-Regulated Counting (Date-Based) ###
In date-based client-regulated counting, the server transmits with each response an authoritative date, representing the date it received the request. The client stores this date, and then with the next request echoes it back to the server. If the date transmitted with a request is within the server's counting window, the request can be discarded as a duplicate. Like days-based client-regulated counting, this algorithm depends upon the client to track data across machine restarts, and therefore also requires a ping_freshness value.

## Packages & Fingerprints ##
An individual version of an app may have multiple files (some of which may be optional). Such files are called "Packages". Every app has at least one package: that package is the file that is downloaded (and run by Omaha Client) to conduct the update. Product installation versions are identified by a version number, but individual package binary versions are identified by a "fingerprint". A fingerprint is a string of the form A.B, where A is a fingerprint type, and B is fingerprint data. The two currently supported fingerprint types are:
  * "1", indicating that the fingerprint data should be interpreted as a SHA256 hash of the binary, represented as a lowercase hexadecimal string. Example: "1.cf937d3ba4d4a7bcc68dba806d16752e12ef7f3f96e964385a9db0f28da19e92"
  * "2", indicating that the fingerprint data should be interpreted as a dotted-quad version number. This is primarily for legacy purposes: format "1" is preferred. Example: "2.35.1.119.154"

## HTTP Post Body ##

### Request ###
An Omaha V3 request MUST contain exactly one `<request>` element at the root level.

---

#### `<request>` ####

##### Attributes #####
  * `acceptformat`: A comma-separated list of strings describing the formats of update payloads that this client accepts. As a special case, the empty string indicates undefined restrictions on the format. Default: "". The following values are supported:
    * `crx2`: The CRX file format, version 2.
    * `crx3`: The CRX file format, version 3.
  * `dedup`: Specifies the preferred de-duplication algorithm for this request. Either "" (unknown or no-preference), "cr" (client-regulated) or "uid" (user-id). Default: "". Omaha Client sends "cr" in all cases.
  * `dlpref`: Specifies the preferred download URL behavior. A comma-separated list of values. The first value is the highest priority, further values reflect secondary, tertiary, et cetera priorities. Legal values are "" (in which case the entire list must be empty, indicating unknown or no-preference) or "cacheable" (the server should prioritize sending URLs that are easily cacheable). Default: "".
  * `installsource`: A string specifying the cause of the update flow. For example: "ondemand", or "scheduledtask". Default: "".
  * `ismachine`: "1" if the client is known to be installed with system-level or administrator privileges. "0" otherwise. Default: "0".
  * `originurl`: If the update flow is invoked from a web page, contains the URL of that page. Otherwise, "". Default: "".
  * `protocol`: The version of the Omaha protocol. Compatible clients MUST provide a value of "3.0". Default: Undefined - compatible clients MUST always transmit this attribute.
  * `requestid`: A randomly-generated (uniformly distributed) GUID. Each request attempt SHOULD have (with high probability) a unique `requestid`. Default: "".
  * `sessionid`: A randomly-generated (uniformly distributed) GUID. Each single update flow (an update check, update application, event ping sequence) SHOULD have (with high probability) a single unique `sessionid`. Default: "".
  * `testsource`: Either "", "dev", "qa", "prober", "auto", or "ossdev". Any value except "" indicates that the request is a test and should not be counted toward normal metrics. Default: "".
  * `updater`: A string identifying the updater software itself (e.g. "Omaha", "Keystone", "Chrome"). Default: "0.0.0.0".
  * `updaterchannel`: If present, identifies the distribution channel of the client (e.g. "stable", "beta", "dev", "canary"). Default: "".
  * `updaterversion`: The version of the updater itself (the entity sending this request). Default: "0.0.0.0".
  * `userid`: A randomly-generated (uniformly distributed) GUID. Each instance of the client SHOULD have (with high probability) either a single unique `userid`, or no `userid` at all (""). Default: "". Omaha Client transmits `userid` only for opt-in users.
  * `version`: *DEPRECATED - use `updater` and `updaterversion` instead* The ID and version number of the client. Default: Undefined - compatible clients MUST always transmit this attribute. The version number MUST be one of the two following forms:
    1. "`A-V`" where `A` is a client identifier, and `V` is the version number of client (e.g. "chromiumcrx-31.1.0.112").
    1. "`V`" where `V` is the version number of the client (e.g. "1.3.23.9"). Compatible clients SHOULD NOT use this form, as it is reserved for Omaha Client.

##### Legal Child Elements #####
  * Any number of `<app>`
  * At most 1 `<hw>`
  * At most 1 `<os>`


---

#### `<hw>` ####
Contains information about the capabilities of the client's hardware.
##### Attributes #####
  * `sse`: "1" if the client's hardware supports the SSE instruction set. "0" if the client's hardware does not. "-1" if unknown. Default: "-1".
  * `sse2`: "1" if the client's hardware supports the SSE2 instruction set. "0" if the client's hardware does not. "-1" if unknown. Default: "-1".
  * `sse3`: "1" if the client's hardware supports the SSE3 instruction set. "0" if the client's hardware does not. "-1" if unknown. Default: "-1".
  * `sse41`: "1" if the client's hardware supports the SSE4.1 instruction set. "0" if the client's hardware does not. "-1" if unknown. Default: "-1".
  * `sse42`: "1" if the client's hardware supports the SSE4.2 instruction set. "0" if the client's hardware does not. "-1" if unknown. Default: "-1".
  * `ssse3`: "1" if the client's hardware supports the SSSE3 instruction set. "0" if the client's hardware does not. "-1" if unknown. Default: "-1".
  * `avx`: "1" if the client's hardware supports the AVX instruction set. "0" if the client's hardware does not. "-1" if unknown. Default: "-1".
  * `physmemory`: The physical memory the client has available to it, measured in gibibytes, truncated down to the nearest gibibyte, or "-1" if unknown. This value is intended to reflect the maximum theoretical storage capacity of the client, not including any hard drive or paging to a hard drive or peripheral. Default: "-1".

##### Legal Child Elements #####
None.


---

#### `<os>` ####
Contains information about the operating system that the client is running under.
##### Attributes #####
  * `platform`: The operating system family that the client is running within (e.g. "win", "mac", "linux", "ios", "android"), or "" if unknown. The operating system name should be transmitted in lowercase with minimal formatting. Default: "".
  * `version`: The primary version of the operating system, or "" if unknown. Default: "".
    * On Windows, the major-minor OS version (e.g. "5.1", "6.2").
    * On Mac OSX, "MacOSX".
    * On iOS, the version number (e.g. "4.3", "5.1.1").
    * On Android, the version number (e.g. "4.1.1", "4.3").
  * `sp`: The secondary version of the operating system, or "" if unknown. Default: "".
    * On Windows, the service pack (e.g. "Service Pack 2").
    * On Mac OSX, the OS version and processor architecture (e.g. "10.5.6\_i486").
  * `arch`: The architecture of the operating system (e.g. "x86", "x64", "arm"), or "" if unknown. Default: "".

##### Legal Child Elements #####
None.


---

#### `<app>` (Request) ####
Each product that is contained in the request is represented by exactly one `<app>` tag.
##### Attributes #####
  * `appid`: The GUID that identifies the product. See [#GUIDs](#guids). Default: Undefined - Compatible clients MUST transmit this attribute.
  * `version`: The version of the product install. See [#Version\_Numbers](#version-numbers). Default: "0.0.0.0".
  * `lang`: The language of the product install, in BCP 47 representation. Default: "".
  * `brand`: The brand code that the product was installed under, if any. A brand code is a short (4-character) string used to identify installations that took place as a result of partner deals or website promotions. Default: "".
  * `client`: A generalized form of brand code that can accept a wider range of values but is used for similar purposes to `brand`. Default: "".
  * `enabled`: Tracks whether the app is enabled on the client. Apps may be disabled for a variety of reasons. A value of "-1" indicates that the enabled status is unknown. "0" indicates that the app is disabled. "1" indicates that the app is enabled. Default: "-1"
  * `experiments`: A key/value list of experiment identifiers. Experiment labels are used to track membership in different experimental groups, and may be set at install or update time. The experiments string is formatted as a semicolon-delimited concatenation of experiment label strings. An experiment label string is an experiment name, followed by a '=' character, followed by an experimental label value. For example: "crdiff=got\_bsdiff;optimized=O3". The client SHOULD NOT transmit the expiration date of any experiments it has, even if the server previously specified a specific expiration date. Default: "".
  * `iid`: A GUID that identifies an installation flow. For example, each download of a product installer is tagged with a unique GUID. Attempts to install using that installer can then be grouped. A client SHOULD NOT persist the iid GUID after the installation flow of a product is complete.
  * `installage`: The number of PST8PDT calendar days since the app was first installed. The first communication to the server should use a special value of "-1". Compatible clients MAY fuzz this value to the week granularity (e.g. send "0" for 0 through 6, "7" for 7 through 13, etc). Default: "0"
  * `installdate`: The date-based counting equivalent of installage: this is a numeric calendar day that the app was installed on. (This value is provided by the server in the response to the first request in the installation flow. See [#Client-Regulated\_Counting\_(Date-Based)](#client-regulated-counting-date-based)). The client MAY fuzz this value to the week granularity (e.g. send "0" for 0 through 6, "7" for 7 through 13, etc). The first communication to the server should use a special value of "-1". A value of "-2" indicates that this value is not known. Default: "-2".
  * `installsource`: A string indicating the cause of this install or update flow. As examples:  "organic" indicating an organic web download, "scheduler" indicating a scheduled update, "ondemand" indicating a user-prompted update. Default: "".
  * `ismachine`: "0" if the app is installed at the user (non-privileged) level; "1" if the app is installed at the system (privileged) level; "-1" if unknown. Default: "-1"
  * `tag`: A field for a client to transmit arbitrary update parameters in string form. Compatible clients and servers MAY use this attribute to negotiate special update rules. Alternatively, they MAY extend the protocol to represent the information more clearly in another parameter. As an example, Omaha Client uses this field to transmit whether a Google Chrome installation is on the "stable", "dev", or "beta" channel, which affects how the server issues update responses for that installation. Default: "".
  * `fp`: If there is only one package, the fingerprint for that package may be transmitted at the `<app>` level. See [#Packages\_&\_Fingerprints](#packages--fingerprints).  Default: "".
  * `cohort`: A machine-readable string identifying the release cohort (channel) that the app belongs to. Limited to ASCII characters 32 to 126 (inclusive) and a maximum length of 1024 characters. Default: "".
  * `cohorthint`: An machine-readable enum indicating that the client has a desire to switch to a different release cohort. The exact legal values are app-specific and should be shared between the server and app implementations. Limited to ASCII characters 32 to 126 (inclusive) and a maximum length of 1024 characters. Default: "".
  * `cohortname`: A stable non-localized human-readable enum indicating which (if any) set of messages the app should display to the user. For example, an app with a cohortname of "beta" might display beta-specific branding to the user. Limited to ASCII characters 32 to 127 (inclusive) and a maximum length of 1024 characters. Default: "".
  * `release_channel`: A string indicating to the server which release channel this installation of the application should receive future updates from. Examples include "stable", "beta", "dev". An empty string indicates no preference. Default: "".

##### Legal Child Elements #####
  * Any number of `<data>`.
  * Any number of `<disabled>`.
  * At most one `<packages>`.
  * At most one `<ping>`.
  * At most one of the following:
    * One or more `<event>`.
    * Exactly one `<updatecheck>`.


---

#### `<data>` (Request) ####
Each `<data>` tag in the request represents either a request for additional textual information from the server, or provides additional textual information to the server.
##### Attributes #####
  * `name`: Indicates the type of data request this is. Legal values are "install", "untrusted", or "". Additions to the protocol must be alphanumeric (that is, they must match [a-zA-Z0-9]+). Default: "".
  * `index`: If `name` is "install", the numeric index of the requested installation data blob. Otherwise, undefined. Default: "0".

##### Legal Child Elements #####
  * May contain arbitrary textual information. Compatible clients and compatible servers SHOULD perform sanitization of this data both when it is sent and received. In practice, this data is frequently supplied by untrusted third parties.


---

#### `<disabled>` (Request) ####
##### Attributes #####
  * `reason`: an integral reason that the app is disabled. This protocol does not require or suggest a meaning for the values, except for "0", which indicates the lack of a reason. Default: "0".

##### Legal Child Elements #####
None.


---

#### `<packages>` (Request) ####
A `<packages>` tag simply contains several `<package>`s.
##### Attributes #####
None.
##### Legal Child Elements #####
  * At least one `<package>`


---

#### `<package>` (Request) ####
A `<package>` tag gives information about an installed package.
##### Attributes #####
  * `fp`: The fingerprint identifying the installed package. See [#Packages\_&\_Fingerprints](#pacakges--fingerprints). Default: "".

##### Legal Child Elements #####
None.


---

#### `<ping>` (Request) ####
Any `<ping>`s contained in a request are used to count active users and potentially deduplicate requests from the same client. See [#Counting\_Algorithms](#counting-algorithms).

A request containing any `<ping>` is called a "ping". Typically, pings are combined with update checks into a single request.
A request containing a `<ping>` with the `active="1"`, `a`, or `ad` attributes explicitly transmitted is further called an "active ping".

New clients are recommended to use the `ad` and `rd` attributes of the `<ping>`, and ignore the others. Existing clients should consider transitioning, as date-based counting does not rely on unreliable client clocks.
##### Attributes #####
  * `active`: "1" if the app was active since the previous request that contained a `<ping>`. Otherwise, "0". If `a` or `ad` is explicitly transmitted, `active` may be omitted. Default: "0".
  * `a`: If transmitted, the app was active since the request that contained a `<ping>`. In this case, the value is the number of integral 24-hour periods that have elapsed since the start of the America/Los\_Angeles calendar day that the previous active ping was sent on. See [#Client-Regulated\_Counting\_(Days-Based)](#client-regulated-Counting-days-based). A value of "-1" signifies that there was no previous active ping.
  * `r`: The number of integral 24-hour periods that have elapsed since the start of the America/Los\_Angeles calendar day that the previous ping was sent on. See [#Client-Regulated\_Counting\_(Days-Based)](#client-regulated-counting-days-based). A value of "-1" signifies that there was no previous active ping. Default: "0".
  * `ad`: The value of the `elapsed_days` attribute of the `<daystart>` element in the server's reply to the previous active ping. See [#Client-Regulated\_Counting\_(Date-Based)](#client-regulated-counting-date-based). A value of "-1" signifies that there was no such previous request. A value of "-2" signifies that the value is not known. Default: "-2".
  * `rd`: The value of the `elapsed_days` attribute of the `<daystart>` element in the server's reply to the previous ping. See [#Client-Regulated\_Counting\_(Date-Based)](#client-regulated-counting-date-based). A value of "-1" signifies that there was no such previous request. A value of "-2" signifies that the value is not known. Default: "-2".
  * `ping_freshness`: A random 128-bit number. See [#Unreliable\_Client\_Storage](#unreliable-client-storage) The client SHOULD store the per-product value alongside whatever data it uses to track `ad`, `rd`, `a`, and `r`. The client MUST rotate the value to a new random 128-bit number whenever the data used to track `ad`, `rd`, `a`, or `r` is updated. The server MAY interpret duplicate `pingfresh` values as indicating that the client has been re-imaged to a previous state. A value of "" signifies that no value was available. Default: "".

##### Legal Child Elements #####
None.


---

#### `<event>` (Request) ####
Throughout and at the end of an update flow, the client MAY send event reports by sending one or more requests containing an `<event>`.

`<event>`s should never appear in the same request as an `<updatecheck>`.
##### Attributes #####
  * `eventtype`: A special value indicating the type of the event. The following values are defined, all others are reserved. There is no default, the eventtype must always be specified.
    * `0`: unknown
    * `1`: download complete
    * `2`: install complete (for the initial installation of the app)
    * `3`: update complete (for an upgrade in the version of the app)
    * `4`: uninstall complete
    * `5`: download started (for the initial installation of the app)
    * `6`: install started
    * `9`: new application install started
    * `10`: setup started
    * `11`: setup finished
    * `12`: update started
    * `13`: update download started
    * `14`: update download complete
    * `15`: update install started
    * `16`: setup update begin
    * `17`: setup update complete
    * `20`: register product complete
    * `30`: OEM install first check
    * `40`: app-specific command started
    * `41`: app-specific command ended
    * `50`: update-check failure (to avoid doubling server load during an outage, this event should only be transmitted a small fraction of the time the client encounters a failure on the update check)
    * `51`: free. Different implementors of the protocol may define a custom meaning.
    * `52`: free. Different implementors of the protocol may define a custom meaning.
    * `53`: free. Different implementors of the protocol may define a custom meaning. (Only known use is by the Chrome Recovery Component.)
    * `54`: free. Different implementors of the protocol may define a custom meaning. (Only known use is by ChromiumOS.)
    * `55`: free. Different implementors of the protocol may define a custom meaning.
    * `100`: setup failure
    * `102`: COM server failure
    * `103`: setup update failure
  * `eventresult`: The result of the event. The following values are defined, all others are reserved. Default: "0".
    * `0`: error
    * `1`: success
    * `2`: success, but a system restart is required
    * `3`: success, but a browser restart is required
    * `4`: cancelled
    * `5`: error in installer MSI
    * `6`: error in installer (non-MSI)
    * `7`: server instructed "no-update"
    * `8`: error in installer (system)
    * `9`: update deferred (another, higher-priority update will be doen first, and launch a new update flow - this should be sent if the updater, for example, must first update itself.)
    * `10`: error during handoff to existing updater
  * `errorcode`: The error code (if any) of the operation, encoded as a signed base-10 integer. Default: "0".
  * `extracode1`: Additional numeric information about the operation's result, encoded as a signed base-10 integer. Default: "0".
  * `errorcat`: An error category, for use in distinguishing between different classes of error codes, encoded as a signed base-10 integer. Default: "0".
  * `download_time_ms`: For events representing a download, the time elapsed between the start of the download and the end of the download, in milliseconds. For events representing an entire update flow, the sum of all such download times over the course of the update flow. Default: "0". Sent in `<event>`s that have an `eventtype` of "1", "2", "3", and "14" only.
  * `downloaded`: For events representing a download, the number of bytes successfully downloaded. For events representing an entire update flow, the sum of all such successfully downloaded bytes over the course of the update flow. Default: "0". Sent in `<event>`s that have an `eventtype` of "1", "2", "3", and "14" only.
  * `downloader`: A string identifying the download algorithm / stack. Example values include: "bits", "direct", "winhttp", "p2p". Default: "". Sent in `<event>`s that have an `eventtype` of "14" only.
  * `total`: For events representing a download, the number of bytes expected to be downloaded. For events representing an entire update flow, the sum of all such expeccted bytes over the course of the update flow. Default: "0".
  * `update_check_time_ms`: For events representing an entire update flow, the time elapsed between the start of the update check and the end of the update check, in milliseconds. Default: "0". Sent in `<event>`s that have an `eventtype` of "2" and "3" only.
  * `install_time_ms`: For events representing an install, the time elapsed between the start of the install and the end of the install, in milliseconds. For events representing an entire update flow, the sum of all such durations. Default: "0". Sent in `<event>`s that have an `eventtype` of "2" and "3" only.
  * `source_url_index`: For events representing a download, the position of the download URL in the list of URLs supplied by the server in a `<urls>` tag.
  * `state_cancelled`: The state of the system at the time that a user cancels the update/install. The following values are defined, all others are reserved. Default: "0".
    * `0`: unknown or not-cancelled
    * `1`: initializing
    * `2`: waiting to check for update
    * `3`: checking for update
    * `4`: update available
    * `5`: waiting to download
    * `6`: retrying download
    * `7`: downloading
    * `8`: download complete
    * `9`: extracting
    * `10`: applying differential patch
    * `11`: ready to install
    * `12`: waiting to install
    * `13`: installing
    * `14`: install complete
    * `15`: paused
    * `16`: no update
    * `17`: error
  * `time_since_update_available_ms`: The number of milliseconds that elapsed from when the update was known to be available to when user cancelled the action. "-2" indicates that there was no cancellation. Default: "-2" Sent in `<event>`s that have an `eventtype` of "2" and "3" only.
  * `time_since_download_start_ms`: The number of milliseconds that elapsed from when the download was begun to when user cancelled the action. "-2" indicates that there was no cancellation. Default: "-2" Sent in `<event>` that have an `eventtype`s of "2" and "3" only.
  * `url`: The URL from which the download was attempted. Default: "".  Sent in `<event>`s that have an `eventtype` of "14" only.
  * `nextversion`: The version of the app that the update flow to which this event belongs attempted to reach, regardless of success or failure of the update operation. See [#Version\_Numbers](#version-numbers). Default: "0.0.0.0".
  * `previousversion`: The version of the app that was present on the machine at the time of the update-check of this update flow, regardless of success or failure of the update operation. See [#Version\_Numbers](#version-numbers). Default: "0.0.0.0".
  * `nextfp`: If the update flow containing this event contained only a single package, the fingerprint that package attempted to reach, regardless of success or failure of the update operation. See [#Packages\_&\_Fingerprints](#packages--fingerprints). Default: "".
  * `previousfp`: If the update flow containing this event contained only a single package, the fingerprint that package had at the time of the update check, regardless of success or failure of the update operation. See [#Packages\_&\_Fingerprints](#packages--fingerprints). Default: "".

##### Legal Child Elements #####
None.


---

#### `<updatecheck>` (Request) ####
##### Attributes #####
  * `tttoken`: An opaque access token that can be used to identify the requesting client as a member of a trusted-tester group. If non-empty, the request SHOULD be sent over SSL or another. Default: "".
  * `updatedisabled`: An indication of whether the client will honor an update response, if it recieves one. Legal values are "true" (indicating that the client will ignore any update instruction) and "false" (indicating that the client will attempt an update if one is instructed). Default: "false".
  * `targetversionprefix`: A component-wise prefix of a version number, or a complete version number suffixed with the `$` character. The server SHOULD NOT return an update instruction to a version number that does not match the prefix or complete version number. The prefix is interpreted a dotted-tuple that specifies the exactly-matching elements; it is not a lexical prefix. (For example, "1.2.3" MUST match "1.2.3.4" but MUST NOT match "1.2.34".) Default: "".

##### Legal Child Elements #####
None.


---

### Response ###

---

#### `<response>` ####
##### Attributes #####
  * `protocol`: The version of the Omaha protocol. Compatible clients MUST provide a value of "3.0". Default: Undefined - compatible servers MUST always transmit this attribute.
  * `server`: A string identifying the server or server family for diagnostic purposes. As examples, "production", "test". Default: "".

##### Legal Child Elements #####
  * At most one `<daystart>`.
  * Any number of `<app>`.


---

#### `<daystart>` ####
##### Attributes #####
  * `elapsed_seconds`: The number of seconds since the most recent midnight of the server's locale, at the time the request was received. Default: Undefined - compatible servers MUST always transmit this attribute.
  * `elapsed_days`: The number of calendar days that have elapsed since January 1st, 2007 in the server's locale, at the time the request was received.

##### Legal Child Elements #####
None.


---

#### `<app>` (Response) ####
Each product that is contained in the response is represented by exactly one `<app>` tag. If a product appears in the request, it MUST be contained in the response. The response MAY contain additional products. If it does, the client MAY install or update these products in addition to the requested ones. This mechanism supports the installation of dependencies that are distributed as separate physical products.
##### Attributes #####
  * `appid`: The GUID that identifies the product. See [#GUIDs](#guids). Default: Undefined - Compatible clients MUST transmit this attribute.
  * `status`: The state of the product on the server. The following values are defined, all others are reserved. Default: "0".
    * `ok`: The product is recognized.
    * `restricted`: The product is recognized, but due to policy restrictions (such as export law compliance) the server must refuse to give a meaningful response.
    * `error-unknownApplication`: The server is not aware of this product.
    * `error-invalidAppId`: The server could not parse the product's GUID.
  * `experiments`: A key/value list of experiment identifiers that the client should store its membership in and report to the server in future requests. Experiment labels are used to track membership in different experimental groups, and may be set at install or update time. The experiments string is formatted as a semicolon-delimited concatenation of experiment label strings. An experiment label string is an experiment name, followed by a '=' character, followed by an experimental label value, followed by a '|' character, followed by a full date-time of the format "Thu, 19 Nov 2015 15:46:57 -0700" (RFC 2822 compatible). Past the given date-time, the client SHOULD cease reporting membership in the experiment group. The client SHOULD NOT report the date-time in future requests to the server. For example: "crdiff=got\_bsdiff|Thu, 19 Nov 2015 15:46:57 -0700;optimized=O3|Wed, 18 Nov 2015 02:00:00 -0300". Default: "".
  * `cohort`: See cohort in the `<request>`. If and only if this attribute is transmitted in the response (even if the value is empty-string), the client should overwrite the current cohort of this app with the sent value. Limited to ASCII characters 32 to 127 (inclusive) and a maximum length of 1024 characters. No default value: the lack of a transmitted value has a different meaning than any transmitted value.
  * `cohorthint`: See cohorthint in the `<request>`. If and only if sent (even if the value is empty-string), the client should overwrite the current cohorthint of this app with the sent value. Limited to ASCII characters 32 to 127 (inclusive) and a maximum length of 1024 characters. No default value: the lack of a transmitted value has a different meaning than any transmitted value.
  * `cohortname`: See cohortname in the `<request>`. If and only if sent (even if the value is empty-string), the client should overwrite the current cohortname of this app with the sent value. Limited to ASCII characters 32 to 127 (inclusive) and a maximum length of 1024 characters. No default value: the lack of a transmitted value has a different meaning than any transmitted value.

##### Legal Child Elements #####
  * At most one `<ping>`.
  * Any number of `<data>`.
  * At most one of:
    * Any number of `<event>`.
    * Exactly one `<updatecheck>`.
  * Any number of `<unknown>`.


---

#### `<ping>` (Response) ####
##### Attributes #####
  * `status`: Indicates whether the ping was recognized by the server. Legal values: "ok". No default value: compatible servers MUST always transmit this attribute.

##### Legal Child Elements #####
None.


---

#### `<data>` (Response) ####
Each `<data>` tag in the response represents an answer to a data request from the client. The textual contents of this item MUST be sanitized by the server to prevent the injection of arbitrary XML.
##### Attributes #####
  * `name`: A repeat of the value of the `<data name>` from the request to which this tag is responding. Default: "".
  * `index`: A repeat of the value of `<data index>` from the request, if present, to which this tag is responding. Default: "0".
  * `status`: The type of the data response. The following values are defined, all others are reserved. Default: "0".
    * `ok`: This tag contains the appropriate data response, even if such a response is the empty string.
    * `error-invalidargs`: The data request could not be parsed or understood.
    * `error-nodata`: The data request was understood, but the server does not know the data for the requested entry. This is a distinct state from knowing that the correct response is zero-length data.

##### Legal Child Elements #####
  * May contain arbitrary non-XML textual information.


---

#### `<event>` (Response) ####
##### Attributes #####
  * `status`: Indicates whether the event was recognized by the server. Legal values: "ok". No default value: compatible servers MUST always transmit this attribute.

##### Legal Child Elements #####
None.


---

#### `<updatecheck>` (Response) ####
##### Attributes #####
  * `status`: Indicates the outcome of the updatecheck. No default value: compatible servers MUST always transmit this attribute.
    * `ok`: An update is available and should be applied. In this case, this tag will contain children that further detail the update.
    * `noupdate`: No update is available for this client at this time.
    * `error-internal`: The server encountered an unspecified internal error.
    * `error-hash`: The server attempted to serve an update, but could not provide a valid hash for the download.
    * `error-osnotsupported`: The server recognized the product, but the product is not available for the client's operating system.
    * `error-hwnotsupported`: The server recognized the product, but the product is not available for the client's hardware  
    configuration.
    * `error-unsupportedProtocol`: The server recognized the product, but is incapable of delivering a response in this version of the protocol.
  * `info`: gives more infomation about the status response.

##### Legal Child Elements #####
  * At most one `<urls>`.
  * At most one `<manifest>`.


---

#### `<urls>` (Response) ####
##### Attributes #####


##### Legal Child Elements #####
  * At least one `<url>`.


---

#### `<url>` (Response) ####
##### Attributes #####
  * `codebase`:
  * `codebasediff`:

##### Legal Child Elements #####
  * None.


---

#### `<manifest>` (Response) ####
  * `version`:

##### Attributes #####


##### Legal Child Elements #####
  * Exactly one `<packages>`.
  * Exactly one `<actions>`.


---

#### `<packages>` (Response) ####
##### Attributes #####
None.

##### Legal Child Elements #####
  * At least one `<package>`.


---

#### `<package>` (Response) ####
##### Attributes #####
  * `name`:
  * `namediff`:
  * `required`:
  * `size`:
  * `sizediff`:
  * `hash`:
  * `hashdiff`:
  * `hash_sha256`:
  * `hashdiff_sha256`:
  * `fp`:

##### Legal Child Elements #####
None.


---

#### `<actions>` (Response) ####
##### Attributes #####
None.

##### Legal Child Elements #####
  * At least one `<action>`.


---

#### `<action>` (Response) ####
##### Attributes #####
  * `event`:
  * `run`:
  * `arguments`:
  * `successurl`:
  * `terminateallbrowsers`:
  * `successaction`:

##### Legal Child Elements #####
None.


---

#### `<unknown>` (Response) ####
##### Attributes #####
  * `status`:

##### Legal Child Elements #####
None.


---

## HTTP Headers ##
### Request Headers ###
  * `X-Goog-Update-Interactivity` -- Either 'fg' or 'bg' when present, 'fg' indicating a user-initiated foreground update. 'bg' indicates that the request is part of a background update. If the server is under extremely high load, it may use this to prioritize 'fg' requests over 'bg' requests.
  * `X-Goog-Update-AppId`  -- Indicates the apps associated with the request. Clients updating multiple apps should specify a comma-separated list of values.
  * `X-Goog-Update-Updater` -- Indicates the identity of the updater. This is the "updater" version string also present in the request; in the case of Omaha, prepend "Omaha-" to the version string.
  
### Response Headers ###
  * `X-Retry-After` -- When present, a positive integral number of seconds during which the client MUST NOT contact the server again for background updates, including but not limited to retried attempts to connect to the server due to an unparseable response or apparent error. Clients MUST respect this header even if paired with non-successful HTTP response code. Servers SHOULD NOT send a value in excess of 86400 (24 hours), and clients SHOULD treat values greater than 86400 as 86400.

### Additional Headers ###
Compatible clients MAY include additional headers in requests to the server. Such headers are purely advisory in nature, and their presence and content MUST NOT be required in order to provide responses to update checks. Compatible servers MUST be able to tolerate unexpected headers.

Compatible servers MAY include additional headers in responses to the client. Compatible clients MUST be able to tolerate unexpected headers.

Omaha Client uses additional headers in the request and response to implement [CUP](cup.html), as well as send network diagnostics. Omaha Client sends the following custom headers:
  * `X-Last-HR`: On a retry, contains the HRESULT error code returned from `NetworkRequestImple::DoSendHttpRequestion()` for the previous attempt.
  * `X-Last-HTTP-Status-Code`: On a retry, contains the HTTP status code of the previous attempt, if any HTTP status code was received.
  * `X-Proxy-Manual-Auth`: When present, this header indicates that the request was sent through at least one proxy that required authentication and that Omaha had no cached credentials for the proxy, causing it to display a UI to prompt the user. The header's value, when present, is always `1`.
  * `X-Proxy-Retry-Count`: On a retry, contains the number of times an HTTP 407 status code was received.
  * `X-Retry-Count`: The total number of times that this network request has been retried (e.g. using different proxy settings, different DNS servers, or simple retries).
  * `X-Request-Age`: The presence of this header indicates that this request originally failed to send, and was persisted and later retried. The value of the header is the time interval in seconds between the client's current time and the client time at which the request was originally attempted.

Omaha Client uses CUP to secure the request and response. The following request headers are used to implement CUP:
  * If-Match: A signature that proves that the client knows the client's private key.
  * Cookie: An encrypted copy of the client's private verification key. Only early versions of Omaha Client send this.

The following response headers are used to implement CUP in Omaha Client:
  * ETag: Contains the signature of the response.
  * Set-Cookie: Encrypted copy of the client's private key, for the client to send in the next request. Only early versions of Omaha Client send this.

Additionally, Omaha Client uses a URL parameter `w` to encode a proposed private key.

## Examples ##
